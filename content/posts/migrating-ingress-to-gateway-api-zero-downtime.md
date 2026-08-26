+++
title = "Migrating Ingress to the Gateway API with Zero-Downtime on a Production Cluster"
date = "2026-08-25"
description = "How to migrate a production Ingress-based Kubernetes cluster to the Gateway API with zero downtime — run both controllers side-by-side, shift traffic through a DNS/SLB cutover, and keep a rollback path."
tags = ["kubernetes", "gateway-api", "ingress", "nginx", "nginx-gateway-fabric", "traffic-management", "zero-downtime", "devops"]
categories = ["DevOps"]
author = "Nahid Hasan"
aliases = ["/posts/migrating-ingress-to-gateway-api-zero-downtime/"]
+++

Most Gateway API tutorials start on an empty cluster: install the CRDs, create a `GatewayClass`, apply an `HTTPRoute`, curl it. That path avoids the hard problem almost entirely.

The hard problem is migrating a **live** cluster. You have production traffic flowing through an Ingress controller, TLS certificates already issued, a load-balancer IP that clients and DNS point at, and zero appetite for a maintenance window. This post is a production migration playbook: run the new Gateway API data plane **alongside** the old Ingress one, shift traffic incrementally through a DNS/load-balancer cutover, and keep a rollback path until you're certain.

<!--more-->

## The architecture difference that makes coexistence possible

Ingress and Gateway API are not two names for the same object — they are different enough that they can share a cluster.

| | Ingress (ingress-nginx) | Gateway API (NGINX Gateway Fabric) |
|---|---|---|
| Object model | Single `Ingress` + controller annotations | Role-split: `GatewayClass` → `Gateway` → `HTTPRoute` |
| Who owns the data plane | One controller, one `ingressClassName` | Each controller adopts `GatewayClass`es it `spec.controllerName` matches |
| Address | `status.loadBalancer` on the Ingress | `status.addresses` on the `Gateway` |
| Multi-tenancy | Implicitly one controller for all | Explicit per-`Gateway` ownership and `ReferenceGrant` |
| Backend selection | `servicePort` in the Ingress | `backendRef` with `port` on the `HTTPRoute` |

The crucial property: **these are independent control planes attached to separate load balancers.** `ingress-nginx` keeps serving on its existing `LoadBalancer` IP while NGINX Gateway Fabric (NGF) spins up its own `Gateway`, which gets its own address. Nothing about running the second controller stops the first. Coexistence is not just possible — it is the entire basis of a zero-downtime migration.

## The migration strategy in one breath

**Never delete the old path until the new path is proven in production.** The migration is a *slip*:

1. Install the Gateway API CRDs + NGF alongside ingress-nginx.
2. Bring up a `Gateway` on a **disjoint listener** — its own hostname and its own load-balancer IP. Production keeps using ingress-nginx untouched.
3. Smoke-test against the Gateway's IP directly (`curl --resolve`).
4. Shift traffic incrementally — weighted DNS or external-LB weights.
5. Full cutover: point primary DNS / the upstream LB at the Gateway's address.
6. Keep the old Ingress objects alive for rollback. Remove them only after you've watched the new path in production.

Each of these has real mechanics worth getting right.

## Step 1 — Run the two controllers side by side

```bash
# Gateway API CRDs (Standard channel)
kubectl kustomize "https://github.com/kubernetes-sigs/gateway-api/config/crd/standard" | kubectl apply -f -

# NGINX Gateway Fabric into its own namespace
kubectl create namespace nginx-gateway
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --create-namespace -n nginx-gateway
```

A default NGF install ships a `GatewayClass` named `nginx`, but the controller only **adopts** GatewayClasses whose `spec.controllerName` matches its own. If you are running a second Gateway-API controller, keep their `controllerName` values distinct so they never fight over the same class:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: nginx.org/gateway-fabric
```

Two controllers may coexist; a `GatewayClass` is owned by exactly one. That single-owner rule is what prevents the two data planes from stomping on each other.

## Step 2 — A Gateway on a disjoint listener

The `Gateway` is the data-plane instance. Give it its **own** listener (hostname) and let it provision its **own** load balancer:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: app-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      hostname: "*.app.example.internal"
      tls:
        certificateRefs:
          - name: app-tls
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
spec:
  parentRefs:
    - name: app-gateway
  hostnames:
    - "app.example.internal"
  rules:
    - backendRefs:
        - name: app-service
          port: 80
```

Confirm the Gateway got its own address:

```bash
kubectl get gateway app-gateway -o jsonpath='{.status.addresses}'
# → [{"type":"IPAddress","value":"<gateway-ip>"}]
```

That `<gateway-ip>` is a **new** load-balancer address. It does not touch the ingress-nginx IP, so the existing `Ingress` objects and their DNS keep serving production traffic as if nothing happened. This is the entire point of the disjoint listener.

Keep the listener hostnames *disjoint from the old Ingress rules* for now — overlapping a `hostname` across an `Ingress` and a `Gateway` isn't a hard error, but it invites confusion during cutover.

## Step 3 — Smoke-test before touching DNS

Never change DNS first. Probe the new path in isolation from a host that can reach the Gateway IP:

```bash
curl --resolve app.example.internal:443:<gateway-ip> \
  https://app.example.internal/healthz -s -o /dev/null -w "%{http_code}\n"
# → 200

# Or pin the Host header if you can't rewrite DNS from this host
curl -s -H "Host: app.example.internal" https://<gateway-ip>/healthz
```

Check three things: the TLS handshake (cert serves without warnings), the route resolves (200, not 404/503), and the backend is reachable through the new path. If the Gateway's cert is issued for the same hostname, `--resolve` is all you need to test the exact traffic your users will send.

## Step 4 — Shift traffic incrementally

You have two live, equivalent paths. Now slide the weight.

**Option A — weighted DNS:** if your clients resolve through a weighted record set, shift a small percentage, then more:

```text
app.example.internal 100 A <ingress-ip>
app.example.internal 0  A <gateway-ip>     # 0% today, raise it as you watch
```

**Option B — external load balancer weights:** insert a TCP/HTTPS LB in front and tune its target weights between the ingress-nginx address and the Gateway address.

Whichever you use, a few quality gates belong *before* you raise the weight:

- **Readiness:** the Gateway is `Programmed: True` and `status.addresses` is populated.
- **Health:** the `/healthz` through the Gateway returns 200 with the expected backend.
- **Error budget:** watch error rate for a few minutes at each step before raising the weight further.

## Step 5 — Full cutover

Once the weighted split holds up, point the **primary** record (or the upstream LB) at the Gateway's address and drop the ingress-nginx weight to zero:

```text
app.example.internal 100 A <gateway-ip>
```

Observe for a full traffic day before touching the old objects. DNS TTLs and client caches mean traffic may trickle to the old path for up to a full TTL — that is expected and safe, because ingress-nginx is still running.

## Step 6 — Roll back if you need to

Because you never deleted the old Ingress, rollback is the inverse slip, in seconds:

```text
app.example.internal 100 A <ingress-ip>
```

Keep the `Ingress` resources, the ingress-nginx namespace, and the original load-balancer Service intact until you are certain the Gateway path is stable. The one thing you must **not** do during this entire window is delete the old `LoadBalancer` Service — that frees the VIP clients still resolving to it.

## Step 7 — Decommission the old path

Only after you have watched the new path under real production load do you remove the old:

```bash
kubectl delete ingress <app-ingress>
# keep ingress-nginx installed in case another app still uses it
```

Then flatten the remaining Gateway API resources: remove the now-unused `Gateway`, `HTTPRoute`, and the old listener. Your namespace is clean — one control plane, one data plane, one listener.

## Rollback survival kit

The difference between "migration" and a "risky migration" is how fast you can reverse:

- **Do not delete the old `LoadBalancer` Service until cutover is proven.** Its IP is what rollback points back at.
- **Keep old Ingress objects.** Cheap to keep, expensive to recreate from memory.
- **Pin both TLS certs.** The Gateway's certificate must cover the same hostnames; a cert mismatch turns a silent migration into an outage the moment you cut over.
- **Two `controllerName`s, never shared.** The instant two controllers race a `GatewayClass`, neither reconciles it, and `status` stays `Unknown` (a classic observation — see the companion post on a stuck `GatewayClass`).

## Pitfalls quick reference

| Pitfall | Symptom | Fix |
|---|---|---|
| Two controllers share a `GatewayClass` | `GatewayClass` stuck `Accepted: Unknown`; neither reconciles | Distinct `spec.controllerName` per controller |
| Gateway IP not provisioned | Gateway not `Programmed: True`; no address | Check the controller is adopted + a LB is allocatable |
| Listener hostname overlaps old `Ingress` | Confusion during cutover | Keep listeners disjoint (own hostname) until switch |
| TLS cert for the Gateway differs from Ingress | Handshake warnings on cutover | Install the same cert, exact hostnames, on the Gateway |
| Deleted old `LoadBalancer` Service too early | Rollback has no IP to return to | Keep it alive until cutover is proven |
| `ReferenceGrant` missing (cross-namespace backend) | Route `ResolvedRefs: False` | Add a `ReferenceGrant` from the backend's namespace |
| Weighed DNS raised before smoke test | Traffic hits an unproven path | Verify `--resolve` 200 + error budget first |

## The takeaway

A zero-downtime Ingress→Gateway migration is not a big-bang replace; it is a **controlled slip** between two data planes that can coexist. Stand up the Gateway on its own listener and IP, prove it with `curl --resolve`, shift weight gradually, and keep the old path alive for rollback long enough to trust the new one. The Gateway API is genuinely nicer to operate — but you get to adopt it the way you'd adopt any production change: **without taking the site down.**

## References

- Gateway API overview & concepts: https://gateway-api.sigs.k8s.io/
- NGINX Gateway Fabric docs: https://docs.nginx.com/nginx-gateway-fabric/
- NGINX Gateway Fabric GatewayClass adoption: https://docs.nginx.com/nginx-gateway-fabric/overview/gateway-api-compatibility/
- Ingress-to-Gateway API migration (ingress2gateway): https://gateway-api.sigs.k8s.io/guides/ingress-to-gateway/
- Kubernetes Gateway API spec: https://kubernetes.io/docs/concepts/services-networking/gateway/
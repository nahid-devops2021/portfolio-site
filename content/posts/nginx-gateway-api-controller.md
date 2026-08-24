+++
title = "NGINX Gateway API Controller: From a Stuck GatewayClass to Healthy Leader Election"
date = "2026-08-24"
description = "A real debugging narrative: the GatewayClass sits stuck on Accepted: Unknown even though the controller Pod is running. The root cause is a single missing RBAC 'update' verb in the leases list that silently breaks leader election — and why 'status never reconciles' is usually a control-plane problem, not a routing problem."
tags = ["kubernetes", "gateway-api", "nginx", "nginx-gateway-fabric", "rbac", "leader-election", "traffic-management", "sre", "devops"]
categories = ["DevOps"]
author = "Nahid Hasan"
featuredImage = ""
aliases = ["/posts/nginx-gateway-api-controller/"]
+++

Most Gateway API tutorials stop at *"install the CRDs, apply an `HTTPRoute`, curl your service."* They treat the controller as a magic black box that just makes the statuses appear.

This post is about the other 90% — the operational reality. On a minimal lab cluster the controller started, the Pod was `Running`, nothing *looked* wrong in a basic `kubectl get`, yet the `GatewayClass` sat stuck on `Accepted: Unknown`, reason `Waiting for controller`, with zero conditions ever reconciled.

The fix turned out to be a single missing verb in the RBAC `leases` list — `update` — and it manifests so invisibly that, unless you know where to look, you will blame your `HTTPRoute`, your Gateway, and finally your cluster before you ever glance at leader election.

<!--more-->

## The setup: a minimal cluster, no managed ingress

My lab cluster is deliberately boring: a single-node control plane, standard networking, no managed ingress, nothing exotic. I was building a Gateway API-powered edge for a sandbox app and wanted to try NGINX Gateway Fabric (NGF) — an open-source, CNCF-hosted implementation that uses NGINX as the data plane and speaks the Gateway API natively.

**Bootstrap the Gateway API (Standard Channel) and install the controller:**

```bash
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard" | kubectl apply -f -
kubectl create namespace nginx-gateway

helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --create-namespace -n nginx-gateway

kubectl get pods -n nginx-gateway          # control-plane pod → Running
kubectl get gatewayclass                   # default class `nginx` exists
```

So far, so textbook. The manifests ship a default `GatewayClass`, the control-plane Deployment comes up, and nothing in the basic output suggests trouble. I applied my `Gateway` (pointing at `gatewayClassName: nginx`) and a matching `HTTPRoute`, then curled the Service.

Nothing. Of course — I hadn't checked status yet.

## The symptom: a status that never reconciles

```bash
kubectl get gatewayclass
kubectl describe gatewayclass <gateway-class>
```

```text
Status:
  Conditions:
    Last Transition Time:  1970-01-01T00:00:00Z
    Message:               Waiting for controller
    Reason:                Pending
    Status:                Unknown
    Type:                  Accepted
```

The chief signpost in the Gateway API world is `status.conditions`. Any serious debugging session starts there, because `conditions` are the contract: `type`, `status` (`True`/`False`/`Unknown`), `reason`, and `message`. The presence of **`Accepted: Unknown` with `Waiting for controller`** is a precise claim: *no controller that owns this class is reconciling it*.

## The systematic differential (the part nobody writes down)

An experienced reader can clear the obvious traps in minutes:

1. **Controller Pod is up** — `kubectl get pods -n nginx-gateway` shows `Running`, not `CrashLoopBackOff`. Not a resource/startup problem.
2. **`controllerName` matches** — `spec.controllerName: nginx.org/gateway-fabric`. The classic "wrong `controllerName` → class silently ignored" trap is ruled out.
3. **No admission/webhook error** on the class.

So the class is present, the right controller exists, and the Pod is healthy — yet nobody is claiming it. That contradiction should immediately nudge you toward the **control plane's ability to act**, and specifically toward **leader election**.

## Why leader election is the crux

NGF (built on `controller-runtime`) enables **lease-based leader election by default** on the control plane. From the official scaling docs, the key semantic is:

> only the pod with the leader lease can actively manage configuration status updates.

Read that twice, because it explains the entire mystery. All control-plane replicas can *push config* to data planes — but **only the current leader may write status** onto Gateway API resources like `GatewayClass.status`. So a leader-election fault doesn't look like an outage. It looks like "status never reconciles," with a perfectly healthy Pod.

The lease lives in the control-plane namespace:

```bash
kubectl get lease -n nginx-gateway
kubectl describe lease -n nginx-gateway ngf-nginx-gateway-fabric-leader-election
```

A healthy lease shows a `HolderIdentity` and a fresh `RenewTime`. Mine showed neither — no holder, stale renew. That's the pivot.

## The smoking gun in the controller logs

```bash
kubectl logs -n nginx-gateway deploy/ngf-nginx-gateway-fabric | grep -iE "lease|forbidden|leader"
```

```text
E… controller-runtime/leaderelection … Failed to renew lease …
Forbidden: … cannot update resource "leases" in API group "coordination.k8s.io" …
```

There it is. The controller can `create`/`get` the lease but **cannot `update` it** — so it can't renew leadership, becomes no one's leader, and therefore never writes status. Everything else looks fine because everything else *is* fine.

## Confirming the RBAC hole

Before touching anything, prove it with `kubectl auth can-i` — this turns a hunch into evidence:

```bash
# Simulate the control-plane ServiceAccount
kubectl auth can-i get leases -n nginx-gateway \
  --as=system:serviceaccount:nginx-gateway:<sa>          # yes
kubectl auth can-i update leases -n nginx-gateway \
  --as=system:serviceaccount:nginx-gateway:<sa>          # no  ← the hole
kubectl auth can-i update gatewayclasses/status \
  --as=system:serviceaccount:nginx-gateway:<sa>          # yes (when healed)
```

`get` is allowed, `update` is not. That asymmetry is the fingerprint of the dropped verb.

## The root cause: a mis-indented `verbs:` list

Leader election needs these verbs on the control plane's `ClusterRole`:

```yaml
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["create", "get", "update", "list", "watch"]
```

And — because the leader writes conditions — status updates too:

```yaml
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["gatewayclasses"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["gatewayclasses/status"]
  verbs: ["update"]
```

My hand-curated manifest had this:

```yaml
# Broken — update silently absent
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["create", "get"]
```

A one-line indentation slip in the `verbs:` block of a hand-written `ClusterRole` had quietly dropped `update`. The controller still *started* and still *watched* — it just could no longer renew its lease. Nothing crashes; no obvious error at deploy time; only `Forbidden` in the logs and a `GatewayClass` that never heals.

## The fix and the payoff

```yaml
# Fixed
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["create", "get", "update", "list", "watch"]
```

```bash
kubectl apply -f <fixed-rbac.yaml>
# Force a leader re-take (or wait — the controller retries)
kubectl delete pod -n nginx-gateway -l app.kubernetes.io/name=nginx-gateway-fabric
```

Then watch it heal in real time:

```bash
kubectl get lease -n nginx-gateway             # leader now held, fresh RenewTime
kubectl describe gatewayclass <gateway-class>  # Accepted: True, Ready
kubectl get gatewayclass                       # Accepted column flips to True
```

From there everything falls into place: the `Gateway` becomes `Programmed: True` (an address is assigned), the `HTTPRoute` resolves and is accepted, and traffic flows.

## The reusable troubleshooting ritual

When a Gateway API resource's status looks frozen, run this ladder:

- **Step 0 — read `status.conditions`.** `type`, `status`, `reason`, `message`. Trust only conditions where `observedGeneration == metadata.generation`.
- **Step 1 — is the class even adopted?** `kubectl describe gatewayclass`; confirm `spec.controllerName` matches the controller. `Waiting for controller` = not being reconciled.
- **Step 2 — is the controller alive?** Pod `Running`? `kubectl logs`? `kubectl get events -n nginx-gateway`?
- **Step 3 — leader election.** `kubectl get lease` — is there a `HolderIdentity` and a fresh `RenewTime`?
- **Step 4 — RBAC introspection.** `kubectl auth can-i …` as the controller's ServiceAccount. This is where the hole shows itself.
- **Step 5 — correlate.** The `Forbidden … cannot update resource "leases"` line ties steps 3 and 4 together and is conclusive.

## Pitfall quick-reference

| Pitfall | Symptom | Fix |
|---|---|---|
| `leases` missing `update` (core bug) | `GatewayClass` stuck `Unknown`/`Waiting for controller`; no leader; Pod `Running` | Add `update` to `leases`; restart control plane |
| Wrong `controllerName` | Class ignored — `Waiting for controller` | Set `spec.controllerName: nginx.org/gateway-fabric` |
| Missing `<resource>/status` `update` | Class adopted but conditions never written | Grant `update` on `*.gateway.networking.k8s.io/*/status` |
| Hand-rolled vs Helm RBAC drift | Intermittent `Forbidden` after a change | Treat `leases create/get/update` as mandatory |
| Scaling control > data planes | Config pushes but status writes can hit non-leader data plane | Keep data-plane pods ≥ control-plane replicas |
| Listener conflicts (hostname/port overlap) | Listener `Conflicted: True`; routes won't attach | Scope hostnames; don't overlap `port`+`hostname` |
| `ResolvedRefs: False` on route | Route not programmed despite healthy class | Add `ReferenceGrant`; verify backend Service/port |

## The takeaway

**"Status never reconciles" is frequently a control-plane / controller-runtime problem, not a routing problem.** Before you rewrite your `HTTPRoute` or tear down the Gateway, check leases, run `kubectl auth can-i`, and read the controller logs. The Gateway API gives you one of the best-built debugging breadcrumbs in Kubernetes — `status.conditions` — and a healthy leader election is the silent precondition that makes the statuses you're staring at actually get written.

And when you hand-write RBAC "just to make it run": treat `leases` `create`, `get`, **and `update`** as load-bearing. One dropped verb is all it takes to make a perfectly healthy controller look like it's doing nothing at all.
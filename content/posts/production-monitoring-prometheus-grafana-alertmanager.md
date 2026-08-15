+++
title = "Building a Production-Grade Monitoring Stack with Prometheus, Grafana, and Alertmanager"
date = "2026-08-15"
description = "A production-focused guide to designing, deploying, securing, and operating Prometheus, Grafana, Alertmanager, and Node Exporter — from architecture to alerting to capacity planning."
tags = ["prometheus", "grafana", "alertmanager", "node-exporter", "monitoring", "observability", "sre", "docker", "devops"]
categories = ["DevOps"]
author = "Nahid Hasan"
aliases = ["/posts/prometheus-grafana-alertmanager/"]
+++

# Building a Production-Grade Monitoring Stack with Prometheus, Grafana, and Alertmanager

Monitoring is not a feature you bolt on after the fact — it is the feedback loop that tells you whether your infrastructure is actually doing what you designed it to do. This article walks through designing, deploying, securing, validating, and operating a production-grade monitoring stack built on Prometheus, Grafana, Alertmanager, and Node Exporter.

This is not a "run this container and you're done" tutorial. Every decision in this article — where to scrape, how to alert, what to record, how to secure the platform — exists because of a production requirement. If you follow along, you will end up with a working stack and, more importantly, an understanding of why it is built the way it is.

**What this article covers:**

- Why the architecture is designed the way it is
- A complete Docker Compose deployment (portable to VMs and Kubernetes)
- Meaningful, non-noisy alert design
- Security hardening for the monitoring platform
- Troubleshooting, validation, and capacity planning
- Monitoring the monitoring stack itself

**Assumptions:** You are comfortable with Linux, basic networking, and Docker. You do not need deep Prometheus experience — the concepts are explained as we go.

**Scope note:** This guide targets a small-to-medium production environment (tens of hosts, thousands of series). Large-scale multi-cluster architectures are discussed in Section 14, but the implementation here is intentionally single-instance.


## Why Production Monitoring Matters

Before writing any configuration, it is worth being explicit about what monitoring buys you in production.

### Availability

An unmonitored outage is just an outage. With monitoring, it becomes an incident — something you can detect, measure, and learn from. The difference between "the site was down for four hours" and "the site was down for four minutes" is usually not luck; it is detection speed.

### Performance and Reliability

Metrics reveal degradation before users feel it. A slowly climbing error rate, a growing queue, or a page-cache collapse are all visible in time-series data. Monitoring gives you the ability to notice that something is *becoming* a problem rather than discovering it is already one.

### Capacity Planning

You cannot plan storage, compute, or network growth without data. Retention-aware metric history answers questions like "how fast is disk filling?" and "when will we run out of memory?" with evidence rather than guesses.

### Incident Detection and Troubleshooting

During an incident, metrics are the first place an engineer looks. A well-designed dashboard compresses hours of "where do we start?" into seconds. Alerts tell you *what* broke; dashboards tell you *where to look*.

### SLA and SLO Awareness

If your team operates against service-level objectives, you need a way to measure error budgets. Prometheus's native support for ratio queries (for example, `sum(rate(http_requests_total{code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])))`) makes SLO burn-rate alerting practical.

### Metrics, Logs, Traces, and Alerts — and Why This Stack Is About Metrics

The four pillars of observability serve different purposes:

| Telemetry | Answers | Primary Tooling |
|-----------|---------|-----------------|
| **Metrics** | "Is something wrong?" | Prometheus + Grafana |
| **Logs** | "What exactly happened?" | Loki, Elasticsearch, etc. |
| **Traces** | "Where did the request go wrong?" | Jaeger, Tempo, OpenTelemetry |
| **Alerts** | "What needs my attention now?" | Alertmanager |

This article focuses on the metrics-and-alerts pillar. Metrics are the most efficient signal to collect at scale — each sample is a small, well-structured number — and they are the foundation that alerts are built on. Logs and traces are complementary, and are mentioned where they belong in the pipeline, but a complete logging pipeline is out of scope.


## Architecture Overview

The stack follows a simple, proven data flow: exporters expose metrics, Prometheus collects and stores them, Grafana visualizes them, and Alertmanager routes alerts generated from Prometheus rules to notification channels.

{{< mermaid >}}
graph TB
    subgraph Targets
        T1[Linux Hosts] --> N1[Node Exporter :9100]
        T2[Applications] --> N2[App Exporter]
        T3[Databases / Services] --> N3[DB / Service Exporter]
    end
    N1 --> P[Prometheus :9090]
    N2 --> P
    N3 --> P
    P --> G[Grafana :3000]
    P --> A[Alertmanager :9093]
    A --> C1[Chat / Incident Channel]
    A --> C2[Email]
    P --> R[Rules / Recording]
{{< /mermaid >}}

### Why Each Component Exists

**Exporters** exist because Prometheus scrapes over HTTP but most systems do not natively expose Prometheus metrics. Node Exporter translates Linux kernel and `/proc` data into a `/metrics` endpoint. Application exporters do the same for databases, message queues, web servers, and custom services.

**Prometheus** is the collector and store. Its pull model — Prometheus initiates every scrape — means targets do not need to know who is monitoring them, which simplifies access control and makes discovery possible. Data is stored as a local time-series database (TSDB) with a query language (PromQL) designed specifically for this data shape.

**Grafana** is the visualization layer. It queries Prometheus (and other data sources) and renders dashboards. It is intentionally decoupled from collection: Grafana can go down without affecting metric collection, and Prometheus can go down without losing dashboard definitions.

**Alertmanager** is the notification brain. Prometheus evaluates rules and pushes alerts; Alertmanager deduplicates, groups, routes, and repeats them according to routing rules, and delivers them through receivers (chat, email, PagerDuty, webhooks). Decoupling alert evaluation (Prometheus) from delivery (Alertmanager) means notification policy can change without touching metric configuration.

### Data Flow

1. Prometheus scrapes each target's `/metrics` endpoint on its scrape interval (default 15s).
2. Samples are stored in the local TSDB, indexed by metric name and label set.
3. Grafana queries the TSDB through PromQL and renders panels.
4. Prometheus evaluates alert rules on the evaluation interval; matching conditions transition an alert from *inactive* to *pending* to *firing*.
5. Firing alerts are pushed to Alertmanager, which applies routing and delivers notifications.


## Technology Stack

| Component | Purpose | Port |
|-----------|---------|------|
| **Prometheus** | Metrics collection, storage, and alert rule evaluation | 9090 |
| **Node Exporter** | Exposes Linux host metrics (CPU, memory, disk, network) | 9100 |
| **Grafana** | Visualization, dashboards, and querying | 3000 |
| **Alertmanager** | Alert deduplication, routing, and notification delivery | 9093 |

This is the minimum set. Add other exporters (mysqld_exporter, blackbox_exporter, cAdvisor) only when you have the corresponding workload — every additional target is additional load on Prometheus and additional dashboard surface to maintain.

## Infrastructure Requirements

The requirements below describe a *small-to-medium* production environment (roughly 10–50 hosts, 5k–50k active series). They are starting points, not universal production requirements.

### Hardware

| Resource | Small (10 hosts) | Medium (50 hosts) |
|----------|-------------------|--------------------|
| **CPU** | 2 vCPU | 4 vCPU |
| **Memory** | 4 GB | 8 GB |
| **Storage** | 50 GB SSD | 200 GB SSD |

### Sizing Reality Check

Actual sizing depends on variables that are specific to your environment:

- **Number of targets** — each target adds scrape overhead
- **Metrics per target** — a Node Exporter exposes ~1000 metrics; an application exporter can expose 10x that
- **Scrape interval** — 15s vs 30s doubles or halves samples per second
- **Retention period** — local TSDB retention directly determines disk usage
- **Cardinality** — the product of all label values across a metric family (see Section 16)
- **Query workload** — dashboard-heavy environments need more CPU and memory

A reasonable formula for estimating storage: each time series consumes roughly **1–2 bytes per sample** on disk (compressed). If you scrape 20k series every 15s, that is ~1,333 samples/second → ~115M samples/day → roughly **3–6 GB/day** including index overhead. Plan for double that to absorb churn and compaction.

### Software and Network

- **OS:** any modern Linux (RHEL/AlmaLinux, Ubuntu, Debian). All examples use systemd and Docker.
- **DNS:** use resolvable hostnames for targets; Prometheus resolves them at scrape time.
- **Firewall:** only the following ports need to be reachable, and only from the hosts that need them:

| Port | Service | Who needs access |
|------|---------|------------------|
| 9090 | Prometheus UI/API | Operators, Grafana server |
| 9100 | Node Exporter | Prometheus server only |
| 3000 | Grafana | Operators, reverse proxy |
| 9093 | Alertmanager UI | Operators, Prometheus server |

Everything else should be denied at the firewall (see Section 13).


## Deployment Strategy

The concepts in this article apply to any deployment method. We implement with **Docker Compose** for three reasons:

1. **It is reproducible** — the full stack is defined in one file, versioned in git
2. **It is honest about state** — volumes for data, configs mounted read-only
3. **It ports cleanly** — the same images and configuration transfer to Kubernetes (as Deployments and ConfigMaps) or bare-metal (as systemd services) with minimal change

A single host running Docker Compose is a legitimate deployment for a small production environment. For a single Prometheus instance, it is operationally fine — the OS runs, Docker restarts the containers, and data lives on a persistent volume. When you outgrow it, Section 14 covers the path to HA and long-term storage.

## Prometheus Installation

### 7.1 Installation

We use the official `prom/prometheus` image. Pin a specific version in production (for example `prom/prometheus:v2.53.0`); `latest` is a moving target and breaks reproducible deployments.

### 7.2 Directory Structure

```
/etc/prometheus/
├── prometheus.yml
├── rules/
│   ├── node-alerts.yml
│   └── application-alerts.yml
└── targets/
    └── servers.yml
```

Keeping rules and targets in separate files (loaded via `rule_files` and file-based service discovery) means you can add an alert or a server without touching the main configuration.

### 7.3 Prometheus Configuration

The core configuration file, `/etc/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    environment: "production"
    cluster: "primary"

rule_files:
  - "/etc/prometheus/rules/*.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

- **`scrape_interval`** — how often Prometheus pulls metrics from targets. 15s is a good default for host monitoring; consider 30s for very large fleets to halve ingestion load.
- **`evaluation_interval`** — how often alert and recording rules are evaluated. Keep it equal to the scrape interval so rules see fresh data.
- **`scrape_configs`** — the list of targets and jobs.
- **`rule_files`** — alert and recording rules, loaded from a glob.
- **`alerting`** — where to send alerts (Section 10).
- **`external_labels`** — labels added to every series and alert. Critical later if you ever use remote write or multi-cluster setups.


### 7.4 Scrape Targets

Realistic targets, split by job:

```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    file_sd_configs:
      - files:
          - "/etc/prometheus/targets/servers.yml"
        refresh_interval: 1m
```

And `/etc/prometheus/targets/servers.yml`:

```yaml
- targets:
    - "web01.example.internal:9100"
    - "web02.example.internal:9100"
    - "db01.example.internal:9100"
  labels:
    role: "webserver"

- targets:
    - "db01.example.internal:9100"
  labels:
    role: "database"
```

File-based service discovery means adding a new server is a one-line file edit — no Prometheus restart required (see Section 12).

### 7.5 Recording Rules

Recording rules precompute expensive queries. They are useful when a query is:

- **Expensive** — ranges over many series (for example, per-instance CPU rates across a fleet)
- **Repeated** — used by many dashboards or alerts

Example, `/etc/prometheus/rules/node-alerts.yml`:

```yaml
groups:
  - name: node-recording-rules
    rules:
      - record: instance:node_cpu_utilization:rate5m
        expr: |
          1 - avg without (cpu) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          )

      - record: instance:node_memory_available_percent
        expr: |
          node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100
```

Dashboards then query `instance:node_cpu_utilization:rate5m` instead of recomputing the full expression each time. Use recording rules sparingly — each one adds evaluation load and another name to remember.

### 7.6 Alert Rules

Alert rules live in the same rule files. They consist of an expression, a `for` duration, labels, and annotations:

```yaml
groups:
  - name: node-alerts
    rules:
      - alert: NodeFilesystemSpaceFillingUp
        expr: |
          (
            node_filesystem_avail_bytes{mountpoint="/"}
            / node_filesystem_size_bytes{mountpoint="/"}
          ) * 100 < 20
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }} root filesystem below 20%"
          description: "{{ $labels.instance }} has {{ $value | humanize }}% free on {{ $labels.mountpoint }}"
          runbook_url: "https://your-wiki.example.internal/runbooks/node-filesystem"
```

The `for` clause is the single most effective tool against alert noise — it requires the condition to persist (here, 10 minutes) before the alert fires.

### 7.7 Configuration Validation

**Never restart a production Prometheus after a config change without validating first.** Use `promtool`:

```bash
promtool check config /etc/prometheus/prometheus.yml
promtool check rules /etc/prometheus/rules/*.yml
```

In the container, mount the config read-only and run the check in an ephemeral container:

```bash
docker run --rm -v /etc/prometheus:/etc/prometheus:ro \
  prom/prometheus:v2.53.0 promtool check config /etc/prometheus/prometheus.yml
```

Only reload after a clean check:

```bash
curl -X POST http://localhost:9090/-/reload
```

The `/-/reload` endpoint applies rule and target changes without dropping scrapes. A full restart is only needed for `global` or storage settings.


## Node Exporter

Node Exporter exposes Linux host metrics: CPU, memory, disk, filesystem, network, load, and more. It reads from `/proc` and `/sys` and publishes them as Prometheus-format metrics on port 9100.

### Installation

```bash
docker run -d --name node_exporter \
  --restart unless-stopped \
  --network host \
  --pid host \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /:/rootfs:ro \
  prom/node-exporter:v1.8.2 \
    --path.procfs=/host/proc \
    --path.sysfs=/host/sys \
    --path.rootfs=/rootfs
```

The mounts are read-only: Node Exporter only *reads* host kernel state; it never writes to it. `--network host` binds it directly to the host's port 9100 — appropriate for host metrics.

### Service Configuration (Bare-Metal / systemd)

On a host without Docker, run it as a systemd unit:

```ini
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
Type=simple
User=nobody
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Prometheus Integration

Add the host to the target file (Section 7.4) — the job already exists. Verify the endpoint manually:

```bash
curl http://HOST_IP:9100/metrics | head
```

### Key Node Exporter Metrics

| Metric | Meaning |
|--------|---------|
| `node_cpu_seconds_total` | Cumulative CPU time per mode (user, system, idle, ...). Rate it, don't sum it raw |
| `node_memory_MemAvailable_bytes` | Memory realistically available for new allocations |
| `node_filesystem_avail_bytes` | Free space per filesystem (not "used" — more useful for alerts) |
| `node_load1` / `node_load5` / `node_load15` | Load average over 1/5/15 minutes |
| `node_network_receive_bytes_total` / `node_network_transmit_bytes_total` | Cumulative bytes in/out per interface. Rate them for throughput |

The common thread: counters (everything ending `_total`) must be `rate()`d, not averaged; gauges (load, memory) are read as-is.


## Grafana

### 9.1 Installation

```yaml
  grafana:
    image: grafana/grafana:11.1.0
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=YOUR_PASSWORD
    depends_on:
      - prometheus
```

### 9.2 Initial Configuration

Change the admin password immediately after first login. In production, configure Grafana against a real identity provider via LDAP or OAuth rather than local accounts, and place it behind a reverse proxy with TLS (Section 13).

### 9.3 Prometheus Data Source

Data sources are provisioned either through the UI or, better, declaratively. Production teams should provision:

```yaml
# /etc/grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

Declarative provisioning means dashboards and data sources are version-controlled, and a fresh Grafana instance is fully configured on first boot.

### 9.4 Dashboard Design

A useful dashboard is an investigation tool, not a screensaver. Design two tiers:

**Infrastructure Overview** (one row per host, or one panel per resource):
- CPU utilization (`instance:node_cpu_utilization:rate5m` — the recording rule from Section 7.5)
- Memory usage (`node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes`)
- Disk usage per mountpoint
- Load average
- Network throughput (`rate(node_network_receive_bytes_total[5m])`)
- Host availability (`up`)

**Application Overview** (RED method — Rate, Errors, Duration):
- Request rate: `sum(rate(http_requests_total[5m])) by (service)`
- Error rate: `sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)`
- Latency percentiles: `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))`
- HTTP status codes by class

> **The principle:** Dashboards are primarily for investigation and visualization; alerts are for actionable events. A dashboard should answer "what is happening right now?" An alert should answer "what needs your attention now?" They serve different purposes and should be designed separately.


## Alertmanager

### Alertmanager Architecture

Alertmanager sits between Prometheus rule evaluation and the notification channels. Its four jobs:

- **Deduplication** — identical alerts from multiple replicas collapse into one
- **Grouping** — related alerts (same alertname, job, or instance) batch into a single notification
- **Inhibition** — a higher-severity alert suppresses lower-severity noise (for example, `InstanceDown` inhibits its `NodeFilesystemSpaceFillingUp` child alerts)
- **Routing** — a routing tree decides which receiver gets which alert, based on labels

### Configuration

```yaml
# /etc/alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ["alertname", "instance"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: "team-default"
  routes:
    - matchers:
        - severity = "critical"
      receiver: "incident-channel"
      continue: false
    - matchers:
        - severity = "warning"
      receiver: "team-chat"
    - matchers:
        - severity = "info"
      receiver: "email"

receivers:
  - name: "team-default"
    webhook_configs:
      - url: "http://your-internal-sink:PORT/hook"
  - name: "incident-channel"
    webhook_configs:
      - url: "YOUR_INCIDENT_WEBHOOK_URL"
  - name: "team-chat"
    webhook_configs:
      - url: "YOUR_CHAT_WEBHOOK_URL"
  - name: "email"
    email_configs:
      - to: "oncall@example.internal"
        from: "alertmanager@example.internal"
        smarthost: "smtp.example.internal:587"
        auth_username: "YOUR_SMTP_USER"
        auth_password: "YOUR_SMTP_PASSWORD"
```

All credentials above are placeholders — never commit real webhook URLs, tokens, or SMTP passwords. Manage secrets via environment variables or a secret manager (Section 13).

### Alert Flow

{{< mermaid >}}
graph LR
    P[Prometheus rule firing] --> A[Alertmanager]
    A -->|severity=warning| C[Team Chat]
    A -->|severity=critical| I[Incident Channel]
    A -->|severity=info| E[Email]
{{< /mermaid >}}

Prometheus is pointed at Alertmanager via the `alerting` block in `prometheus.yml`:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]
```


## Production-Grade Alerting Strategy

This is the section that separates a monitoring stack from a pager-spammer. Alert design is an engineering discipline; the goal is alerts that are *actionable, correct, and rare enough to be taken seriously*.

### Thresholds Adapted to Workload

The examples below are starting points. A database server, a web frontend, and a batch worker have different profiles — tune thresholds per role, not globally.

### CPU

```yaml
- alert: NodeHighCpuLoad
  expr: instance:node_cpu_utilization:rate5m > 0.80
  for: 10m
  labels: { severity: warning }
  annotations:
    summary: "CPU above 80% for 10 minutes on {{ $labels.instance }}"

- alert: NodeCriticalCpuLoad
  expr: instance:node_cpu_utilization:rate5m > 0.95
  for: 5m
  labels: { severity: critical }
  annotations:
    summary: "CPU above 95% for 5 minutes on {{ $labels.instance }}"
```

### Disk

```yaml
- alert: NodeFilesystemSpaceFillingUp
  expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 20
  for: 10m
  labels: { severity: warning }
  annotations:
    summary: "Root filesystem below 20% free on {{ $labels.instance }}"

- alert: NodeFilesystemSpaceCritical
  expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 10
  for: 5m
  labels: { severity: critical }
  annotations:
    summary: "Root filesystem below 10% free on {{ $labels.instance }}"
```

### Memory

Prefer **available-memory** alerts over naive used-percentage alerts, because Linux caches aggressively and a high "used" figure is often healthy:

```yaml
- alert: NodeLowMemory
  expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.10
  for: 10m
  labels: { severity: warning }
  annotations:
    summary: "Memory available below 10% on {{ $labels.instance }}"
```

### Alert Design Principles

- **`for` durations filter transient conditions** — a 1-minute CPU spike is noise; a 10-minute plateau is a problem
- **Severity labels drive routing, not just cosmetics** — critical routes to the incident channel, warning to chat, info to email
- **Annotations carry context** — summary, description with the actual `$value`, and a `runbook_url`
- **Runbook URLs turn alerts into action** — every alert should link to a doc that says what to check and who owns it
- **Alert fatigue is a design failure** — if engineers start ignoring alerts because they're always wrong, the alert design is the bug
- **False positives are expensive** — every false alert trains the team to ignore the real ones. Prefer "later but correct" over "now but wrong"


## Service Discovery

Static configuration works until you add your tenth server. Production monitoring needs discovery.

### Static Configuration

Fine for a handful of fixed targets. Every change requires editing a file (and, with `file_sd_configs`, only a reload).

### File-Based Service Discovery

As shown in Section 7.4, Prometheus watches target files and picks up changes automatically on the refresh interval — no restart, no reload, no human. This is the sweet spot for VM fleets managed by configuration management: your CM tool writes the target file, Prometheus notices.

### Kubernetes Service Discovery

Inside Kubernetes, Prometheus discovers targets natively through the API:

```yaml
scrape_configs:
  - job_name: "kubernetes-pods"
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: "(.+)"
```

Pods opt in with an annotation (`prometheus.io/scrape: "true"`), and relabeling maps Kubernetes metadata onto Prometheus labels. This is the standard pattern for Kubernetes monitoring and is what makes Prometheus scale to dynamic environments.

### Cloud Service Discovery

AWS, GCP, and Azure integrations discover instances through the cloud APIs (`ec2_sd_configs`, `gce_sd_configs`, `azure_sd_configs`), tagging instances with labels from cloud metadata. File-based discovery is a fine start; move to cloud or K8s discovery as the fleet grows.

The takeaway: **design for discovery from day one** — even with static configs, keep them in separate files and label targets by role so the jump to dynamic discovery is configuration-only.

## Security Hardening

A monitoring platform holds a complete map of your infrastructure. It is a high-value target and must be locked down accordingly.

### Network Segmentation and Firewall

- Scrape traffic should be **prometheus-server → exporter only**. Exporters (9100) should refuse connections from everything else:

```bash
# firewalld example: allow 9100 only from the Prometheus host
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" \
  source address=10.0.0.10 port port=9100 protocol=tcp accept'
firewall-cmd --reload
```

- Alertmanager and Grafana sit in an operator-facing zone, not the public zone.

### TLS

- **Exporter TLS:** Node Exporter can serve TLS with a cert/key pair (`--web.config.file`). Recommended when scrape traffic crosses untrusted segments.
- **Grafana:** terminate TLS at a reverse proxy (nginx/Caddy) in front of Grafana — also gives you centralized access control.
- **Prometheus/Alertmanager:** same reverse-proxy pattern applies.

### Authentication

- **Grafana:** require authentication (default) and integrate with LDAP/OAuth for real user management.
- **Prometheus:** the UI/API should not be open. Basic auth via a `web.yml` config, or — better — put it behind the reverse proxy with SSO.
- **Alertmanager:** same treatment; its UI can silence alerts, so it must be protected.

### Reverse Proxy Pattern

```nginx
# /etc/nginx/conf.d/grafana.conf
server {
    listen 443 ssl;
    server_name monitoring.example.internal;

    ssl_certificate     /etc/nginx/tls/fullchain.pem;
    ssl_certificate_key /etc/nginx/tls/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
    }
}
```

### Secrets Management

- Never store credentials in configuration files committed to git
- Use environment variables or a secrets manager (Vault, sops, Docker secrets)
- Rotate webhook URLs and SMTP passwords on a schedule
- All examples in this article use placeholders (`YOUR_PASSWORD`, `YOUR_WEBHOOK_URL`) — follow that pattern in your own repos

### Least Privilege

- Run exporters as unprivileged users (`nobody` for Node Exporter)
- Give Grafana's service account only the query permissions it needs
- Keep the Prometheus server's filesystem access minimal — it does not need to write anywhere but its TSDB


## High Availability and Scalability

It is important to be precise about what a single Prometheus gives you: **no HA and bounded scalability**. The architecture in this article is correct for a small-to-medium environment; when you outgrow it, the path forward is well-trodden.

### What Happens When Components Fail

- **Prometheus fails** → no scrapes, no rule evaluation, no alerts, dashboards go stale. Recovery is restore-from-backup of the TSDB.
- **Grafana fails** → visualization is lost, but collection and alerting continue. Grafana is stateless (config + dashboards are just files), so it restarts in seconds.
- **Alertmanager fails** → alerts are still generated by Prometheus but not delivered. Prometheus queues alerts with a bounded buffer; prolonged Alertmanager downtime means missed notifications.

### Prometheus HA

Two identical Prometheus instances scraping the same targets, both feeding alerts to the same Alertmanager cluster. Alertmanager deduplicates the duplicate alerts. This gives failover, not consistency — each instance keeps its own storage, and you accept that they will diverge.

### Alertmanager Clustering

Alertmanager nodes gossip over a cluster port and coordinate deduplication, inhibition, and silences across replicas:

```bash
docker run -d --name alertmanager \
  --restart unless-stopped \
  -p 9093:9093 \
  -p 9094:9094 \
  -v /etc/alertmanager:/etc/alertmanager:ro \
  prom/alertmanager:v0.27.0 \
    --config.file=/etc/alertmanager/alertmanager.yml \
    --cluster.listen-address=0.0.0.0:9094
```

Point Prometheus at all replicas in the `alerting` block.

### Remote Write and Long-Term Storage

Prometheus's local TSDB is not a long-term archive. For months of retention at scale, use remote write to a long-term store:

- **Thanos** — adds sidecars to Prometheus for object-storage-backed retention and a global query view across multiple Prometheus instances
- **VictoriaMetrics** — a drop-in Prometheus-compatible TSDB that is far more storage-efficient and handles higher ingestion
- **Grafana Mimir** — Grafana's horizontally scalable, multi-tenant long-term store; overkill for small fleets, the standard for large ones

### The Honest Distinction

| Scale | Appropriate Architecture |
|-------|--------------------------|
| **Small environment** (this article) | Single Prometheus + Alertmanager, local TSDB, 15-day retention |
| **Large-scale production** | HA Prometheus pairs, Alertmanager cluster, Thanos/Mimir/VictoriaMetrics for retention, federated or global querying, dedicated dashboard tier |

Do not bolt HA onto a single-instance setup until you have actually outgrown it — complexity is a cost, and the single instance is operationally simpler to run correctly.


## Monitoring the Monitoring Stack

The monitoring platform must monitor itself. During an incident, the worst possible situation is discovering that *the system that would have told you about the problem was itself down*.

### Prometheus Self-Monitoring

- **Target failures:** `up == 0` — a scrape target stopped responding
- **Scrape failures:** `scrape_samples_scraped == 0` while `up == 1` — the target responds but returns nothing
- **Rule evaluation failures:** `prometheus_rule_evaluation_failures_total` increasing — broken rules silently stop evaluating
- **TSDB health:** `prometheus_tsdb_head_series` growth, `prometheus_tsdb_compactions_failed_total`
- **Storage:** disk usage of the TSDB volume

### Grafana Self-Monitoring

- `grafana_http_request_duration_seconds` and error rates
- Grafana's own health endpoint (`/api/health`) probed by an external blackbox check

### Alertmanager Self-Monitoring

- `alertmanager_notifications_failed_total` — delivery failures (the alert that never arrived)
- `alertmanager_notifications_total` — volume baseline
- `alertmanager_cluster_members` — cluster health when running replicated
- Queue state: `alertmanager_notifications_queue_length`

### Why This Matters

A monitor that isn't monitored is a single point of failure you cannot see. Add the self-monitoring alerts (Prometheus → itself → Alertmanager) as the *first* alert rules you deploy — before the server alerts. If the platform dies silently, the "monitoring stack is down" alert is the one that still fires, because it is evaluated by the only component still running.


## Performance and Capacity Planning

### Samples per Second

The fundamental throughput unit: `samples_per_second = total_series × (1 / scrape_interval)`. With 20k series at 15s: 1,333 samples/s. Every exporter added, every label added, every scrape interval halved increases this.

### Storage

Rough planning: **1–2 bytes per sample** on disk (compressed), plus index overhead (typically 10–20%). Retention is configured in Prometheus:

```yaml
storage:
  tsdb:
    retention:
      time: 15d
      size: 50GB   # either/or; size-based is safer for SSD-bound hosts
```

Prefer `retention.size` over `retention.time` on small disks — it prevents the TSDB from filling the disk, which is a much worse failure than dropping old data.

### Query Performance

- Dashboards that compute heavy expressions on every refresh hammer the TSDB
- Recording rules (Section 7.5) move expensive computation to write time
- Limit dashboard refresh rates in large deployments

### Cardinality — The Concept That Bites

Cardinality of a metric = number of unique combinations of label values. This is the most common cause of Prometheus memory blowups, and it is almost always accidental.

**Bad example** — a label with unbounded unique values:

```
http_requests_total{client_ip="203.0.113.42", ...}
```

Every unique client IP creates a new series. Ten million requests from ten million IPs = ten million series for one metric. This will exhaust memory on any reasonable server.

**Good example** — bounded, meaningful labels:

```
http_requests_total{status="200", method="GET", service="checkout"}
```

Status, method, and service each have a small, known set of values — the metric stays tractable no matter how many requests flow through.

**The rule:** labels should identify *dimensions you aggregate by*. If a label can take an unbounded set of values (IPs, user IDs, request IDs, trace IDs), it does not belong on a metric — put it in a log instead.

### Monitoring Your Own Load

Track `prometheus_tsdb_head_series` over time. A steadily climbing line with no plateau is the early-warning sign of cardinality growth — catch it before it becomes a memory problem.


## Troubleshooting

Six realistic scenarios, with the diagnostic steps an engineer actually runs.

### Scenario 1: A Target Is DOWN

Prometheus → **Status → Targets** shows the target red with the last error.

```bash
# 1. Is the exporter reachable from the Prometheus host?
curl -v http://TARGET_IP:9100/metrics

# 2. Is it listening on the right port?
ss -tlnp | grep 9100          # on the target host

# 3. Is the service running?
systemctl status node_exporter   # or: docker ps | grep node

# 4. Firewall / network
ping -c1 TARGET_IP              # basic reachability
nc -zv TARGET_IP 9100           # port reachability
```

Common causes: exporter service stopped, wrong port in the target file, firewall blocking 9100 from the Prometheus host, DNS resolving to the wrong address, or a config typo (e.g. `127.0.0.1` instead of the real IP).

### Scenario 2: An Alert Is Not Firing

- Prometheus → **Alerts** shows whether the rule is inactive, pending, or firing
- Check the rule expression manually in **Graph** view — if the query returns nothing, the expression is wrong (or the metric name/label is)
- Check the `for` duration — the condition may be true but not sustained long enough
- Verify the rule file is loaded: **Status → Rules** lists parsed rules; `promtool check rules` catches syntax errors

### Scenario 3: Alertmanager Receives No Alerts

- Is the `alerting` block in `prometheus.yml` pointing at the right host:port?
- Prometheus → **Status → Runtime & Build** → Alertmanagers — are they listed as *available*?
- Check `prometheus_alertmanager_notifications_total` and `prometheus_notifications_errors_total`
- `curl http://ALERTMANAGER:9093/-/ready` — is Alertmanager itself healthy?

### Scenario 4: Grafana Shows No Data

- **Explore** tab → run a trivial query (`up`) against the Prometheus data source
- Data source URL wrong? It must reach Prometheus — use the container name (`http://prometheus:9090`) from within the compose network, not `localhost`
- Are the series actually present? Query Prometheus directly: `curl http://PROMETHEUS:9090/api/v1/query?query=up`
- Time-range issue — the panel's selected time window may be older than your retention

### Scenario 5: Notifications Are Not Delivered

- Alertmanager → **Status** shows whether the notification succeeded and the last error
- `alertmanager_notifications_failed_total` increasing = delivery failures; check the receiver config (webhook URL, SMTP settings)
- Silences active? Check Alertmanager → **Silences**
- Grouping: `group_wait`/`repeat_interval` may legitimately delay delivery — that is configuration, not failure

### Scenario 6: Prometheus Memory Usage Keeps Growing

- Check `prometheus_tsdb_head_series` — if it climbs without plateau, it is cardinality growth (Section 14)
- Use the **TSDB Status** page: top label-value pairs, top series by cardinality
- Check scrape load: too many targets at a short interval
- Query load: heavy dashboard queries on every refresh

The common thread across all scenarios: **Prometheus exposes the state of every part of itself as metrics** — use them before guessing.


## Testing and Validation

"Installation successful" is not validation. Prove the pipeline end-to-end:

### 1. Exporters Expose Metrics

```bash
curl http://TARGET_IP:9100/metrics | grep node_memory_MemTotal_bytes
```

### 2. Prometheus Scrapes Successfully

```bash
curl 'http://PROMETHEUS:9090/api/v1/query?query=up'
# Expect: up{instance="TARGET_IP:9100"} → 1
```

### 3. Rules Parse and Evaluate

```bash
promtool check rules /etc/prometheus/rules/*.yml
curl 'http://PROMETHEUS:9090/api/v1/query?query=instance:node_cpu_utilization:rate5m'
```

### 4. Alert Pipeline — Forced Test

Use a deliberately low-threshold test alert:

```yaml
- alert: TestAlert
  expr: vector(1)
  for: 1m
  labels: { severity: warning }
  annotations:
    summary: "Test alert — confirm delivery and silence afterwards"
```

Watch it flow: **pending → firing → Alertmanager received → notification delivered** (check the receiver's endpoint and Alertmanager Status page). Then silence it.

### 5. Full Pipeline Validation

```
Generate test condition
        ↓
Prometheus detects metric
        ↓
Alert rule evaluates
        ↓
Alert becomes Pending
        ↓
Alert becomes Firing
        ↓
Alertmanager receives alert
        ↓
Notification delivered
```

### 6. Test on Schedule

Notifications rot: webhook URLs expire, SMTP credentials change, chat apps move. Re-run this validation monthly, or automate it with a cron that sends a heartbeat alert.


## Production Checklist

```text
[ ] Prometheus configured (scrape, rules, alerting)
[ ] Exporters deployed (Node Exporter on all hosts)
[ ] Targets monitored (Status → Targets all green)
[ ] Recording rules configured for expensive queries
[ ] Alert rules configured (severity, runbook URLs)
[ ] Grafana configured (data source, dashboards, auth)
[ ] Alertmanager configured (routing, grouping, receivers)
[ ] Notifications tested end-to-end
[ ] TLS enabled (Grafana, exporters where exposed)
[ ] Authentication configured (Grafana, Prometheus, Alertmanager)
[ ] Firewall configured (only required ports, source-restricted)
[ ] Secrets protected (no credentials in git)
[ ] Backup strategy defined (Prometheus TSDB, Grafana config)
[ ] Monitoring stack monitored (self-monitoring alerts)
[ ] Capacity planning completed (series, storage, retention)
```

## Best Practices and Lessons Learned

- **Avoid unnecessary high-cardinality labels** — they are the #1 cause of Prometheus memory failures
- **Do not create an alert for every metric** — alerts are for actionable conditions; everything else belongs on a dashboard
- **Use meaningful severity levels** — and make routing depend on them, not on the alert name
- **Configure grouping** — related alerts in one notification beat fifty individual messages
- **Use `for` durations** — they are the cheapest false-positive filter you have
- **Document critical alerts** — a runbook URL on every alert; if it has no runbook, it is not production-ready
- **Monitor the monitoring platform** — self-monitoring first, everything else second
- **Plan retention and storage** — decide on retention *before* the disk fills
- **Test notifications regularly** — monthly heartbeat alerts catch rot early
- **Review alerts periodically** — every quarter, delete alerts that never fired or always fired
- **Validate before reloading** — `promtool check` before every config change in production
- **Pin image versions** — `latest` is not a deployment strategy


## Conclusion

This article covered a complete production monitoring architecture: why the components exist, how they communicate, how to deploy them with Docker Compose, how to design alerts that engineers actually respond to, how to secure the platform, how to troubleshoot it, and how to plan its capacity.

The key engineering decisions worth restating:

- **Metrics-first observability** with Prometheus as the collector and store
- **Decoupled layers** — collection, visualization, and notification are independent services that fail independently
- **Alerts designed for action** — `for` durations, severity routing, runbook URLs, and ruthless false-positive reduction
- **Security by default** — least-privilege network segmentation, TLS, auth, and secrets hygiene
- **Honest scalability** — single-instance where it fits, HA/remote-write architectures when it doesn't

### Realistic Next Steps

- **Kubernetes monitoring** — kube-prometheus-stack for cluster-level observability
- **Long-term metric storage** — Thanos or VictoriaMetrics for months of retention
- **Multi-cluster monitoring** — federated or global-query setups across environments
- **SLO/SLI monitoring** — burn-rate alerting against real error budgets
- **Advanced service discovery** — cloud and K8s native discovery as the fleet grows
- **Centralized observability** — pairing metrics with Loki (logs) and Tempo (traces)

A monitoring stack is never finished. It evolves with the infrastructure it watches — and the engineers who operate it get better at noticing, diagnosing, and fixing problems precisely because the data is there when they need it.

## References

- Prometheus documentation: https://prometheus.io/docs/
- Prometheus configuration reference: https://prometheus.io/docs/prometheus/latest/configuration/configuration/
- Alertmanager configuration: https://prometheus.io/docs/alerting/latest/alertmanager/
- Node Exporter README: https://github.com/prometheus/node_exporter
- Grafana provisioning docs: https://grafana.com/docs/grafana/latest/administration/provisioning/
- Prometheus best practices (naming, labels): https://prometheus.io/docs/practices/naming/
- Recording rules documentation: https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/

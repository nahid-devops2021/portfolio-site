+++
title = "How to Build a Production-Grade Monitoring Stack with Prometheus, Grafana & AlertManager"
date = "2026-08-14"
description = "A complete, battle-tested guide to standing up Prometheus for metrics collection, Grafana for visualization, and AlertManager for alerting — the open-source observability stack that powers modern infrastructure."
tags = ["prometheus", "grafana", "alertmanager", "monitoring", "observability", "docker", "devops"]
categories = ["DevOps"]
author = "Nahid Hasan"
+++

You can't fix what you can't see. Every production system — whether a single VPS or a 100-node Kubernetes cluster — needs observability before it needs features. This post walks through the industry-standard open-source stack: **Prometheus** for metrics, **Grafana** for dashboards, and **AlertManager** for notifications. By the end, you'll have a fully working monitoring stack running in Docker Compose, with alerts landing in your inbox or chat.

<!--more-->

## Why Prometheus, Grafana, and AlertManager?

The stack has become the industry-standard choice for observability, and for good reason:

- **Prometheus** — a high-performance time-series database that scrapes metrics from your servers and applications over HTTP. It's pull-based, which makes it simple to secure and operate, and its query language (PromQL) is powerful enough for complex analytics.
- **Grafana** — the visualization layer. It turns raw time-series data into beautiful, shareable dashboards and is widely used across the industry.
- **AlertManager** — the notification brain. It receives alerts from Prometheus, deduplicates them, groups related ones, and routes them to Slack, Telegram, email, PagerDuty, and more.

Each tool excels at one job, and together they cover the full observability loop: **collect → visualize → alert**.

## The Architecture

Here's the data flow at a glance:

{{< mermaid >}}
graph TB
    A[Servers / Apps] -->|exposes /metrics| B[Prometheus]
    B -->|scrapes every 15s| C[Time-Series DB]
    B --> D[Grafana Dashboards]
    B --> E[AlertManager]
    E --> F[Slack / Email / Telegram]
    C --> D
{{< /mermaid >}}

Prometheus scrapes `/metrics` endpoints, stores the data internally, serves it to Grafana, and evaluates alert rules that get handed to AlertManager when thresholds are breached.

## 1. Docker Compose Setup

The fastest way to stand this up is Docker Compose. Create a `docker-compose.yml` with three services:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"

volumes:
  prometheus-data:
  grafana-data:
```

Then start everything:

```bash
docker compose up -d
```

> **Security note:** change the Grafana default credentials (`admin/admin`) immediately after first login. In production, also consider putting Grafana behind a reverse proxy with TLS.

## 2. Prometheus Configuration

Create `prometheus/prometheus.yml` to define what to scrape:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['192.168.0.43:9100']

  - job_name: 'docker'
    static_configs:
      - targets: ['192.168.0.43:9323']
```

One tip I've learned the hard way: **always set `evaluation_interval` to match your `scrape_interval`.** If alert rules are evaluated on data scraped much earlier, you'll chase false positives that have already resolved.

## 3. Grafana: Connect and Visualize

Grafana runs at `http://localhost:3000`. After logging in with the credentials from the compose file:

1. **Add a data source** → Prometheus → URL: `http://prometheus:9090` (use the container name — it resolves on the compose network).
2. **Import a dashboard** — dashboard **1860** (Node Exporter Full) is an excellent starting point for server metrics.
3. **Build custom panels** for metrics that matter to you.

A few PromQL queries worth bookmarking:

| Metric | PromQL |
|--------|--------|
| **CPU usage %** | `100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` |
| **Memory used (bytes)** | `node_memory_MemTotal_bytes - node_memory_MemFree_bytes - node_memory_Buffers_bytes - node_memory_Cached_bytes` |
| **Disk usage %** | `100 - ((node_filesystem_avail_bytes{mountpoint="/"} * 100) / node_filesystem_size_bytes{mountpoint="/"})` |
| **Network RX rate** | `rate(node_network_receive_bytes_total[5m])` |

## 4. AlertManager: Routing Notifications

AlertManager decides *where* alerts go. Create `alertmanager/alertmanager.yml`:

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'job']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'telegram'

receivers:
  - name: 'telegram'
    telegram_configs:
      - bot_token: 'YOUR_BOT_TOKEN'
        chat_id: YOUR_CHAT_ID
        send_resolved: true
```

Switching to Slack is a one-block change:

```yaml
receivers:
  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts'
        send_resolved: true
```

The `group_by`, `group_wait`, and `repeat_interval` knobs control notification batching — tune them to avoid alert fatigue while never missing a real incident.

## 5. Alert Rules: Define What Matters

Alerts are defined as Prometheus rules. Create `prometheus/alerts.yml`:

```yaml
groups:
  - name: server_alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU usage above 80% on {{ $labels.instance }}"

      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space below 10% on {{ $labels.instance }}"

      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
```

Then reference the rules file from `prometheus.yml`:

```yaml
rule_files:
  - "alerts.yml"
```

The `for` clause is important — it means the condition must persist for that duration before firing, which filters out transient spikes.

## 6. Node Exporter: Collecting Host Metrics

To monitor a server's CPU, memory, disk, and network, run a Node Exporter on each host:

```bash
docker run -d --name node_exporter \
  --restart unless-stopped \
  --network host \
  --pid host \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /:/rootfs:ro \
  prom/node-exporter:latest \
    --path.procfs=/host/proc \
    --path.sysfs=/host/sys \
    --path.rootfs=/rootfs
```

Verify it's exporting metrics at `http://<host>:9100/metrics`, then add it as a scrape target in Prometheus.

## What to Monitor in Production

Don't monitor everything — monitor what breaks. These are the metrics I track across production infrastructure:

| Category | Metrics | Why |
|----------|---------|-----|
| **CPU** | Usage %, load average, iowait | Capacity planning, runaway processes |
| **Memory** | Total, used, cached, swap | OOM prevention |
| **Disk** | Usage %, inode usage, I/O latency | Prevent full disks from crashing apps |
| **Network** | Bandwidth, errors, dropped packets | Detect issues before users do |
| **Docker** | Container count, restart count | Catch crashing containers |
| **Kubernetes** | Pod status, node health, resource quotas | Cluster health |

## Next Steps

Once the basics are running, take it further:

1. **Grafana Loki** — add log aggregation so metrics and logs live side by side
2. **Blackbox Exporter** — monitor external endpoints (HTTP, TCP, ICMP) from the outside
3. **AlertManager silences** — schedule maintenance windows so planned work doesn't page you
4. **Grafana annotations** — mark deployments on dashboards to correlate changes with performance
5. **Service discovery** — in Kubernetes, let Prometheus discover targets automatically via service monitors

## Wrapping Up

A monitoring stack is never "done" — it evolves with your infrastructure. Start with these three services, add exporters as you grow, and iterate on your alert rules until they're quiet during normal operations and loud when something actually breaks.

If you found this useful, check out my other posts on Kubernetes and automation — and feel free to reach out if you have questions or want to share how your own stack looks.

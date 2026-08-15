+++
title = "Setting Up Monitoring: Prometheus + Grafana + AlertManager"
date = "2026-08-14"
description = "A practical guide to setting up Prometheus for metrics collection, Grafana for visualization, and AlertManager for notifications — all with Docker Compose."
tags = ["prometheus", "grafana", "alertmanager", "monitoring", "docker", "devops"]
categories = ["DevOps"]
author = "Nahid Hasan"
+++

## Why This Stack?

Every production system needs monitoring. You can't fix what you can't see. The **Prometheus + Grafana + AlertManager** stack is the industry standard for open-source observability:

- **Prometheus** — collects and stores metrics (time-series database)
- **Grafana** — visualizes metrics on dashboards
- **AlertManager** — sends alerts when things go wrong (email, Slack, Telegram, etc.)

I've set this up multiple times for FinTech and telecom infrastructure. Here's the pattern I use.

<!--more-->

## The Architecture

```
                    ┌─────────────┐
                    │  Servers /  │
                    │  Apps       │
                    └──────┬──────┘
                           │ exposes /metrics
                           ▼
                    ┌─────────────┐
                    │ Prometheus  │  ← scrapes metrics every 15s
                    └──────┬──────┘
                           │
                ┌──────────┼──────────┐
                ▼          ▼          ▼
         ┌──────────┐ ┌────────┐ ┌──────────┐
         │  Grafana │ │Alert   │ │  Other   │
         │  Dashboards │Manager │ │  Tools   │
         └──────────┘ └────────┘ └──────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  Slack/Email │
                    │  /Telegram   │
                    └──────────────┘
```

## 1. Docker Compose Setup

I run this on my home server with Docker Compose. Here's the full stack:

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

## 2. Prometheus Configuration

Create `prometheus/prometheus.yml`:

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

A quick tip I learned the hard way: always set `evaluation_interval` to match your `scrape_interval` — otherwise your alert rules evaluate on stale data and you get false positives.

## 3. Grafana Dashboards

Once Grafana is running at `http://localhost:3000` (default login: `admin/admin`):

1. **Add Prometheus data source** → URL: `http://prometheus:9090`
2. **Import a dashboard** — I use dashboard **1860** (Node Exporter Full) for server metrics
3. **Create custom panels** for your specific needs

My go-to dashboard panels:
- **CPU Usage** — `100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`
- **Memory** — `node_memory_MemTotal_bytes - node_memory_MemFree_bytes - node_memory_Buffers_bytes - node_memory_Cached_bytes`
- **Disk** — `100 - ((node_filesystem_avail_bytes{mountpoint="/"} * 100) / node_filesystem_size_bytes{mountpoint="/"})`
- **Network** — `rate(node_network_receive_bytes_total[5m])`

## 4. AlertManager Configuration

Create `alertmanager/alertmanager.yml`:

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

For **Slack** instead of Telegram, swap the receiver:

```yaml
receivers:
  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts'
        send_resolved: true
```

## 5. Alert Rules

Create `prometheus/alerts.yml` and reference it in your prometheus config:

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

Add it to your `prometheus.yml`:

```yaml
rule_files:
  - "alerts.yml"
```

## 6. Node Exporter (collecting host metrics)

On each server you want to monitor, run:

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

## What I Monitor in Production

Here are the key metrics I track across production infrastructure:

| Category | Metrics | Why |
|----------|---------|-----|
| **CPU** | Usage %, load average, iowait | Capacity planning, runaway processes |
| **Memory** | Total, used, cached, swap | OOM prevention |
| **Disk** | Usage %, inode usage, I/O latency | Prevent full-disks from crashing apps |
| **Network** | Bandwidth, errors, dropped packets | Detect network issues early |
| **Docker** | Container count, restart count | Catch crashing containers |
| **Kubernetes** | Pod status, node health, resource quotas | Cluster health |

## Next Steps

Once you have the basics running:

1. Add **Grafana Loki** for log aggregation (pair logs with metrics for full observability)
2. Set up **Prometheus blackbox exporter** for external endpoint monitoring (HTTP, TCP, ICMP)
3. Configure **AlertManager silences** for planned maintenance windows
4. Use **Grafana annotations** to mark deployments on your dashboards (correlate performance changes with deployments)

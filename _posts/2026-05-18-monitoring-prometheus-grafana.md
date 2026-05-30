---
title: "Monitoring với Prometheus và Grafana: Setup trong 30 phút"
date: 2026-05-18 09:00:00 +0700
categories: [DevOps, Monitoring]
tags: [prometheus, grafana, monitoring, observability, docker]
---

Prometheus thu thập metrics, Grafana hiển thị dashboard. Đây là stack monitoring phổ biến nhất cho infrastructure tự quản lý.

## Kiến trúc tổng quan

```
App/Server → Exporter → Prometheus (scrape) → Grafana (visualize)
                                     ↓
                               AlertManager → Slack/Email
```

## Setup với Docker Compose

```yaml
# docker-compose.yml
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'

volumes:
  prometheus_data:
  grafana_data:
```

## Cấu hình Prometheus scrape

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
        labels:
          environment: 'production'
          server: 'app-01'

  - job_name: 'myapp'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['myapp:8080']

  # Service discovery với Docker
  - job_name: 'docker'
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
    relabel_configs:
      - source_labels: [__meta_docker_container_label_prometheus_scrape]
        regex: 'true'
        action: keep
```

## Expose metrics từ ứng dụng Node.js

```javascript
const express = require('express');
const client = require('prom-client');

const register = new client.Registry();
client.collectDefaultMetrics({ register });

// Custom metric
const httpRequestDuration = new client.Histogram({
    name: 'http_request_duration_seconds',
    help: 'HTTP request duration',
    labelNames: ['method', 'route', 'status_code'],
    buckets: [0.1, 0.3, 0.5, 1, 2, 5],
    registers: [register]
});

// Middleware đo thời gian request
app.use((req, res, next) => {
    const end = httpRequestDuration.startTimer();
    res.on('finish', () => {
        end({ method: req.method, route: req.path, status_code: res.statusCode });
    });
    next();
});

// Endpoint cho Prometheus scrape
app.get('/metrics', async (req, res) => {
    res.set('Content-Type', register.contentType);
    res.send(await register.metrics());
});
```

## PromQL — Query cơ bản

```promql
# CPU usage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage %
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk usage %
100 - ((node_filesystem_avail_bytes * 100) / node_filesystem_size_bytes)

# HTTP error rate
rate(http_requests_total{status_code=~"5.."}[5m]) / rate(http_requests_total[5m]) * 100

# 95th percentile latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

## Alert Rules

```yaml
# alerts.yml
groups:
  - name: infrastructure
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU cao trên {{ $labels.instance }}"
          description: "CPU đang ở {{ $value | printf \"%.1f\" }}%"

      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 15
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Disk sắp đầy: {{ $labels.instance }}"
```

---

Import dashboard có sẵn tại **grafana.com/grafana/dashboards** — Dashboard ID `1860` cho Node Exporter là điểm khởi đầu tốt nhất.

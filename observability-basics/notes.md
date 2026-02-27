# Observability Basics -- Notes

---

## Key Terminology

| Term                | Definition                                                               |
| ------------------- | ------------------------------------------------------------------------ |
| Observability       | Ability to understand internal system state from external outputs        |
| Three pillars       | Logs, metrics, traces                                                    |
| SLI                 | Service Level Indicator -- a measurable aspect of service quality        |
| SLO                 | Service Level Objective -- internal target for an SLI                    |
| SLA                 | Service Level Agreement -- contractual commitment with consequences      |
| Golden signals      | Latency, traffic, errors, saturation                                     |
| Cardinality         | Total number of unique time series; high cardinality increases cost      |
| Span                | A single unit of work within a trace                                     |
| Trace               | End-to-end record of a request across services                           |
| Context propagation | Passing trace IDs across service boundaries (e.g., `traceparent` header) |
| MTTD                | Mean Time to Detect -- how long until an issue is noticed                |
| MTTR                | Mean Time to Resolve -- how long from detection to resolution            |
| Error budget        | Allowed amount of unreliability over a period, derived from SLO          |

---

## PromQL Quick Reference

### Rate and increase

```promql
# Per-second rate of a counter over 5 minutes
rate(http_requests_total[5m])

# Total increase of a counter over 1 hour
increase(http_requests_total[1h])
```

### Aggregation

```promql
# Total request rate across all instances
sum(rate(http_requests_total[5m]))

# Request rate grouped by status code
sum by (status) (rate(http_requests_total[5m]))

# Average memory usage across all pods
avg(container_memory_usage_bytes)
```

### Quantiles from histograms

```promql
# p99 latency over 5 minutes
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# p50 (median) latency grouped by service
histogram_quantile(0.50, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
```

### Useful operators

```promql
# Error ratio
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# Filter series where value exceeds threshold
node_cpu_seconds_total > 0.9

# Absent (fires when a metric disappears -- useful for dead-man alerts)
absent(up{job="my-app"})
```

---

## Prometheus Configuration Essentials

### Minimal prometheus.yml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alerts.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: "app"
    metrics_path: "/metrics"
    scrape_interval: 10s
    static_configs:
      - targets: ["app:8080"]
```

### Alerting rule file (alerts.yml)

```yaml
groups:
  - name: service-alerts
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"

      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Memory usage above 90% on {{ $labels.instance }}"
```

### Common CLI commands

```bash
# Check config syntax
promtool check config prometheus.yml

# Check alerting rules syntax
promtool check rules alerts.yml

# Reload config without restart (requires --web.enable-lifecycle)
curl -X POST http://localhost:9090/-/reload
```

---

## Grafana Dashboard Essentials

### Adding a Prometheus data source

1. Navigate to Configuration > Data Sources.
2. Click "Add data source" and select Prometheus.
3. Set the URL (e.g., `http://prometheus:9090`).
4. Click "Save & Test" to verify connectivity.

### Useful panel types

| Panel       | Best for                                                          |
| ----------- | ----------------------------------------------------------------- |
| Time series | Trends over time (request rate, latency)                          |
| Stat        | Single value with optional sparkline (uptime, current error rate) |
| Gauge       | Current value against a threshold (CPU usage, memory)             |
| Table       | Listing multiple items (top endpoints by error rate)              |
| Heatmap     | Distribution over time (latency buckets)                          |
| Logs        | Viewing logs from Loki data source inline                         |

### Template variables

Template variables make dashboards reusable. Define a variable in Dashboard Settings > Variables:

- **Type:** Query
- **Data source:** Prometheus
- **Query:** `label_values(up, job)` -- returns all unique `job` label values
- **Usage in panel:** `rate(http_requests_total{job="$job"}[5m])`

---

## Loki LogQL Quick Reference

Loki is Grafana's log aggregation system. LogQL is its query language.

### Stream selectors

```logql
# Select logs from a specific app
{app="api"}

# Multiple label matchers
{app="api", namespace="production"}

# Regex match
{app=~"api|worker"}
```

### Filter expressions

```logql
# Line contains "error"
{app="api"} |= "error"

# Line does not contain "debug"
{app="api"} != "debug"

# Regex filter
{app="api"} |~ "status=[45].."
```

### Structured metadata parsing

```logql
# Parse JSON logs and filter on fields
{app="api"} | json | level="error"

# Parse and extract a field
{app="api"} | json | duration > 1s
```

### Log metric queries

```logql
# Count error log lines per second
rate({app="api"} |= "error" [5m])

# Count log lines by extracted level field
sum by (level) (count_over_time({app="api"} | json [5m]))
```

---

## OpenTelemetry Architecture

```
+------------------+       +-------------------+       +------------------+
|  Application     |       |  OTel Collector   |       |  Backends        |
|  (OTel SDK)      | --->  |  (receive,        | --->  |  - Prometheus    |
|                  | OTLP  |   process,        |       |  - Jaeger/Tempo  |
|  Logs, Metrics,  |       |   export)         |       |  - Loki          |
|  Traces          |       |                   |       |  - Datadog, etc. |
+------------------+       +-------------------+       +------------------+
```

### Collector pipeline

The OTel Collector processes telemetry through three stages:

1. **Receivers** -- accept data (OTLP, Prometheus scrape, Jaeger, Zipkin, Fluent Forward).
2. **Processors** -- transform data (batching, filtering, attribute manipulation, sampling).
3. **Exporters** -- send data to backends (OTLP, Prometheus remote write, Jaeger, Loki).

### Minimal collector config (otel-collector-config.yaml)

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: "0.0.0.0:4317"
      http:
        endpoint: "0.0.0.0:4318"

processors:
  batch:
    timeout: 5s
    send_batch_size: 1024

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
  otlp/jaeger:
    endpoint: "jaeger:4317"
    tls:
      insecure: true

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger]
```

---

## Common Ports Reference

| Service                 | Default Port |
| ----------------------- | ------------ |
| Prometheus              | 9090         |
| Alertmanager            | 9093         |
| node_exporter           | 9100         |
| Grafana                 | 3000         |
| Jaeger UI               | 16686        |
| Jaeger Collector (gRPC) | 14250        |
| OTel Collector (gRPC)   | 4317         |
| OTel Collector (HTTP)   | 4318         |
| Elasticsearch           | 9200         |
| Kibana                  | 5601         |
| Loki                    | 3100         |

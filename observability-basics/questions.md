# Observability Basics -- Interview Questions

---

## Q1: What is observability, and how does it differ from monitoring?

**Observability** is the ability to understand the internal state of a system by examining its external outputs -- logs, metrics, and traces. It lets you ask arbitrary, new questions about system behavior without deploying new instrumentation for every question.

**Monitoring** is a subset of observability. It focuses on collecting predefined metrics, setting thresholds, and alerting when known failure conditions occur.

| Aspect         | Monitoring                             | Observability                                        |
| -------------- | -------------------------------------- | ---------------------------------------------------- |
| Focus          | Known failure modes                    | Unknown unknowns                                     |
| Approach       | Dashboards and alerts on fixed metrics | Explore data to diagnose novel issues                |
| Question style | "Is CPU above 90%?"                    | "Why is request latency high for users in region X?" |

In short, monitoring tells you **when** something is wrong; observability helps you figure out **why**.

---

## Q2: What are the three pillars of observability?

The three pillars are **logs**, **metrics**, and **traces**.

### Logs

Discrete, timestamped records of events. They can be unstructured (plain text) or structured (JSON). Logs provide the richest detail about what happened inside a service at a specific moment.

### Metrics

Numeric measurements collected over time (time-series data). Metrics are lightweight, aggregatable, and ideal for dashboards and alerting. Examples: request rate, error count, CPU utilization.

### Traces

Records of a request's journey through a distributed system. A trace is composed of **spans**, where each span represents a unit of work (e.g., an HTTP call, a database query). Traces let you pinpoint which service or operation in a chain is slow or failing.

---

## Q3: What is the difference between structured and unstructured logging?

**Unstructured logs** are free-form text strings:

```
2026-02-27 10:15:03 ERROR Failed to connect to database host=db01 timeout=5s
```

**Structured logs** use a consistent, machine-parseable format like JSON:

```json
{
  "timestamp": "2026-02-27T10:15:03Z",
  "level": "ERROR",
  "message": "Failed to connect to database",
  "host": "db01",
  "timeout_seconds": 5
}
```

Structured logging is preferred in production because:

- It is easier to parse, filter, and query in log aggregation systems (Elasticsearch, Loki).
- It enables consistent field names across services.
- It avoids brittle regex-based parsing.

---

## Q4: What are the main metric types in Prometheus?

Prometheus defines four core metric types:

### Counter

A monotonically increasing value that only goes up (or resets to zero on process restart). Used for things that accumulate, like total HTTP requests served.

```
http_requests_total{method="GET", status="200"} 14832
```

### Gauge

A value that can go up or down. Used for things like current memory usage, temperature, or active connections.

```
node_memory_MemAvailable_bytes 4194304000
```

### Histogram

Samples observations (e.g., request durations) and counts them in configurable buckets. Useful for calculating quantiles server-side. Prometheus creates `_bucket`, `_sum`, and `_count` sub-metrics automatically.

### Summary

Similar to a histogram but calculates quantiles (e.g., p50, p99) on the client side. Summaries are less flexible because the quantiles are fixed at instrumentation time and cannot be aggregated across instances.

**Rule of thumb:** Prefer histograms over summaries for most use cases because they support aggregation and allow recalculating quantiles at query time.

---

## Q5: What are SLIs, SLOs, and SLAs?

### SLI (Service Level Indicator)

A quantitative measurement of a specific aspect of service quality. Examples: request latency at the 99th percentile, error rate, availability percentage.

### SLO (Service Level Objective)

A target value or range for an SLI that the team commits to internally. Example: "99.9% of requests complete in under 300ms over a rolling 30-day window."

### SLA (Service Level Agreement)

A formal contract with customers that defines consequences (usually financial) if SLOs are not met. Example: "If uptime drops below 99.95% in a calendar month, the customer receives a 10% service credit."

The relationship:

```
SLI (what you measure) --> SLO (what you target) --> SLA (what you promise)
```

---

## Q6: What are the four golden signals?

Defined by the Google SRE book, the four golden signals are the most important metrics to monitor for any user-facing service:

### 1. Latency

The time it takes to serve a request. It is important to distinguish between the latency of successful requests and the latency of failed requests (a fast error is still an error).

### 2. Traffic

The demand placed on the system. For a web service, this is typically measured in HTTP requests per second. For a streaming service, it might be network I/O or concurrent sessions.

### 3. Errors

The rate of requests that fail, either explicitly (HTTP 5xx), implicitly (HTTP 200 with wrong content), or by policy (any response slower than a threshold).

### 4. Saturation

How "full" the service is. This measures resource utilization against capacity -- CPU, memory, disk I/O, open file descriptors. High saturation typically leads to degraded performance.

---

## Q7: What is Prometheus and how does it collect metrics?

Prometheus is an open-source time-series monitoring and alerting system. It uses a **pull-based** model: Prometheus scrapes HTTP endpoints (typically `/metrics`) exposed by instrumented applications at regular intervals.

Key components:

- **Prometheus server** -- scrapes and stores time-series data in its local TSDB.
- **Exporters** -- processes that expose metrics from third-party systems (e.g., `node_exporter` for Linux host metrics, `blackbox_exporter` for probing endpoints).
- **Alertmanager** -- handles alerts fired by Prometheus rules, routing them to email, Slack, PagerDuty, etc.
- **PromQL** -- the query language used to select, aggregate, and transform metrics.

A minimal scrape configuration in `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: "my-app"
    scrape_interval: 15s
    static_configs:
      - targets: ["app-server:8080"]
```

---

## Q8: What is the difference between pull-based and push-based metrics collection?

### Pull-based (Prometheus model)

The monitoring server periodically scrapes metrics from application endpoints. The application exposes a `/metrics` endpoint and does not need to know about the monitoring infrastructure.

**Advantages:** Simpler application code, easy to detect if a target is down (scrape fails), monitoring config is centralized.

### Push-based (StatsD, Datadog Agent, OTLP push model)

The application pushes metrics to a collector or aggregator. The application must be configured with the collector's address.

**Advantages:** Works better for short-lived jobs (batch processes, lambdas) that may finish before a scrape occurs. Also works in environments where the monitoring server cannot reach the application (e.g., behind a firewall).

Prometheus provides **Pushgateway** for short-lived jobs that cannot be scraped. The job pushes its metrics to Pushgateway, and Prometheus scrapes the gateway.

---

## Q9: What is Grafana and what role does it play?

Grafana is an open-source visualization and dashboarding platform. It does not store data itself; it connects to data sources and visualizes the data.

Common data sources:

- Prometheus (metrics)
- Loki (logs)
- Elasticsearch (logs, search)
- Jaeger or Tempo (traces)
- PostgreSQL, MySQL (relational data)

Key concepts:

- **Dashboard** -- a collection of panels organized on a grid.
- **Panel** -- a single visualization (graph, gauge, table, heatmap, etc.).
- **Variables** -- template variables that let users filter dashboards dynamically (e.g., select a namespace, service, or instance from a dropdown).
- **Alerting** -- Grafana can evaluate alert rules against data sources and send notifications.

---

## Q10: What is the ELK stack?

ELK stands for **Elasticsearch**, **Logstash**, and **Kibana**:

- **Elasticsearch** -- a distributed search and analytics engine that stores and indexes log data.
- **Logstash** -- a data processing pipeline that ingests, transforms, and forwards logs to Elasticsearch.
- **Kibana** -- a web UI for searching, visualizing, and building dashboards over data in Elasticsearch.

A common variation is the **EFK stack**, which replaces Logstash with **Fluentd** or **Fluent Bit**. Fluentd is lighter, Kubernetes-native, and often preferred in containerized environments.

Typical data flow:

```
Application --> Fluentd/Fluent Bit (collect, parse, filter) --> Elasticsearch (store, index) --> Kibana (visualize)
```

---

## Q11: What is distributed tracing?

Distributed tracing tracks a single request as it flows through multiple services in a microservices architecture. Each unit of work creates a **span**. Spans are linked together into a **trace** via a shared **trace ID** that propagates across service boundaries through HTTP headers or gRPC metadata.

Key terms:

- **Trace** -- the end-to-end record of a request across all services.
- **Span** -- a single operation within a trace (e.g., "POST /api/orders", "SELECT from orders table").
- **Parent span / child span** -- spans form a tree; a service calling another service creates a child span.
- **Context propagation** -- the mechanism that passes trace IDs between services (e.g., the `traceparent` header in the W3C Trace Context standard).

Popular tracing tools: **Jaeger**, **Zipkin**, **Grafana Tempo**.

---

## Q12: What is OpenTelemetry (OTel)?

OpenTelemetry is a vendor-neutral, open-source observability framework. It provides standardized APIs, SDKs, and tools for generating, collecting, and exporting telemetry data (logs, metrics, and traces).

Core components:

- **OTel SDK** -- libraries you add to your application to instrument it. Available for Go, Java, Python, Node.js, .NET, and more.
- **OTel Collector** -- a standalone process that receives, processes, and exports telemetry data. It acts as a pipeline between your applications and your backends.
- **OTLP (OpenTelemetry Protocol)** -- the native protocol for transmitting telemetry data. Supports gRPC and HTTP.

Why it matters:

- Avoids vendor lock-in. You instrument once with OTel and can export to Prometheus, Jaeger, Datadog, New Relic, or any compatible backend.
- Merges the previously separate OpenTracing and OpenCensus projects into a single standard.

---

## Q13: What are labels (or tags) and why does cardinality matter?

**Labels** (Prometheus terminology) or **tags** (Datadog, StatsD terminology) are key-value pairs attached to a metric to add dimensions. For example:

```
http_requests_total{method="GET", status="200", service="api"}
```

Here `method`, `status`, and `service` are labels. Each unique combination of label values creates a separate time series.

**Cardinality** is the total number of unique time series. High cardinality occurs when a label has many possible values (e.g., user IDs, request paths with UUIDs). This dramatically increases storage and memory consumption in the metrics backend.

**Best practices:**

- Avoid labels with unbounded values (user IDs, email addresses, full URLs).
- Group low-frequency values under an "other" bucket where possible.
- Use logs or traces (not metrics) for high-cardinality investigation.

---

## Q14: What is an exporter in the Prometheus ecosystem?

An exporter is a process that collects metrics from a third-party system and exposes them in Prometheus format on an HTTP endpoint.

Common exporters:

| Exporter             | What it monitors                                   |
| -------------------- | -------------------------------------------------- |
| `node_exporter`      | Linux host metrics (CPU, memory, disk, network)    |
| `blackbox_exporter`  | Probes endpoints (HTTP, TCP, ICMP, DNS)            |
| `mysqld_exporter`    | MySQL database metrics                             |
| `postgres_exporter`  | PostgreSQL database metrics                        |
| `redis_exporter`     | Redis server metrics                               |
| `kube-state-metrics` | Kubernetes object state (pods, deployments, nodes) |

Install and run `node_exporter` on RHEL:

```bash
# Download and extract
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar xvfz node_exporter-1.8.1.linux-amd64.tar.gz

# Run
./node_exporter-1.8.1.linux-amd64/node_exporter &

# Verify metrics endpoint
curl http://localhost:9100/metrics | head
```

---

## Q15: What is alerting, and what is alert fatigue?

**Alerting** is the practice of automatically notifying operators when a metric crosses a threshold or a condition is met. In Prometheus, alerting is configured with **alerting rules** that evaluate PromQL expressions.

Example Prometheus alerting rule:

```yaml
groups:
  - name: example
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error rate above 5% for 5 minutes"
```

**Alert fatigue** occurs when operators receive too many alerts, many of which are noisy or non-actionable. This leads to ignored alerts and slower response to real incidents.

Ways to reduce alert fatigue:

- Alert on **symptoms** (high error rate, high latency) rather than **causes** (high CPU).
- Use the `for` duration to avoid firing on brief spikes.
- Group related alerts in Alertmanager to reduce notification volume.
- Regularly review and prune alerts that are never acted upon.

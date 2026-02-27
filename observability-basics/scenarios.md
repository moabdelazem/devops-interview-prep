# Observability Basics -- Scenario Questions

---

## Scenario 1: Debugging a Latency Spike

**Situation:** Users report that your web application has become slow over the past 30 minutes. You have Prometheus, Grafana, and Jaeger deployed.

**Question:** Walk through how you would diagnose the root cause.

**Answer:**

1. **Check the golden signals dashboard in Grafana.** Look at the latency panel (p50, p95, p99) for the affected service. Confirm the spike is real and identify when it started.

2. **Check traffic and error rate panels.** Determine whether the latency spike correlates with increased traffic or a rise in error rates.

3. **Check saturation.** Review CPU, memory, and disk I/O for the service's pods or hosts. High saturation often explains latency increases.

4. **Narrow down the service.** If you have multiple services, use per-service dashboards or PromQL to isolate which service introduced the latency:

   ```promql
   histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{service="api"}[5m]))
   ```

5. **Inspect traces in Jaeger.** Filter traces by the affected service and the time window. Sort by duration to find the slowest traces. Examine the span waterfall to see which downstream call (database, external API, another microservice) is taking the most time.

6. **Check logs.** Once you identify the problematic span, check the corresponding service logs for errors, timeouts, or connection issues during that window.

---

## Scenario 2: Setting Up Basic Prometheus Monitoring

**Situation:** You have a new Go application exposing metrics at `/metrics` on port 8080. You need to set up Prometheus to scrape it and create a basic Grafana dashboard.

**Question:** Describe the steps to get monitoring working.

**Answer:**

1. **Add the scrape target to `prometheus.yml`:**

   ```yaml
   scrape_configs:
     - job_name: "my-go-app"
       scrape_interval: 15s
       static_configs:
         - targets: ["my-go-app:8080"]
   ```

2. **Reload or restart Prometheus:**

   ```bash
   # If running as a systemd service on RHEL
   sudo systemctl reload prometheus

   # Or send SIGHUP
   kill -HUP $(pidof prometheus)
   ```

3. **Verify the target is being scraped.** Open `http://<prometheus>:9090/targets` and confirm the target state is `UP`.

4. **Add Prometheus as a data source in Grafana.** Go to Configuration > Data Sources > Add data source > Prometheus. Set the URL to `http://prometheus:9090`.

5. **Create a dashboard.** Add panels for key metrics:
   - Request rate: `rate(http_requests_total{job="my-go-app"}[5m])`
   - Error rate: `rate(http_requests_total{job="my-go-app", status=~"5.."}[5m])`
   - Latency p99: `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="my-go-app"}[5m]))`

---

## Scenario 3: Investigating a Spike in 5xx Errors

**Situation:** Alertmanager fires a `HighErrorRate` alert for your API service. The error rate exceeded 5% over a 5-minute window.

**Question:** How would you triage this?

**Answer:**

1. **Acknowledge the alert** in your incident channel and check the alert details for context (which service, which labels matched).

2. **Open the Grafana dashboard** for the affected service. Check:
   - Is the error rate still climbing, stable, or recovering?
   - Did traffic spike at the same time (could indicate the service is overwhelmed)?
   - Which HTTP status codes are contributing? Use a breakdown query:

     ```promql
     sum by (status) (rate(http_requests_total{job="api", status=~"5.."}[5m]))
     ```

3. **Check recent deployments.** A new deploy is the most common cause of sudden error spikes. Check your CI/CD system or `kubectl rollout history` for recent changes:

   ```bash
   kubectl rollout history deployment/api -n production
   ```

4. **Check application logs.** Filter logs for the affected service and time window, looking for stack traces, connection refused errors, or timeout messages:

   ```bash
   # If using Loki via LogCLI
   logcli query '{app="api"} |= "error"' --since=30m
   ```

5. **Check dependencies.** If the logs show connection errors to a database or downstream service, check that dependency's health.

6. **Mitigate.** If the error correlates with a deploy, roll back. If a dependency is down, check its status and escalate. If the service is overwhelmed, consider scaling:

   ```bash
   kubectl scale deployment/api --replicas=5 -n production
   ```

---

## Scenario 4: Choosing Alert Thresholds

**Situation:** You are setting up alerting for a new service. The product team tells you the service should respond to 99% of requests within 500ms.

**Question:** How would you translate this into a Prometheus alerting rule, and what would you consider when setting the threshold?

**Answer:**

Define the SLO first: "99% of requests complete within 500ms over a rolling window."

Create an alerting rule that fires when the SLO is at risk:

```yaml
groups:
  - name: latency-slo
    rules:
      - alert: LatencySLOBreach
        expr: |
          histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="my-service"}[10m])) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "p99 latency exceeds 500ms for 10 minutes"
```

Considerations:

- **Use `for` duration** to avoid alerting on transient spikes. A 10-minute `for` window means the condition must be sustained before the alert fires.
- **Choose the right evaluation window.** A `rate()` window of 5-10 minutes smooths out noise. Too short (1m) produces flappy alerts; too long (1h) makes the alert slow to fire.
- **Set severity levels.** A `warning` alert at p99 > 500ms for 10 minutes, and a `critical` alert at p99 > 1s for 5 minutes.
- **Burn-rate alerting** (advanced) measures how fast you are consuming your error budget. It is more robust than simple threshold alerts for SLO-based monitoring.

---

## Scenario 5: Correlating Logs, Metrics, and Traces

**Situation:** A customer reports intermittent 504 Gateway Timeout errors. Your architecture has an NGINX ingress, an API service, and a PostgreSQL database. You have all three pillars instrumented.

**Question:** How do you correlate data from logs, metrics, and traces to find the root cause?

**Answer:**

1. **Start with metrics.** Check the NGINX ingress error rate dashboard for 504 responses. Identify the time window and confirm the issue:

   ```promql
   rate(nginx_http_requests_total{status="504"}[5m])
   ```

2. **Check API latency metrics.** If API latency spikes correlate with the 504s, the timeout is likely happening between NGINX and the API:

   ```promql
   histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{service="api"}[5m]))
   ```

3. **Find sample traces.** In Jaeger, search for traces from the affected time window with a duration exceeding the NGINX timeout (e.g., > 60s). Open a slow trace and examine the span tree. If the longest span is a database query, the database is the bottleneck.

4. **Correlate with database metrics.** Check PostgreSQL metrics for active connections, query duration, and lock waits:

   ```promql
   pg_stat_activity_count{state="active"}
   pg_stat_activity_max_tx_duration_seconds
   ```

5. **Check logs with the trace ID.** Copy the trace ID from Jaeger and search your log aggregation system:

   ```bash
   # Loki
   logcli query '{app="api"} |= "trace_id=abc123def456"'
   ```

   The logs may reveal a slow query, a connection pool exhaustion, or a lock contention error.

6. **Confirm and fix.** If you find a slow query, add an index or optimize the query. If connection pool exhaustion, increase the pool size or fix connection leaks. Verify the fix by watching the metrics dashboard after the change.

---

## Scenario 6: Prometheus Target Is DOWN

**Situation:** You check the Prometheus targets page and see one of your application targets in a `DOWN` state. The `last error` column shows `connection refused`.

**Question:** How would you troubleshoot this?

**Answer:**

1. **Verify the application is running:**

   ```bash
   # If running as a systemd service on RHEL
   sudo systemctl status my-app

   # If running in Kubernetes
   kubectl get pods -l app=my-app -n production
   ```

2. **Check the metrics endpoint manually from the Prometheus server:**

   ```bash
   curl -v http://<target-host>:8080/metrics
   ```

   If `connection refused`, the application is either not running, not listening on the expected port, or a firewall is blocking the connection.

3. **Check firewall rules on RHEL:**

   ```bash
   sudo firewall-cmd --list-ports
   sudo firewall-cmd --add-port=8080/tcp --permanent
   sudo firewall-cmd --reload
   ```

4. **Check if the application is listening on the correct interface.** An app bound to `127.0.0.1` will refuse connections from Prometheus on another host. It should bind to `0.0.0.0`:

   ```bash
   ss -tlnp | grep 8080
   ```

5. **In Kubernetes, check network policies and service configuration.** Verify the Service selector matches the Pod labels and that no NetworkPolicy is blocking Prometheus scrapes.

---

## Scenario 7: Dashboard Shows No Data

**Situation:** You create a Grafana dashboard panel using the query `rate(http_requests_total[5m])`, but the panel shows "No data."

**Question:** What could cause this, and how would you debug it?

**Answer:**

1. **Check the data source.** Ensure the panel is using the correct Prometheus data source. If you have multiple Prometheus instances, you may be querying the wrong one.

2. **Run the query directly in Prometheus.** Go to `http://<prometheus>:9090/graph`, paste the query, and execute it. If it returns data there, the issue is in Grafana's data source configuration.

3. **Check the metric name.** The metric may not exist or may have a different name. In Prometheus, start typing `http_requests` in the expression browser to see autocomplete suggestions.

4. **Check the time range.** Grafana defaults to the last 6 hours. If your application just started, there may not be enough data. Adjust the time range in Grafana's time picker.

5. **Check label filters.** If your query includes label matchers (e.g., `{job="api"}`), verify the label values exactly match what Prometheus has. Labels are case-sensitive.

6. **Check the scrape interval.** `rate()` requires at least two data points to compute a rate. If your scrape interval is 60s and the `rate` window is `[1m]`, you may not have enough samples. Use a window at least 4x the scrape interval (e.g., `[5m]` for a 60s interval).

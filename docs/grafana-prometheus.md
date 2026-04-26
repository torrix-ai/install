# Grafana / Prometheus

Torrix exposes a `/metrics` endpoint in Prometheus text format. Scrape it to build Grafana dashboards using your existing monitoring stack.

---

## Scrape the metrics endpoint

```bash
curl http://localhost:8088/metrics -H "Authorization: Bearer <your-torrix-api-key>"
```

Example output:
```
torrix_requests_total 142
torrix_cost_usd_total 0.023400
torrix_tokens_total 58300
torrix_errors_total 2
torrix_latency_p50_ms 312
torrix_latency_p95_ms 891
torrix_latency_p99_ms 1423
torrix_requests_by_model{model="gpt-4o-mini"} 98
torrix_requests_by_model{model="claude-3-5-sonnet-20241022"} 44
```

---

## Prometheus configuration

Add this to your `prometheus.yml` scrape config:

```yaml
scrape_configs:
  - job_name: torrix
    scrape_interval: 30s
    static_configs:
      - targets: ['host.docker.internal:8088']
    metrics_path: /metrics
    authorization:
      credentials: <your-torrix-api-key>
```

---

## Grafana dashboard

Once Prometheus is scraping, create a Grafana dashboard using the `torrix_*` metrics. Useful panels:

- `torrix_requests_total` — total request count over time
- `torrix_cost_usd_total` — cumulative spend
- `torrix_errors_total` — error rate
- `torrix_latency_p50_ms` / `p95` / `p99` — latency percentiles
- `torrix_requests_by_model` — breakdown by model

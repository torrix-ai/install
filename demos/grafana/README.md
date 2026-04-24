# Demo: Torrix Metrics in Grafana

This demo shows how to scrape Torrix LLM metrics into Prometheus and visualise them in Grafana. You will be able to chart request volume, total cost, token usage, latency percentiles, and requests broken down by model over time.

## Prerequisites

Make sure Torrix is running at [http://localhost:8088](http://localhost:8088) and has at least a few LLM calls logged. If you have not installed Torrix yet:

```bash
curl -o docker-compose.yml https://raw.githubusercontent.com/torrix-ai/install/main/docker-compose.community.yml
docker compose up
```

You also need Docker installed to run Prometheus and Grafana.

## Step 1: Get your Torrix API key

Go to [http://localhost:8088](http://localhost:8088), open **Settings**, and copy your API key. It starts with `trxk_`.

## Step 2: Add your API key to prometheus.yml

Open `prometheus.yml` from this folder and replace `YOUR_TORRIX_API_KEY_HERE` with your actual key:

```yaml
authorization:
  credentials: trxk_your_actual_key_here
```

Save the file.

## Step 3: Start Prometheus and Grafana

From this folder, run:

```bash
docker compose up
```

This starts two containers. Prometheus is available at [http://localhost:9090](http://localhost:9090) and Grafana at [http://localhost:3000](http://localhost:3000).

On Linux, the `extra_hosts` entry in docker-compose.yml maps `host.docker.internal` to your machine automatically. On Mac and Windows this mapping is built into Docker Desktop.

## Step 4: Verify Prometheus is scraping Torrix

Open [http://localhost:9090/targets](http://localhost:9090/targets). You should see a `torrix` job with status **UP**. If it shows **DOWN**, check that your API key is correct in prometheus.yml and that Torrix is running.

To confirm the metrics are coming through, go to [http://localhost:9090](http://localhost:9090), type `torrix_requests_total` in the query box, and click **Execute**. You should see a value matching the total run count in your Torrix dashboard.

## Step 5: Connect Grafana to Prometheus

1. Open Grafana at [http://localhost:3000](http://localhost:3000) and log in with username `admin` and password `admin`
2. Go to **Connections** in the left sidebar and click **Add new connection**
3. Search for **Prometheus** and click **Add new data source**
4. Set the URL to `http://prometheus:9090` (Prometheus container name, not localhost)
5. Click **Save and test**. You should see a green confirmation message.

## Step 6: Explore Torrix metrics

Go to **Explore** in Grafana (compass icon in the sidebar), select Prometheus as the data source, and run any of these queries:

**Total requests over time**
```
increase(torrix_requests_total[5m])
```

**Cumulative cost in USD**
```
torrix_cost_usd_total
```

**Requests by model**
```
torrix_requests_by_model
```

**p95 latency**
```
torrix_latency_p95_ms
```

**Error count**
```
torrix_errors_total
```

## Step 7: Build a dashboard

1. Click **Dashboards** in the sidebar and then **New Dashboard**
2. Click **Add visualization**
3. Select Prometheus as the data source
4. Enter a query from Step 6, for example `torrix_requests_by_model`
5. Set the panel title and click **Apply**
6. Repeat for each metric and click **Save dashboard**

## Available metrics

| Metric | Type | Description |
|---|---|---|
| `torrix_requests_total` | counter | Total LLM requests logged |
| `torrix_cost_usd_total` | counter | Total cost across all requests in USD |
| `torrix_tokens_total` | counter | Total tokens used (input and output) |
| `torrix_errors_total` | counter | Total requests that returned an error |
| `torrix_latency_p50_ms` | gauge | Median response latency in milliseconds |
| `torrix_latency_p95_ms` | gauge | p95 response latency in milliseconds |
| `torrix_latency_p99_ms` | gauge | p99 response latency in milliseconds |
| `torrix_requests_by_model` | gauge | Request count labelled by model name |

## Stopping the demo

```bash
docker compose down
```

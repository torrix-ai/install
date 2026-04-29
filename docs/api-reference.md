# Torrix API Reference

Base URL: `http://localhost:8088` (or your configured host)

All authenticated endpoints require `Authorization: Bearer trxk_...` header.

---

## Proxy

### POST /proxy

Forward LLM requests through Torrix for automatic logging.

**Headers:**
| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | `Bearer trxk_...` (Torrix API key) |
| `x-target-url` | Yes | Upstream LLM API URL |
| `x-upstream-authorization` | Yes | Auth for the upstream (e.g. `Bearer sk-...`) |
| `x-torrix-name` | No | Human-readable name for this run |
| `x-torrix-trace` | No | Trace ID for grouping related runs |
| `x-torrix-session` | No | Session ID for conversation grouping |

**Body:** Pass the upstream request body as-is (JSON).

**Response:** Returns the upstream response verbatim. Torrix logs tokens, cost, and latency in the background.

```bash
curl http://localhost:8088/proxy \
  -H "Authorization: Bearer trxk_YOUR_KEY" \
  -H "x-target-url: https://api.openai.com/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer sk-..." \
  -H "x-torrix-name: summarize-email" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Hello"}]}'
```

---

## SDK Ingest

### POST /api/ingest

Receive telemetry from SDKs (Python, TypeScript).

**Body (JSON):**
```json
{
  "run_id": "uuid",
  "name": "my-task",
  "provider": "openai",
  "model": "gpt-4o",
  "input_tokens": 150,
  "output_tokens": 80,
  "cost_usd": 0.00125,
  "latency_ms": 1200,
  "status": 200,
  "finish_reason": "stop",
  "request_bytes": 512,
  "response_bytes": 1024,
  "prompt": "What is...",
  "response": "The answer is...",
  "summary": "HTTP 200 · 1200ms · model: gpt-4o",
  "source": "sdk",
  "trace_id": "optional-trace",
  "session_id": "optional-session"
}
```

**Response:** `{ "ok": true, "run_id": "..." }`

---

## Dashboard

### GET /api/dashboard

Aggregated stats for the dashboard.

**Response:**
```json
{
  "total_runs": 1234,
  "total_cost": 5.67,
  "total_tokens": 890000,
  "avg_latency_ms": 1500,
  "runs_today": 45,
  "cost_today": 0.89,
  "models": [{ "model": "gpt-4o", "count": 100, "cost": 2.50 }],
  "providers": [{ "provider": "openai", "count": 200 }],
  "hourly": [{ "hour": "2024-01-15T10:00:00Z", "count": 12, "cost": 0.15 }]
}
```

---

## Runs

### GET /api/runs

Paginated list of runs.

**Query params:**
| Param | Default | Description |
|-------|---------|-------------|
| `limit` | 50 | Max results (1-500) |
| `offset` | 0 | Pagination offset |

**Response:**
```json
{
  "runs": [{ "id": "...", "name": "...", "model": "gpt-4o", "provider": "openai", "status": 200, "cost_usd": 0.001, "latency_ms": 1200, "created_at": "..." }],
  "total": 1234,
  "hasMore": true
}
```

### GET /api/runs/:id

Single run detail.

**Response:** Full run object including `input_tokens`, `output_tokens`, `thinking_tokens`, `prompt`, `response`, `trace_id`, `session_id`.

### GET /api/runs/:id/events

Events (messages) for a specific run.

**Response:** Array of event objects with `role`, `message`, `created_at`.

### GET /api/runs/:id/cost-comparison

Compare the cost of this run across different models.

**Response:** Array of `{ model, label, provider, cost, savings, savingsPct, actualCostKnown }`.

### GET /api/runs/export.csv

Export all runs as CSV.

### GET /api/runs/compare?ids=id1,id2

Side-by-side comparison of multiple runs.

---

## Search (Pro)

### GET /api/search

Full-text search across prompts and responses.

**Query params:**
| Param | Required | Description |
|-------|----------|-------------|
| `q` | Yes | Search query (min 2 chars) |
| `limit` | No | Max results (default 50, max 100) |

**Response:** Array of matching runs with `snippet` field containing highlighted matches.

**Edition:** Pro only. Returns 403 on Community.

---

## Traces and Sessions

### GET /api/traces/:traceId

All runs belonging to a trace.

**Response:** `{ "trace_id": "...", "runs": [...] }`

### GET /api/sessions/:sessionId

All runs belonging to a session.

**Response:** `{ "session_id": "...", "runs": [...] }`

---

## Configuration

### GET /api/config

Server configuration and usage stats.

**Response:**
```json
{
  "edition": "community",
  "editionLabel": "Community",
  "maxRuns": 100,
  "retentionDays": 7,
  "runCount": 45,
  "nextWipeAt": 1700000000000
}
```

---

## Alerts

### GET /api/alert-settings

Current alert/webhook configuration.

**Response:**
```json
{
  "budget_usd": 10,
  "hard_cap_usd": null,
  "webhook_url": "https://hooks.slack.com/...",
  "enabled": 1,
  "notify_on_error": 1,
  "webhook_secret": "abc123..."
}
```

### PUT /api/alert-settings

Update alert configuration.

**Body:**
```json
{
  "budget_usd": 10,
  "webhook_url": "https://hooks.slack.com/...",
  "enabled": true,
  "notify_on_error": true
}
```

---

## License

### GET /api/license

Current license status.

### POST /api/license

Activate a license key.

**Body:** `{ "key": "TRX-PRO-..." }`

### DELETE /api/license

Deactivate the current license.

---

## Streaming (SSE)

### GET /api/stream

Server-Sent Events stream for real-time run updates.

**Query params:** `key` (API key, alternative to Authorization header for EventSource).

**Events:** Each event is a JSON run object.

```javascript
const es = new EventSource('/api/stream?key=trxk_...');
es.onmessage = (e) => console.log(JSON.parse(e.data));
```

---

## OpenTelemetry

### POST /v1/traces

Accepts OTLP/HTTP trace data (JSON format). Maps LLM spans to Torrix runs using semantic conventions (`gen_ai.*` attributes).

---

## Health

### GET /health

Returns `{ "status": "ok" }`. Used by Docker HEALTHCHECK.

---

## Metrics (Prometheus)

### GET /metrics

Prometheus-compatible metrics endpoint.

---

## Rate Limits

All endpoints are rate-limited:
- **Global:** 300 req/min
- **Auth endpoints** (`/auth/*`): 10 req/min
- **Proxy/Ingest:** 120 req/min (Community), 300 req/min (Pro)

Rate limit headers: `x-ratelimit-limit`, `x-ratelimit-remaining`, `x-ratelimit-reset`.

---

## Webhook Signatures

Outbound webhooks include an `x-torrix-signature` header with HMAC-SHA256 of the request body. Verify with:

```javascript
const crypto = require('crypto');
const expected = 'sha256=' + crypto.createHmac('sha256', webhookSecret)
  .update(rawBody).digest('hex');
if (signature !== expected) throw new Error('Invalid signature');
```

The signing secret is available in `GET /api/alert-settings` response.

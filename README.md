# Torrix: AI Observability

Track every LLM request: tokens, cost, latency, full prompt traces, and reasoning token capture. Works with OpenAI, Anthropic, Google Gemini, Groq, Mistral, Azure OpenAI, DeepSeek, Perplexity, Fireworks, Together AI, Cohere, HuggingFace, Replicate, Ollama, and any HTTP endpoint. Self-hosted, no data leaves your machine.

---

## Getting Started

The only requirement is [Docker Desktop](https://www.docker.com/products/docker-desktop/).

### Mac

Open Terminal and run:

```bash
curl -o docker-compose.yml https://raw.githubusercontent.com/torrix-ai/install/main/docker-compose.community.yml
docker compose up
```

> This downloads the community edition config and saves it as `docker-compose.yml` so Docker picks it up automatically.

### Windows

Open PowerShell and run:

```powershell
curl.exe -o docker-compose.yml https://raw.githubusercontent.com/torrix-ai/install/main/docker-compose.community.yml
docker compose up
```

> This downloads the community edition config and saves it as `docker-compose.yml` so Docker picks it up automatically.

Or download the file manually:
1. Go to [github.com/torrix-ai/install](https://github.com/torrix-ai/install)
2. Click `docker-compose.community.yml` then click **Raw**
3. Save the file as `docker-compose.yml`
4. Open a terminal in that folder and run `docker compose up`

### After startup

1. Open [http://localhost:8088](http://localhost:8088)
2. Create your account
3. Copy your API key from Settings
4. Start sending LLM calls through the proxy or SDK

### Verify your setup

Check the server is running (no API key needed):

```bash
curl http://localhost:8088/health
```

Expected response:
```json
{"ok":true,"name":"Torrix","version":"0.1.0"}
```

Check runs are being logged (requires your API key from Settings):

**Mac / Linux:**
```bash
curl http://localhost:8088/api/runs -H "Authorization: Bearer <your-torrix-api-key>"
```

**Windows (PowerShell):**
```powershell
Invoke-WebRequest http://localhost:8088/api/runs -Headers @{Authorization="Bearer <your-torrix-api-key>"} | Select-Object -ExpandProperty Content
```

Returns a list of all logged runs. An empty array `[]` means the server is working but no runs have been sent yet.

### Send a test run

Send a real request through the Torrix proxy to confirm runs appear in the dashboard. Even if the OpenAI key is invalid, Torrix will still log the attempt.

**Mac / Linux:**
```bash
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://api.openai.com/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer <your-openai-key>" \
  -H "x-torrix-name: test-run" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello"}]}'
```

**Windows (PowerShell):**
```powershell
Invoke-WebRequest -Method Post http://localhost:8088/proxy `
  -Headers @{
    "Authorization"="Bearer <your-torrix-api-key>";
    "x-target-url"="https://api.openai.com/v1/chat/completions";
    "x-upstream-authorization"="Bearer <your-openai-key>";
    "x-torrix-name"="test-run"
  } `
  -ContentType "application/json" `
  -Body '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello"}]}' | Select-Object -ExpandProperty Content
```

Then open [http://localhost:8088](http://localhost:8088). The run should appear in your dashboard.

---

## Sending data to Torrix

### Option 1: Python SDK

```bash
pip install torrix
```

**OpenAI:**
```python
import torrix
from openai import OpenAI

torrix.init(api_key="<your-torrix-api-key>", base_url="http://localhost:8088")
client = torrix.wrap(OpenAI(api_key="<your-openai-key>"))

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Hello!"}],
    torrix_name="my-run",
)
print(response.choices[0].message.content)
```

**Anthropic:**
```python
from anthropic import Anthropic

client = torrix.wrap(Anthropic(api_key="<your-anthropic-key>"))

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello!"}],
    torrix_name="my-run",
)
print(response.content[0].text)
```

**Streaming:**
```python
stream = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Hello!"}],
    stream=True,
)
for chunk in stream:
    print(chunk.choices[0].delta.content or "", end="", flush=True)
```

### Option 2: Node.js SDK

```bash
npm install torrix openai
# or: npm install torrix @anthropic-ai/sdk
```

**OpenAI:**
```typescript
import * as torrix from 'torrix'
import OpenAI from 'openai'

torrix.init('<your-torrix-api-key>', 'http://localhost:8088')
const client = torrix.wrap(new OpenAI({ apiKey: '<your-openai-key>' }))

const response = await client.chat.completions.create({
  model: 'gpt-4o-mini',
  messages: [{ role: 'user', content: 'Hello!' }],
  torrix_name: 'my-run',
})
console.log(response.choices[0].message.content)
```

**Anthropic:**
```typescript
import Anthropic from '@anthropic-ai/sdk'

const client = torrix.wrap(new Anthropic({ apiKey: '<your-anthropic-key>' }))

const response = await client.messages.create({
  model: 'claude-3-5-sonnet-20241022',
  max_tokens: 1024,
  messages: [{ role: 'user', content: 'Hello!' }],
  torrix_name: 'my-run',
})
console.log(response.content[0].text)
```

### Option 3: HTTP Proxy (any language or tool)

Route any HTTP request through Torrix. Works with Google Gemini, Azure OpenAI, Groq, Mistral, DeepSeek, Perplexity, Fireworks, Together AI, Cohere, HuggingFace, Replicate, SAP AI Core, GitHub Copilot, n8n, Make, curl, and any OpenAI-compatible API.

```bash
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://api.openai.com/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer <your-openai-key>" \
  -H "x-torrix-name: my-run" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello"}]}'
```

| Header | Description |
|---|---|
| `Authorization` | Your Torrix API key (from Settings) |
| `x-target-url` | The real LLM endpoint to forward to |
| `x-upstream-authorization` | Your LLM provider API key (omit if using `?key=` in URL) |
| `x-torrix-name` | Optional label for this run |
| `x-torrix-provider` | Optional provider hint: `openai`, `anthropic`, `google` |
| `x-torrix-trace` | Optional trace ID to group multiple calls into one agent run |
| `x-torrix-session` | Optional session ID to group a multi-turn conversation |

**Google Gemini** (uses `?key=` instead of Bearer token):
```python
import requests

response = requests.post(
    "http://localhost:8088/proxy",
    headers={
        "Authorization": "Bearer <your-torrix-api-key>",
        "x-target-url": "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=<your-gemini-key>",
        "x-torrix-provider": "google",
        "x-torrix-name": "gemini-test",
    },
    json={"contents": [{"parts": [{"text": "Hello!"}]}]},
)
```

**Azure OpenAI:**
```bash
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://<your-resource>.openai.azure.com/openai/deployments/<your-deployment>/chat/completions?api-version=2024-02-01" \
  -H "x-upstream-authorization: Bearer <your-azure-key>" \
  -H "x-torrix-name: azure-test" \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Hello"}]}'
```

**Groq:**
```bash
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://api.groq.com/openai/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer <your-groq-key>" \
  -H "x-torrix-name: groq-test" \
  -H "Content-Type: application/json" \
  -d '{"model":"llama3-8b-8192","messages":[{"role":"user","content":"Hello"}]}'
```

**Mistral:**
```bash
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://api.mistral.ai/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer <your-mistral-key>" \
  -H "x-torrix-name: mistral-test" \
  -H "Content-Type: application/json" \
  -d '{"model":"mistral-small-latest","messages":[{"role":"user","content":"Hello"}]}'
```

**DeepSeek:**
```bash
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://api.deepseek.com/chat/completions" \
  -H "x-upstream-authorization: Bearer <your-deepseek-key>" \
  -H "x-torrix-name: deepseek-test" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-chat","messages":[{"role":"user","content":"Hello!"}]}'
```

**Ollama (local models):**
```bash
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: http://host.docker.internal:11434/v1/chat/completions" \
  -H "x-torrix-name: ollama-test" \
  -H "Content-Type: application/json" \
  -d '{"model":"llama3.2","messages":[{"role":"user","content":"Hello!"}]}'
```

No API key needed for Ollama: omit `x-upstream-authorization`. Use `host.docker.internal` instead of `localhost` when running Torrix in Docker on Mac or Windows. On Linux, use your machine's actual IP address (e.g. `172.17.0.1`) instead.

**n8n workflow:** Use the HTTP Request node pointed at `http://host.docker.internal:8088/proxy` with these headers:

| Header | Value |
|---|---|
| `Authorization` | `Bearer <your-torrix-api-key>` |
| `x-target-url` | `https://api.openai.com/v1/chat/completions` |
| `x-upstream-authorization` | `Bearer <your-openai-key>` |
| `Content-Type` | `application/json` |

### n8n Community Node

Install the official Torrix node directly in n8n for a native drag-and-drop experience:

1. In n8n, go to **Settings → Community Nodes**
2. Click **Install** and enter `@torrix-ai/n8n-nodes-torrix`
3. Restart n8n when prompted
4. The **Torrix Proxy** node will appear in your node palette

Or import the ready-to-use workflow template:

1. Download [torrix-workflow-template.json](torrix/n8n/torrix-workflow-template.json)
2. In n8n, go to **Workflows → Import from file**
3. Follow the setup notes inside the workflow

---

## Features

### Agent trace grouping

Add `x-torrix-trace` to every call in an agent run to group them into a single chain timeline. Generate one UUID per agent invocation and reuse it across all steps:

```bash
TRACE_ID=$(python3 -c "import uuid; print(uuid.uuid4())")

# Step 1
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://api.openai.com/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer <your-openai-key>" \
  -H "x-torrix-trace: $TRACE_ID" \
  -H "x-torrix-name: classify" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Classify this ticket..."}]}'

# Step 2 - same TRACE_ID links it to step 1
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://api.openai.com/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer <your-openai-key>" \
  -H "x-torrix-trace: $TRACE_ID" \
  -H "x-torrix-name: respond" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Now write a reply..."}]}'
```

Both runs appear in the Runs list with a **trace** badge. Click it to open the chain timeline showing each step with its model, tokens, cost, and latency.

### Conversation session grouping

Add `x-torrix-session` to every call in a multi-turn conversation to group them together. Generate one session ID per conversation and reuse it across all turns:

```bash
SESSION_ID=$(python3 -c "import uuid; print(uuid.uuid4())")

# Turn 1
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://api.openai.com/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer <your-openai-key>" \
  -H "x-torrix-session: $SESSION_ID" \
  -H "x-torrix-name: user-message-1" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello"}]}'

# Turn 2 - same SESSION_ID links it to turn 1
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://api.openai.com/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer <your-openai-key>" \
  -H "x-torrix-session: $SESSION_ID" \
  -H "x-torrix-name: user-message-2" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Follow-up question"}]}'
```

Runs appear with a **session** badge showing the turn count. Click it to see the full conversation with combined cost and tokens.

### Real-time cost tracking
Every API call is logged with token counts, model, cost, and latency. See exactly what you're spending as it happens.

### Regression testing (Evals)
Mark any run as a golden baseline. Replay it against the LLM with one click and compare outputs side-by-side. Catch regressions when switching models or changing prompts.

### Model cost comparison
On any run detail page, see what the same request would have cost across 300+ models, live priced and sorted cheapest to most expensive.

### Budget alerts
Set a daily spend threshold. Torrix fires a webhook when you exceed it. Fires once per day, no noise. Slack webhook URLs (`https://hooks.slack.com/…`) are automatically formatted as native Slack messages with Block Kit.

### Run comparison
Pick any two runs and compare them side-by-side: model, cost, tokens, latency, prompt, and response.

### Run scoring
On any run detail page, click **👍 Good** or **👎 Bad** to score the response. Add an optional note. Scored runs show a green or red badge in the runs list. Use the **Score** filter dropdown to show only good runs, only bad runs, or only unscored runs. Export runs to CSV to download your scored dataset for offline eval pipelines.

### CSV export
Click **Export CSV** on the Runs page to download all currently filtered runs as a CSV file. The file includes run ID, name, provider, model, status, tokens, cost, latency, finish reason, source, score, score note, and timestamp. Apply filters first to export a subset such as only good runs or only a specific model.

### Thinking & reasoning capture
Captures chain-of-thought reasoning from OpenAI o1/o3/o4, DeepSeek R1, Claude extended thinking, Gemini 2.5, and Ollama Qwen3. Reasoning steps appear in the Event Timeline alongside the final response. Reasoning tokens are tracked separately where the model reports them.

---

## Editions

Community is free forever. Pro and Enterprise are coming soon.

| Feature | Community | Pro | Enterprise |
|---|---|---|---|
| Users | 1 | Up to 10 | Unlimited |
| Data retention | 7 days | 30 days | 90 days |
| Runs shown | 100 most recent | Unlimited | Unlimited |
| Budget alerts | ✓ | ✓ | ✓ |
| Evals &amp; regression testing | ✓ | ✓ | ✓ |
| Model cost comparison | ✓ | ✓ | ✓ |
| Run scoring | ✓ | ✓ | ✓ |
| CSV export | ✓ | ✓ | ✓ |
| Prompt version control | No | Coming soon | Coming soon |
| Prompt playground | No | Coming soon | Coming soon |
| Scheduled cost reports | No | Coming soon | Coming soon |
| SSO (SAML / Okta) | No | No | Coming soon |
| PII detection &amp; masking | No | No | Coming soon |
| Audit log export | No | No | Coming soon |
| Helm chart (Kubernetes) | No | No | Coming soon |
| Support | Community | Priority | Dedicated |

Pro and Enterprise are coming soon at [torrix.ai](https://torrix.ai)

---

## Updating Torrix

To pull the latest version:

```bash
docker compose pull
docker compose up -d
```

---

## Stopping Torrix

```bash
docker compose down
```

Your data is preserved in the `./data/` folder and will be available when you start again.

---

## Configuration

| Environment variable | Default | Description |
|---|---|---|
| `DB_PATH` | `/data/torrix.sqlite` | Path to SQLite database inside the container |
| `TORRIX_TELEMETRY` | `true` | Set to `false` to opt out of anonymous usage stats |

To set environment variables, add an `environment` block to your `docker-compose.yml`:

```yaml
services:
  torrix:
    image: torrixai/torrix:latest
    ports:
      - "8088:8088"
    volumes:
      - ./data:/data
    environment:
      - TORRIX_TELEMETRY=false
    restart: unless-stopped
```

---

## Grafana / Prometheus

Torrix exposes a `/metrics` endpoint in Prometheus text format. Scrape it to build Grafana dashboards with your existing monitoring stack.

**Scrape the endpoint:**

```bash
curl http://localhost:8088/metrics -H "Authorization: Bearer <your-torrix-api-key>"
```

**Example output:**
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

**Prometheus `prometheus.yml` scrape config:**

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

Add this to your Prometheus config and create a Grafana dashboard using the `torrix_*` metrics.

---

## Data Privacy

All data stays on your machine. The SQLite database is stored in `./data/` on your host. Torrix never sends your prompts, responses, or API keys anywhere.

Anonymous telemetry is enabled by default. It sends only your instance ID, OS, and Node version to help improve Torrix. To opt out, set `TORRIX_TELEMETRY=false` in your `docker-compose.yml` as shown above.

---

## Support

For questions or feedback: [contact@torrix.ai](mailto:contact@torrix.ai)

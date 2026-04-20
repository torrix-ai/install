# Torrix - AI Observability

Track every LLM request: tokens, cost, latency, and full prompt traces. Works with OpenAI, Anthropic, and more. Self-hosted, no data leaves your machine.

---

## Getting Started

The only requirement is [Docker Desktop](https://www.docker.com/products/docker-desktop/).

### Mac

Open Terminal and run:

```bash
curl -o docker-compose.yml https://raw.githubusercontent.com/torrix-ai/install/main/docker-compose.community.yml
docker compose up
```

### Windows

Open PowerShell and run:

```powershell
curl.exe -o docker-compose.yml https://raw.githubusercontent.com/torrix-ai/install/main/docker-compose.community.yml
docker compose up
```

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

```bash
curl http://localhost:8088/api/runs \
  -H "Authorization: Bearer <your-torrix-api-key>"
```

Returns a list of all logged runs. An empty array `[]` means the server is working but no runs have been sent yet.

---

## Sending data to Torrix

### Option 1 - HTTP Proxy (any language)

Route your existing LLM calls through the Torrix proxy with no code changes to your app logic.

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
| `x-upstream-authorization` | Your LLM provider API key |
| `x-torrix-name` | Optional label for this run |

### Option 2 - Python SDK

```bash
pip install torrix
```

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

Works with Anthropic too:

```python
from anthropic import Anthropic

client = torrix.wrap(Anthropic(api_key="<your-anthropic-key>"))

response = client.messages.create(
    model="claude-3-haiku-20240307",
    max_tokens=64,
    messages=[{"role": "user", "content": "Hello!"}],
    torrix_name="my-run",
)
```

### Option 3 - n8n Workflow

Use the HTTP Request node pointed at `http://host.docker.internal:8088/proxy` with these headers:

| Header | Value |
|---|---|
| `Authorization` | `Bearer <your-torrix-api-key>` |
| `x-target-url` | `https://api.openai.com/v1/chat/completions` |
| `x-upstream-authorization` | `Bearer <your-openai-key>` |
| `Content-Type` | `application/json` |

---

## Community Edition

| Feature | Community | Cloud |
|---|---|---|
| Users | 1 | Coming Soon |
| Data retention | 7 days | Coming Soon |
| Runs shown | 100 | Coming Soon |
| CSV export | No | Coming Soon |
| Support | Community | Coming Soon |

Torrix Cloud is coming soon at [torrix.ai](https://torrix.ai)

---

## Configuration

| Environment variable | Default | Description |
|---|---|---|
| `DB_PATH` | `torrix.sqlite` | Path to SQLite database |
| `TORRIX_TELEMETRY` | `true` | Set to `false` to opt out of anonymous usage stats |

---

## Data Privacy

All data stays on your machine. The SQLite database is stored in `./data/` on your host. Torrix never sends your prompts, responses, or API keys anywhere.

To opt out of anonymous telemetry (instance ID, OS, Node version), set `TORRIX_TELEMETRY=false` in your `docker-compose.yml`.

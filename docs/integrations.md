# Integrations

Send LLM calls to Torrix using the Python SDK, Node.js SDK, or HTTP proxy. The proxy works with any provider that speaks HTTP.

---

## Python SDK

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

---

## Node.js SDK

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

---

## HTTP Proxy

Route any HTTP request through Torrix by pointing it at `/proxy`. Works with Google Gemini, Azure OpenAI, Groq, Mistral, DeepSeek, Perplexity, Fireworks, Together AI, Cohere, HuggingFace, Replicate, SAP AI Core, GitHub Copilot, n8n, Make, curl, and any OpenAI-compatible API.

```bash
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://api.openai.com/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer <your-openai-key>" \
  -H "x-torrix-name: my-run" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello"}]}'
```

**Proxy headers:**

| Header | Description |
|---|---|
| `Authorization` | Your Torrix API key (from Settings) |
| `x-target-url` | The real LLM endpoint to forward to |
| `x-upstream-authorization` | Your LLM provider API key (omit if using `?key=` in URL) |
| `x-torrix-name` | Optional label for this run |
| `x-torrix-provider` | Optional provider hint: `openai`, `anthropic`, `google` |
| `x-torrix-trace` | Optional trace ID to group multiple calls into one agent run |
| `x-torrix-session` | Optional session ID to group a multi-turn conversation |

### Provider examples

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

No `x-upstream-authorization` needed for Ollama. Use `host.docker.internal` instead of `localhost` when running Torrix in Docker on Mac or Windows. On Linux, use your machine's actual IP (e.g. `172.17.0.1`).

---

## n8n

**Option 1: HTTP Request node**

Point the HTTP Request node at `http://host.docker.internal:8088/proxy` with these headers:

| Header | Value |
|---|---|
| `Authorization` | `Bearer <your-torrix-api-key>` |
| `x-target-url` | `https://api.openai.com/v1/chat/completions` |
| `x-upstream-authorization` | `Bearer <your-openai-key>` |
| `Content-Type` | `application/json` |

**Option 2: Torrix Community Node (recommended)**

Install the official Torrix node for a native drag-and-drop experience:

1. In n8n, go to **Settings → Community Nodes**
2. Click **Install** and enter `@torrix-ai/n8n-nodes-torrix`
3. Restart n8n when prompted
4. The **Torrix Proxy** node will appear in your node palette

Or import the ready-to-use workflow template:

1. Download [torrix-workflow-template.json](../torrix/n8n/torrix-workflow-template.json)
2. In n8n, go to **Workflows → Import from file**
3. Follow the setup notes inside the workflow

---

## Send a test run

Confirm the proxy is working end-to-end. Even if your OpenAI key is invalid, Torrix still logs the attempt.

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

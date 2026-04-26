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

Or download the file manually:
1. Go to [github.com/torrix-ai/install](https://github.com/torrix-ai/install)
2. Click `docker-compose.community.yml` then click **Raw**
3. Save the file as `docker-compose.yml`
4. Open a terminal in that folder and run `docker compose up`

### After startup

1. Open [http://localhost:8088](http://localhost:8088)
2. Create your account
3. Copy your API key from Settings
4. Start sending LLM calls through the proxy or SDK. See [Integrations](docs/integrations.md)

### Verify your setup

```bash
curl http://localhost:8088/health
```

Expected response: `{"ok":true,"name":"Torrix","version":"0.1.0"}`

---

## Features

- **Agent trace grouping**: group multi-step agent calls into a single timeline. [How to use](docs/features.md#agent-trace-grouping)
- **Conversation session grouping**: track cost and tokens across a full multi-turn conversation. [How to use](docs/features.md#conversation-session-grouping)
- **Real-time cost tracking**: dollar cost on every call, live in your dashboard
- **Regression testing (Evals)**: mark golden runs, replay against any model, compare outputs side-by-side. [How to use](docs/features.md#regression-testing-evals)
- **Model cost comparison**: see what a run would have cost across 300+ models
- **Budget alerts**: webhook when daily spend exceeds a threshold (Slack supported natively)
- **Run comparison**: side-by-side diff of any two runs: model, cost, tokens, latency, prompt, response
- **Run scoring and LLM judge**: rate responses manually or auto-score with an AI judge that evaluates prompt quality, response correctness, token efficiency, and reasoning depth. Batch-score multiple runs at once. Export as a labelled eval dataset. [How to use](docs/features.md#run-scoring)
- **CSV export**: download filtered runs including score and score note
- **MCP server**: query your data from any MCP-compatible AI assistant. [How to use](docs/mcp.md)
- **Grafana / Prometheus export**: scrape `/metrics` into your existing monitoring stack. [How to use](docs/grafana-prometheus.md)
- **Thinking and reasoning capture**: captures chain-of-thought from OpenAI o1/o3/o4, DeepSeek R1, Claude extended thinking, Gemini 2.5, Ollama Qwen3

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
| MCP server | ✓ | ✓ | ✓ |
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

## Data Privacy

All data stays on your machine. The SQLite database is stored in `./data/` on your host. Torrix never sends your prompts, responses, or API keys anywhere.

Anonymous telemetry is enabled by default. It sends only your instance ID, OS, and Node version to help improve Torrix. To opt out, set `TORRIX_TELEMETRY=false` in your `docker-compose.yml`. See [Configuration](docs/configuration.md).

---

## Documentation

- [Integrations: Python SDK, Node.js SDK, HTTP proxy, n8n](docs/integrations.md)
- [MCP server](docs/mcp.md)
- [Features in depth](docs/features.md)
- [Configuration](docs/configuration.md)
- [Grafana / Prometheus](docs/grafana-prometheus.md)

---

## Support

For questions or feedback: [contact@torrix.ai](mailto:contact@torrix.ai)

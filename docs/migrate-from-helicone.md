# Migrate from Helicone to Torrix

Helicone was acquired by Mintlify in March 2026 and is now in maintenance mode. This guide covers how to switch to Torrix in under 5 minutes.

---

## What stays the same

Torrix uses the same proxy model as Helicone. You point your LLM SDK at a base URL, add a header with your API key, and every call is logged automatically. No code changes to your prompts, models, or messages.

## What changes

| Item | Helicone | Torrix |
|---|---|---|
| Base URL | `https://oai.helicone.ai/v1` | `http://localhost:8088/proxy` |
| Auth header | `Helicone-Auth: Bearer hc-...` | `Authorization: Bearer trxk_...` |
| Upstream key | `Authorization: Bearer sk-...` | `x-upstream-authorization: Bearer sk-...` |
| Target URL | Encoded in the base URL | `x-target-url: https://api.openai.com` |

Your OpenAI (or Anthropic) API key does not change. Only the routing headers change.

---

## Header mapping

If you are using Helicone custom headers, Torrix reads them automatically as aliases. No changes needed in your code.

| Helicone header | Torrix equivalent | What it does |
|---|---|---|
| `helicone-property-name` | `x-torrix-name` | Label the run with a name |
| `helicone-session-id` | `x-torrix-session` | Group calls into a session |
| `helicone-request-id` | `x-torrix-trace` | Group calls into a trace |
| `helicone-user-id` | `x-agent-name` | Tag the run with an agent name |
| `helicone-auth` | (stripped, not forwarded) | Not forwarded upstream |

---

## Before and after

### Before (Helicone)

```bash
curl https://oai.helicone.ai/v1/chat/completions \
  -H "Authorization: Bearer sk-your-openai-key" \
  -H "Helicone-Auth: Bearer hc-your-helicone-key" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello"}]}'
```

### After (Torrix)

```bash
curl http://localhost:8088/proxy \
  -H "Authorization: Bearer trxk_your-torrix-key" \
  -H "x-target-url: https://api.openai.com" \
  -H "x-upstream-authorization: Bearer sk-your-openai-key" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello"}]}'
```

### Python (OpenAI SDK)

```python
# Before (Helicone)
from openai import OpenAI

client = OpenAI(
    api_key="sk-your-openai-key",
    base_url="https://oai.helicone.ai/v1",
    default_headers={
        "Helicone-Auth": "Bearer hc-your-helicone-key",
    }
)

# After (Torrix)
from openai import OpenAI

client = OpenAI(
    api_key="sk-your-openai-key",
    base_url="http://localhost:8088/proxy",
    default_headers={
        "Authorization": "Bearer trxk_your-torrix-key",
        "x-target-url": "https://api.openai.com",
        "x-upstream-authorization": "Bearer sk-your-openai-key",
    }
)
```

---

## Migration checklist

1. **Install Torrix** — run `docker compose up -d` with the community compose file. See the [README](../README.md).
2. **Create your account** — open `http://localhost:8088` and sign up.
3. **Copy your Torrix API key** from Settings.
4. **Update your base URL** from `oai.helicone.ai/v1` to `http://localhost:8088/proxy`.
5. **Update your auth header** from `Helicone-Auth` to `Authorization: Bearer trxk_...`.
6. **Move your OpenAI key** from `Authorization` to `x-upstream-authorization`.
7. **Add `x-target-url`** pointing to `https://api.openai.com` (or your provider).
8. **Send a test request** and confirm it appears in the Runs page.

---

## Anthropic users

Replace `x-target-url` with `https://api.anthropic.com` and set `x-upstream-authorization` to your Anthropic API key. The proxy handles Anthropic response format automatically.

---

## Need help?

Open an issue at [github.com/torrix-ai/install](https://github.com/torrix-ai/install/issues).

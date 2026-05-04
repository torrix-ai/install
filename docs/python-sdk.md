# Python SDK

The Torrix Python SDK automatically traces OpenAI and Anthropic API calls with zero code changes. It captures model, tokens, cost, latency, prompt, and response for every request and sends them to your Torrix instance in a background thread.

## Supported Providers

| Provider | Auto-instrumentation | Explicit wrap | Streaming |
|----------|---------------------|---------------|-----------|
| OpenAI | Yes | Yes | Yes |
| Anthropic | Yes | Yes | Yes |

For other providers (Google Gemini, Cohere, Mistral), use the HTTP proxy instead of the SDK.

## Installation

Copy the `torrix/` package directory into your project:

```
your-project/
  torrix/
    __init__.py
    client.py
    patcher.py
    wrappers/
      openai_wrapper.py
      anthropic_wrapper.py
  app.py
```

No external dependencies required. The SDK uses only Python standard library (`urllib`, `threading`, `json`).

## Quick Start

```python
import torrix
from openai import OpenAI

torrix.init(api_key="trxk_your_key_here", base_url="http://localhost:8088")

client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello!"}]
)
# That's it. The call is traced automatically.
```

## Auto-instrumentation

`torrix.init()` patches the OpenAI and Anthropic client libraries at the class level. Every call made by any instance is traced automatically:

```python
import torrix
torrix.init(api_key="trxk_your_key_here", base_url="http://localhost:8088")

# All OpenAI calls are now traced, regardless of which client instance you use
from openai import OpenAI
client_a = OpenAI()
client_b = OpenAI(api_key="different-key")

# Both are traced
client_a.chat.completions.create(model="gpt-4o", messages=[...])
client_b.chat.completions.create(model="gpt-4o-mini", messages=[...])
```

Auto-instrumentation works by monkey-patching `openai.resources.chat.completions.Completions.create` and `anthropic.resources.messages.Messages.create`. If the library is not installed, that provider is silently skipped.

## Explicit Wrapping

If you prefer not to patch globally, use `torrix.wrap()` to wrap a single client instance:

```python
import torrix
from openai import OpenAI

torrix.init(api_key="trxk_your_key_here", base_url="http://localhost:8088")

raw_client = OpenAI()
client = torrix.wrap(raw_client)

# Only calls through `client` are traced
client.chat.completions.create(model="gpt-4o", messages=[...])
```

Note: if `torrix.init()` already patched the provider, `wrap()` returns the client unchanged to avoid double-tracing.

## OpenAI Examples

### Non-streaming

```python
import torrix
from openai import OpenAI

torrix.init(api_key="trxk_your_key_here")

client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Explain quantum computing in one sentence."}
    ],
    torrix_name="quantum-explainer"  # optional: names this run in the dashboard
)
print(response.choices[0].message.content)
```

### Streaming

```python
import torrix
from openai import OpenAI

torrix.init(api_key="trxk_your_key_here")

client = OpenAI()
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Write a haiku about code."}],
    stream=True,
    stream_options={"include_usage": True}  # recommended: enables token counting
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
# Run is ingested automatically after the stream completes
```

## Anthropic Examples

### Non-streaming

```python
import torrix
from anthropic import Anthropic

torrix.init(api_key="trxk_your_key_here")

client = Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "What is the capital of France?"}],
    torrix_name="geography-qa"
)
print(response.content[0].text)
```

### Streaming

```python
import torrix
from anthropic import Anthropic

torrix.init(api_key="trxk_your_key_here")

client = Anthropic()
stream = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Tell me a joke."}],
    stream=True
)

for event in stream:
    if event.type == "content_block_delta":
        print(event.delta.text, end="")
# Run is ingested after stream finishes
```

## Custom Run Names

Pass `torrix_name` as a keyword argument to label runs in the dashboard:

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Summarize this document..."}],
    torrix_name="doc-summarizer"
)
```

This makes it easy to filter and group runs by function in the Torrix UI (e.g., filter by name = "doc-summarizer").

## Traces and Sessions

When using the HTTP proxy, you can group related calls using headers:

| Header | Purpose |
|--------|---------|
| `x-torrix-trace` | Groups multi-step agent calls into a single trace |
| `x-torrix-session` | Groups conversation turns into a session |

These are proxy-level features. The SDK does not currently send trace/session IDs automatically. If you need trace grouping with the SDK, use the proxy for those calls.

## How It Works

1. `torrix.init()` creates a `TorrixClient` and patches OpenAI/Anthropic at the class level
2. Each intercepted call measures latency, extracts tokens/cost/prompt/response
3. The result is sent to `/api/ingest` on your Torrix server via a background daemon thread
4. The background thread uses `urllib.request` (no external HTTP library needed)
5. On process exit, `atexit` flushes any pending ingest calls (5s timeout per thread)
6. Errors in the tracing layer are silently swallowed and never crash your application

The SDK never blocks your application. If the Torrix server is unreachable, ingest calls fail silently.

## When to Use the Proxy Instead

Use the HTTP proxy (`/proxy` endpoint) when:

- Your provider does not have an SDK wrapper (Gemini, Cohere, Mistral, custom endpoints)
- You want trace/session grouping via headers
- You prefer not to add any code to your application (just change the base URL)

Proxy example with Gemini:

```bash
curl http://localhost:8088/proxy \
  -H "Authorization: Bearer trxk_your_key" \
  -H "x-target-url: https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent?key=YOUR_GEMINI_KEY" \
  -H "Content-Type: application/json" \
  -d '{"contents":[{"parts":[{"text":"Hello"}]}]}'
```

## API Reference

| Function | Parameters | Description |
|----------|-----------|-------------|
| `torrix.init(api_key, base_url)` | `api_key`: your Torrix API key. `base_url`: server URL (default `http://localhost:8088`) | Initializes the SDK and patches OpenAI/Anthropic globally |
| `torrix.wrap(client)` | `client`: an OpenAI or Anthropic client instance | Returns a wrapped client that traces calls. No-op if `init()` already patched the provider |
| `torrix.get_client()` | None | Returns the internal `TorrixClient` instance (for advanced use) |

## Troubleshooting

**Calls not appearing in Torrix dashboard:**
- Verify your Torrix server is running at the `base_url` you specified
- Check that your API key is valid
- Ensure `torrix.init()` is called before creating any OpenAI/Anthropic client instances
- For streaming calls, ensure you fully consume the iterator (partial iteration still traces on garbage collection)

**Double-tracing:**
- If you see duplicate entries, you may be using both `init()` (auto-patch) and `wrap()` on the same client
- `wrap()` returns the client unchanged when auto-instrumentation is active, so this should not happen in normal usage

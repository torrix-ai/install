# Node.js SDK

The Torrix Node.js SDK automatically traces OpenAI and Anthropic API calls with zero code changes. It captures model, tokens, cost, latency, prompt, and response for every request and sends them to your Torrix instance in a background `setImmediate` call.

## Supported Providers

| Provider | Auto-instrumentation | Explicit wrap | Streaming |
|----------|---------------------|---------------|-----------|
| OpenAI | Yes | Yes | Yes |
| Anthropic | Yes | Yes | Yes |

For other providers (Google Gemini, Cohere, Mistral), use the HTTP proxy instead.

## Installation

Copy the `torrix/` SDK directory into your project:

```
your-project/
  torrix/
    dist/          (compiled output)
    src/
      index.ts
      client.ts
      patcher.ts
      wrappers/
        openai.ts
        anthropic.ts
  app.ts
```

No external dependencies required. The SDK uses only Node.js built-ins (`http`, `https`, `crypto`).

## Quick Start

```typescript
import * as torrix from './torrix';
import OpenAI from 'openai';

torrix.init('trxk_your_key_here', 'http://localhost:8088');

const client = new OpenAI();
const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Hello!' }],
});
// That's it. The call is traced automatically.
```

## Auto-instrumentation

`torrix.init()` patches the OpenAI and Anthropic client libraries at the class prototype level. Every call made by any instance is traced automatically:

```typescript
import * as torrix from './torrix';
torrix.init('trxk_your_key_here', 'http://localhost:8088');

// All OpenAI calls are now traced, regardless of which client instance you use
import OpenAI from 'openai';
const clientA = new OpenAI();
const clientB = new OpenAI({ apiKey: 'different-key' });

// Both are traced
await clientA.chat.completions.create({ model: 'gpt-4o', messages: [...] });
await clientB.chat.completions.create({ model: 'gpt-4o-mini', messages: [...] });
```

Auto-instrumentation works by monkey-patching `Completions.prototype.create` (from `openai/resources/chat/completions`) and `Messages.prototype.create` (from `@anthropic-ai/sdk/resources/messages`). If a library is not installed, that provider is silently skipped.

## Explicit Wrapping

If you prefer not to patch globally, use `torrix.wrap()` to wrap a single client instance:

```typescript
import * as torrix from './torrix';
import OpenAI from 'openai';

torrix.init('trxk_your_key_here', 'http://localhost:8088');

const raw = new OpenAI();
const client = torrix.wrap(raw);

// Only calls through `client` are traced
await client.chat.completions.create({ model: 'gpt-4o', messages: [...] });
```

Note: if `torrix.init()` already patched the provider, `wrap()` returns the client unchanged to avoid double-tracing.

## OpenAI Examples

### Non-streaming

```typescript
import * as torrix from './torrix';
import OpenAI from 'openai';

torrix.init('trxk_your_key_here');

const client = new OpenAI();
const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [
    { role: 'system', content: 'You are a helpful assistant.' },
    { role: 'user', content: 'Explain quantum computing in one sentence.' },
  ],
  torrix_name: 'quantum-explainer', // optional: names this run in the dashboard
} as any);
console.log(response.choices[0].message.content);
```

### Streaming

```typescript
import * as torrix from './torrix';
import OpenAI from 'openai';

torrix.init('trxk_your_key_here');

const client = new OpenAI();
const stream = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Write a haiku about code.' }],
  stream: true,
  stream_options: { include_usage: true }, // recommended: enables token counting
} as any);

for await (const chunk of stream) {
  process.stdout.write(chunk.choices[0]?.delta?.content ?? '');
}
// Run is ingested automatically after the stream completes
```

## Anthropic Examples

### Non-streaming

```typescript
import * as torrix from './torrix';
import Anthropic from '@anthropic-ai/sdk';

torrix.init('trxk_your_key_here');

const client = new Anthropic();
const response = await client.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 1024,
  messages: [{ role: 'user', content: 'What is the capital of France?' }],
  torrix_name: 'geography-qa',
} as any);
console.log(response.content[0].type === 'text' ? response.content[0].text : '');
```

### Streaming

```typescript
import * as torrix from './torrix';
import Anthropic from '@anthropic-ai/sdk';

torrix.init('trxk_your_key_here');

const client = new Anthropic();
const stream = await client.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 1024,
  messages: [{ role: 'user', content: 'Tell me a joke.' }],
  stream: true,
} as any);

for await (const event of stream) {
  if (event.type === 'content_block_delta' && event.delta.type === 'text_delta') {
    process.stdout.write(event.delta.text);
  }
}
// Run is ingested after stream finishes
```

## Custom Run Names

Pass `torrix_name` in the params object to label runs in the dashboard:

```typescript
await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Summarize this document...' }],
  torrix_name: 'doc-summarizer',
} as any);
```

Cast params `as any` to suppress TypeScript strict type checking on the extra field. The SDK strips `torrix_name` before forwarding to the provider.

## How It Works

1. `torrix.init()` creates a `TorrixClient` and patches OpenAI/Anthropic at the class prototype level
2. Each intercepted call measures latency and extracts tokens, cost, prompt, and response
3. The result is sent to `/api/ingest` on your Torrix server via `setImmediate` (fire-and-forget)
4. The background call uses Node.js built-in `http`/`https` with no external dependencies
5. Errors in the tracing layer are silently swallowed and never crash your application

The SDK never blocks your application. If the Torrix server is unreachable, ingest calls fail silently.

## Limitations

- CommonJS only. The patching mechanism uses `require()`, so ESM-only projects should use the HTTP proxy or the explicit `wrap()` approach instead.
- Streaming wrappers expose the `AsyncIterable` interface only. Methods like `.toReadableStream()` on the OpenAI stream object are not available when using auto-instrumentation.

## API Reference

| Function | Parameters | Description |
|----------|-----------|-------------|
| `torrix.init(apiKey, baseUrl)` | `apiKey`: your Torrix API key. `baseUrl`: server URL (default `http://localhost:8088`) | Initializes the SDK and patches OpenAI/Anthropic globally |
| `torrix.wrap(client)` | `client`: an OpenAI or Anthropic client instance | Returns a wrapped client that traces calls. No-op if `init()` already patched the provider |
| `torrix.getClient()` | None | Returns the internal `TorrixClient` instance |

## Troubleshooting

**Calls not appearing in Torrix dashboard:**
- Verify your Torrix server is running at the `base_url` you specified
- Check that your API key is valid
- Ensure `torrix.init()` is called before creating any OpenAI/Anthropic client instances
- For streaming calls, ensure you fully consume the iterator

**TypeScript error on `torrix_name`:**
- Cast your params object `as any`, or add `torrix_name?: string` to your own type extension

**Double-tracing:**
- If you see duplicate entries, you may be using both `init()` and `wrap()` on the same client
- `wrap()` returns the client unchanged when auto-instrumentation is already active, so this should not happen in normal usage

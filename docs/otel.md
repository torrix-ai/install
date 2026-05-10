# OpenTelemetry Setup

Torrix accepts OTLP/HTTP (JSON) traces at `POST /v1/traces`. Any application already instrumented
with an OpenTelemetry GenAI library can point its OTLP exporter at Torrix and get full
observability with zero Torrix SDK changes.

Torrix reads `gen_ai.*` span attributes and span events, then computes cost automatically.

## OTLP endpoint

```
http://your-torrix-host:8088/v1/traces
```

Pass your Torrix API key as a header:

```
Authorization: Bearer trxk_...
```

or via the `OTEL_EXPORTER_OTLP_HEADERS` environment variable:

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:8088
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer trxk_your_key_here"
```

---

## Python: opentelemetry-instrumentation-openai

```bash
pip install opentelemetry-instrumentation-openai opentelemetry-exporter-otlp-proto-http
```

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.openai import OpenAIInstrumentor

exporter = OTLPSpanExporter(
    endpoint="http://localhost:8088/v1/traces",
    headers={"Authorization": "Bearer trxk_your_key_here"},
)
provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)

OpenAIInstrumentor().instrument()

# All subsequent OpenAI calls are automatically traced
from openai import OpenAI
client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "What is Paris?"}],
)
```

---

## Python: opentelemetry-instrumentation-anthropic

```bash
pip install opentelemetry-instrumentation-anthropic opentelemetry-exporter-otlp-proto-http
```

```python
from opentelemetry.instrumentation.anthropic import AnthropicInstrumentor

# Same OTel setup as above, then:
AnthropicInstrumentor().instrument()

import anthropic
client = anthropic.Anthropic()
message = client.messages.create(
    model="claude-3-5-sonnet-latest",
    max_tokens=256,
    messages=[{"role": "user", "content": "What is Paris?"}],
)
```

---

## Java: Spring AI with OTel

Spring AI emits `gen_ai.*` spans automatically when OTel tracing is on the classpath.

```xml
<!-- pom.xml -->
<dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
```

```yaml
# application.yaml
management:
  tracing:
    enabled: true
    sampling:
      probability: 1.0

otel:
  exporter:
    otlp:
      endpoint: http://localhost:8088
      headers:
        Authorization: "Bearer trxk_your_key_here"
```

All Spring AI `ChatClient` calls are traced automatically.

---

## Node.js: @arizeai/openinference-instrumentation-openai

```bash
npm install @arizeai/openinference-instrumentation-openai \
  @opentelemetry/sdk-node \
  @opentelemetry/exporter-trace-otlp-http
```

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { OpenAIInstrumentation } from '@arizeai/openinference-instrumentation-openai';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://localhost:8088/v1/traces',
    headers: { Authorization: 'Bearer trxk_your_key_here' },
  }),
  instrumentations: [new OpenAIInstrumentation()],
});
sdk.start();

// All subsequent openai calls are traced
import OpenAI from 'openai';
const client = new OpenAI();
const response = await client.chat.completions.create({
  model: 'gpt-4o-mini',
  messages: [{ role: 'user', content: 'What is Paris?' }],
});
```

---

## Attribute mapping

Torrix extracts the following from each span:

| OTel attribute / span field | Torrix field |
|---|---|
| `gen_ai.system` | provider |
| `gen_ai.request.model` | model |
| `gen_ai.usage.input_tokens` | input_tokens |
| `gen_ai.usage.output_tokens` | output_tokens |
| `gen_ai.usage.cache_read_input_tokens` | cache_read_tokens |
| `gen_ai.usage.cache_creation_input_tokens` | cache_creation_tokens |
| `gen_ai.response.finish_reasons` (array) | finish_reason |
| `gen_ai.operation.name` or span name | run name |
| `gen_ai.agent.name` or `peer.service` | agent_name |
| `session.id` or `gen_ai.conversation.id` | session_id |
| `torrix.project_id` or `x-torrix-project-id` header | project_id |
| span `traceId` | trace_id |
| span duration (ns) | latency_ms |
| span status code 2 (ERROR) | status 500 |
| span event `gen_ai.content.prompt` | prompt text |
| span event `gen_ai.content.completion` | response text |
| input + output tokens | cost_usd (auto-computed) |

Both the new spec (`gen_ai.content.prompt` event with `gen_ai.prompt` attribute) and the older
spec (`gen_ai.prompt` event with `gen_ai.prompt.0.content` attribute) are supported.

---

## Routing to a project

Pass `x-torrix-project-id` as a header, or set `torrix.project_id` as a span attribute:

```bash
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer trxk_...,x-torrix-project-id=my-project"
```

---

## Troubleshooting

**Runs appear with source "sdk" instead of "otel"**
Torrix v2.2.0+ labels OTLP runs as `otel`. Earlier versions used `sdk`. Update to the latest image.

**Cost shows as empty**
Torrix looks up cost by model name from `gen_ai.request.model`. If the model name does not match
a known pricing entry, cost will be blank. Use the custom model pricing page in Settings to add it.

**Prompt and response are empty**
The instrumentation library must emit `gen_ai.content.prompt` and `gen_ai.content.completion`
span events. Some libraries disable this by default to avoid logging sensitive data. Check the
library's docs for a `capture_content` or similar option.

**No spans arriving**
Confirm the OTLP endpoint is reachable and the API key is valid:

```bash
curl -X POST http://localhost:8088/v1/traces \
  -H "Authorization: Bearer trxk_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans":[{"scopeSpans":[{"spans":[{
    "spanId":"abc123","traceId":"trace001","name":"chat openai",
    "startTimeUnixNano":"1700000000000000000",
    "endTimeUnixNano":"1700000001500000000",
    "status":{"code":0},
    "attributes":[
      {"key":"gen_ai.system","value":{"stringValue":"openai"}},
      {"key":"gen_ai.request.model","value":{"stringValue":"gpt-4o-mini"}},
      {"key":"gen_ai.usage.input_tokens","value":{"intValue":"10"}},
      {"key":"gen_ai.usage.output_tokens","value":{"intValue":"5"}}
    ]
  }]}]}]}'
```

A `{"received":true,"spans":1}` response confirms delivery.

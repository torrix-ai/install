# OpenTelemetry Receiver

Torrix accepts OTLP/HTTP (JSON) traces from any OpenTelemetry SDK at `POST /v1/traces`.

This means any application already instrumented with the OpenTelemetry SDK can send LLM spans to Torrix without additional code changes, as long as the spans include `gen_ai.*` semantic attributes.

## Endpoint

```
POST http://localhost:8088/v1/traces
Content-Type: application/json
x-torrix-api-key: trxk_...
```

The API key is the same key used for the Torrix dashboard. You can also pass it as `Authorization: Bearer trxk_...`.

## Attribute mapping

Torrix maps the following OpenTelemetry semantic attributes:

| OTel attribute | Torrix field |
|---|---|
| `gen_ai.system` | provider |
| `gen_ai.request.model` | model |
| `gen_ai.usage.input_tokens` | input_tokens |
| `gen_ai.usage.output_tokens` | output_tokens |
| `gen_ai.operation.name` | run name |
| `gen_ai.prompt` | prompt text |
| `gen_ai.completion` | response text |
| `session.id` | session_id |
| span `traceId` | trace_id |
| span duration (ns) | latency_ms |
| span status code 2 (ERROR) | status 500 |

## Python example

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

exporter = OTLPSpanExporter(
    endpoint="http://localhost:8088/v1/traces",
    headers={"x-torrix-api-key": "trxk_your_key_here"},
)

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("my-llm-app")

with tracer.start_as_current_span("chat") as span:
    span.set_attribute("gen_ai.system", "openai")
    span.set_attribute("gen_ai.request.model", "gpt-4o-mini")
    span.set_attribute("gen_ai.usage.input_tokens", 512)
    span.set_attribute("gen_ai.usage.output_tokens", 128)
    # ... your LLM call here
```

## Notes

- Cost is not computed from OTEL spans. Torrix does not have the full request/response body, so it cannot apply pricing table lookups. The cost field will be empty.
- For full cost and prompt/response capture, use the Torrix HTTP proxy or Python SDK instead.
- The OTEL receiver is useful for applications that already emit spans and need to route them into Torrix without code changes.

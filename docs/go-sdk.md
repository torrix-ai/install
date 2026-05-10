# Go SDK

The Torrix Go SDK provides a lightweight, zero-dependency client for sending run telemetry to your Torrix instance. Unlike the Python and Node.js SDKs, it does not auto-patch LLM clients (Go has no dominant LLM client convention). Instead you time your calls with `Measure` and ingest the result with `Ingest`.

## Supported providers

Any provider you call manually. Pass the result to `Ingest` with the fields you have.

| Provider | Method |
|----------|--------|
| OpenAI (sashabaranov/go-openai) | Manual: Measure + Ingest |
| Anthropic | Manual: Measure + Ingest |
| Any HTTP-based LLM | Manual: Measure + Ingest |

For zero-code instrumentation of any provider, use the HTTP proxy instead.

## Installation

```bash
go get torrix.ai/sdk/go
```

The module is `torrix.ai/sdk/go`. No external dependencies beyond the Go standard library.

## Quick start

```go
package main

import (
    "context"
    "os"

    torrix "torrix.ai/sdk/go"
    openai "github.com/sashabaranov/go-openai"
)

func ptr[T any](v T) *T { return &v }

func main() {
    torrix.Init(os.Getenv("TORRIX_API_KEY"))

    client := openai.NewClient(os.Getenv("OPENAI_API_KEY"))
    userMsg := "What is the capital of France?"

    var resp openai.ChatCompletionResponse
    latency, err := torrix.Measure(func() error {
        var e error
        resp, e = client.CreateChatCompletion(context.Background(), openai.ChatCompletionRequest{
            Model:    openai.GPT4oMini,
            Messages: []openai.ChatCompletionMessage{{Role: openai.ChatMessageRoleUser, Content: userMsg}},
        })
        return e
    })
    if err != nil {
        // still ingest the error
        errStr := err.Error()
        torrix.Ingest(torrix.IngestPayload{
            Model:     ptr(string(openai.GPT4oMini)),
            LatencyMs: ptr(latency.Milliseconds()),
            Status:    ptr(500),
            Prompt:    &userMsg,
            Error:     &errStr,
        })
        panic(err)
    }

    reply := resp.Choices[0].Message.Content
    finish := string(resp.Choices[0].FinishReason)

    torrix.Ingest(torrix.IngestPayload{
        Model:        &resp.Model,
        InputTokens:  ptr(int(resp.Usage.PromptTokens)),
        OutputTokens: ptr(int(resp.Usage.CompletionTokens)),
        LatencyMs:    ptr(latency.Milliseconds()),
        Status:       ptr(200),
        FinishReason: &finish,
        Prompt:       &userMsg,
        Response:     &reply,
    })
}
```

## Init

```go
torrix.Init(apiKey string, opts ...Option)
```

Call once at program startup. Initialises the global client singleton.

```go
torrix.Init(os.Getenv("TORRIX_API_KEY"))

// Custom base URL (default: http://localhost:8088)
torrix.Init(os.Getenv("TORRIX_API_KEY"), torrix.WithBaseURL("https://torrix.your-domain.com"))
```

## Options

| Option | Description |
|--------|-------------|
| `WithBaseURL(url string)` | Sets the Torrix server URL. Trailing slash is stripped automatically. |

## Ingest

```go
torrix.Ingest(payload torrix.IngestPayload)
```

Sends a run to Torrix in a background goroutine. Never blocks. Never panics. Failures are silently discarded.

`Source` is set to `"sdk"` automatically if not provided. `Prompt` and `Response` are truncated to 800 characters.

## Measure

```go
latency, err := torrix.Measure(func() error {
    // your LLM call here
    return err
})
```

Times the provided function and returns the elapsed `time.Duration` alongside any error. Use `latency.Milliseconds()` for `IngestPayload.LatencyMs`.

## IngestPayload reference

All fields are optional. The server calculates cost from tokens and model if `CostUSD` is not provided.

| Field | Type | Description |
|-------|------|-------------|
| `RunID` | `*string` | Unique ID. Generated server-side if omitted. |
| `Name` | `*string` | Display name. Auto-generated from prompt if omitted. |
| `ProjectID` | `*string` | Project namespace. |
| `Provider` | `*string` | Provider name, e.g. `"openai"`, `"anthropic"`. |
| `Model` | `*string` | Model name, e.g. `"gpt-4o-mini"`. |
| `InputTokens` | `*int` | Prompt token count. |
| `OutputTokens` | `*int` | Completion token count. |
| `ThinkingTokens` | `*int` | Reasoning token count (o1/o3 models). |
| `CostUSD` | `*float64` | Cost in USD. Calculated from tokens if omitted. |
| `LatencyMs` | `*int64` | End-to-end latency in milliseconds. |
| `Status` | `*int` | HTTP status code (200, 401, 429, 500, etc.). |
| `FinishReason` | `*string` | Why the model stopped: `"stop"`, `"length"`, etc. |
| `Prompt` | `*string` | The user prompt. Truncated to 800 chars. |
| `Response` | `*string` | The model response. Truncated to 800 chars. |
| `Error` | `*string` | Short error message. |
| `ErrorDetail` | `*string` | Full error detail or stack trace. |
| `Thinking` | `*string` | Chain-of-thought content. |
| `Summary` | `*string` | Human-readable summary of the run. |
| `TraceID` | `*string` | Groups related runs into a distributed trace. |
| `SessionID` | `*string` | Groups runs into a conversation session. |
| `AgentName` | `*string` | Agent identifier for agent-cost analytics. |
| `Tags` | `[]string` | Freeform labels, e.g. `[]string{"prod", "v2"}`. |
| `Source` | `*string` | Defaults to `"sdk"`. Set to `"proxy"` if forwarding. |

## How it works

1. `Init` creates a package-level `*torrixClient` singleton.
2. `Ingest` marshals the payload to JSON and launches a goroutine.
3. The goroutine POSTs to `{baseURL}/api/ingest` with a 5-second timeout.
4. Any HTTP or network error is silently discarded. Your application is never affected.
5. `Measure` uses `time.Now()` and `time.Since()` with no additional overhead.

## Troubleshooting

**Runs are not appearing in Torrix**

- Confirm `Init` was called with the correct API key before `Ingest`.
- Confirm the base URL matches your Torrix instance (default: `http://localhost:8088`).
- The SDK never logs errors. Add a temporary `fmt.Println` inside `send()` to debug.

**Program exits before ingest completes**

The goroutine is fire-and-forget. If your program exits immediately after `Ingest`, the goroutine may not finish. Add a short `time.Sleep(200 * time.Millisecond)` after `Ingest` in short-lived CLI programs, or use a `sync.WaitGroup` if you need guaranteed delivery.

**Using a custom LLM client**

Any function that returns an error works with `Measure`:

```go
latency, err := torrix.Measure(func() error {
    result, err = myCustomLLMClient.Complete(prompt)
    return err
})
```

Then pass whatever fields you have to `Ingest`. Only `Model` and token counts are needed for cost calculation.

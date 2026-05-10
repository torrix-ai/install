# C# / .NET SDK

The Torrix .NET SDK is a lightweight, zero-dependency client for sending LLM run telemetry to your Torrix instance. It targets .NET 6 and above and uses only the standard library (`System.Net.Http`, `System.Text.Json`, `System.Diagnostics`).

## Installation

```bash
dotnet add package Torrix
```

## Quick start

```csharp
using TorrixAI;

// Call Init once at application startup.
Torrix.Init("<your-torrix-api-key>", new TorrixOptions
{
    BaseUrl = "http://localhost:8088"
});
```

## Supported providers

| Provider | Notes |
|---|---|
| OpenAI (openai-dotnet) | Use `ChatClient.CompleteChatAsync` |
| Azure OpenAI (Azure.AI.OpenAI) | Use `AzureOpenAIClient.GetChatClient` |
| SAP AI Core | Use the HTTP client integration below |
| Any .NET HTTP client | Use `MeasureAsync` + `Ingest` |

## OpenAI example

```csharp
using OpenAI.Chat;
using TorrixAI;

Torrix.Init("<your-torrix-api-key>");

var chatClient = new ChatClient("gpt-4o-mini", "<your-openai-key>");
var userMessage = "What is the capital of France?";

var (response, latencyMs) = await Torrix.MeasureAsync(async () =>
    await chatClient.CompleteChatAsync(userMessage));

Torrix.Ingest(new IngestPayload
{
    Model        = "gpt-4o-mini",
    Provider     = "openai",
    LatencyMs    = latencyMs,
    InputTokens  = response.Value.Usage.InputTokenCount,
    OutputTokens = response.Value.Usage.OutputTokenCount,
    Prompt       = userMessage,
    Response     = response.Value.Content[0].Text,
});
```

## Azure OpenAI example

```csharp
using Azure.AI.OpenAI;
using OpenAI.Chat;
using TorrixAI;

Torrix.Init("<your-torrix-api-key>");

var azureClient = new AzureOpenAIClient(
    new Uri("https://YOUR_RESOURCE.openai.azure.com/"),
    new Azure.AzureKeyCredential("<your-azure-key>"));

var chatClient  = azureClient.GetChatClient("gpt-4o-mini");
var userMessage = "Write a short deployment checklist.";

var (response, latencyMs) = await Torrix.MeasureAsync(async () =>
    await chatClient.CompleteChatAsync(userMessage));

Torrix.Ingest(new IngestPayload
{
    Model        = "gpt-4o-mini",
    Provider     = "azure",
    LatencyMs    = latencyMs,
    InputTokens  = response.Value.Usage.InputTokenCount,
    OutputTokens = response.Value.Usage.OutputTokenCount,
    Prompt       = userMessage,
    Response     = response.Value.Content[0].Text,
    Tags         = new[] { "deployment", "checklist" },
});
```

## SAP AI Core example

SAP AI Core does not have a .NET SDK. Use `HttpClient` directly and point it at your Torrix server with the `x-target-url` header.

```csharp
using System.Net.Http.Json;
using TorrixAI;

Torrix.Init("<your-torrix-api-key>");

var http = new HttpClient();
http.DefaultRequestHeaders.Add("Authorization",  "Bearer <your-torrix-api-key>");
http.DefaultRequestHeaders.Add("x-target-url",  "https://api.ai.prod.eu-central-1.aws.ml.hana.ondemand.com/v2/inference/deployments/<YOUR_ID>/chat/completions");
http.DefaultRequestHeaders.Add("x-upstream-authorization", "Bearer <SAP_TOKEN>");
http.DefaultRequestHeaders.Add("x-torrix-provider", "sap");

var body = JsonContent.Create(new
{
    model = "gpt-4o",
    messages = new[] { new { role = "user", content = "Hello" } }
});

var response = await http.PostAsync("http://localhost:8088/proxy", body);
```

Torrix captures the call automatically when routed through the proxy endpoint.

## Synchronous timing

Use `Measure` for synchronous operations:

```csharp
var (result, latencyMs) = Torrix.Measure(() => SomeBlockingCall());
```

## IngestPayload reference

All fields are optional except those needed by your dashboard.

| Field | Type | Description |
|---|---|---|
| `Model` | `string` | Model name (e.g. `gpt-4o-mini`) |
| `Provider` | `string` | Provider hint: `openai`, `azure`, `anthropic`, `sap`, `google` |
| `LatencyMs` | `long` | Call duration in milliseconds |
| `InputTokens` | `int` | Prompt token count |
| `OutputTokens` | `int` | Completion token count |
| `Prompt` | `string` | User message or full prompt text |
| `Response` | `string` | Model response text |
| `Name` | `string` | Optional label for this run in the dashboard |
| `TraceId` | `string` | Groups multiple calls into one agent trace |
| `SessionId` | `string` | Groups turns in a multi-turn conversation |
| `AgentName` | `string` | Name of the agent making the call |
| `Tags` | `string[]` | Key-value tags e.g. `new[] { "env=prod", "team=backend" }` |
| `ProjectId` | `string` | Scopes the run to a named project |
| `Status` | `int` | HTTP status code from the upstream provider |
| `Error` | `string` | Short error message if the call failed |
| `ErrorDetail` | `string` | Full error detail or stack trace |

## Notes

- `Ingest` is fire-and-forget. It never blocks and never throws. If the Torrix server is unreachable, the call is silently dropped.
- Prompt and response text is truncated at 800 characters before sending.
- The SDK uses a single shared `HttpClient` instance with a 5-second timeout.

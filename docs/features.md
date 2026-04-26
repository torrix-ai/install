# Features

---

## Agent trace grouping

Add `x-torrix-trace` to every call in an agent run to group them into a single chain timeline. Generate one UUID per agent invocation and reuse it across all steps.

```bash
TRACE_ID=$(python3 -c "import uuid; print(uuid.uuid4())")

# Step 1
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://api.openai.com/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer <your-openai-key>" \
  -H "x-torrix-trace: $TRACE_ID" \
  -H "x-torrix-name: classify" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Classify this ticket..."}]}'

# Step 2 - same TRACE_ID links it to step 1
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://api.openai.com/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer <your-openai-key>" \
  -H "x-torrix-trace: $TRACE_ID" \
  -H "x-torrix-name: respond" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Now write a reply..."}]}'
```

Both runs appear in the Runs list with a **trace** badge. Click it to open the chain timeline showing each step with its model, tokens, cost, and latency.

---

## Conversation session grouping

Add `x-torrix-session` to every call in a multi-turn conversation to group them together. Generate one session ID per conversation and reuse it across all turns.

```bash
SESSION_ID=$(python3 -c "import uuid; print(uuid.uuid4())")

# Turn 1
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://api.openai.com/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer <your-openai-key>" \
  -H "x-torrix-session: $SESSION_ID" \
  -H "x-torrix-name: user-message-1" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello"}]}'

# Turn 2 - same SESSION_ID links it to turn 1
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <your-torrix-api-key>" \
  -H "x-target-url: https://api.openai.com/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer <your-openai-key>" \
  -H "x-torrix-session: $SESSION_ID" \
  -H "x-torrix-name: user-message-2" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Follow-up question"}]}'
```

Runs appear with a **session** badge showing the turn count. Click it to see the full conversation with combined cost and tokens.

---

## Real-time cost tracking

Every API call is logged with token counts, model, cost, and latency. See exactly what you are spending as it happens. Costs are calculated from public pricing for 2500+ models and updated on each server start.

---

## Regression testing (Evals)

Mark any run as a **golden baseline** from the run detail page. On the Evals page, replay it against the LLM with one click and compare outputs side-by-side. Catch regressions when switching models or changing prompts.

Community edition supports up to 10 golden runs. Pro supports up to 100.

---

## Model cost comparison

On any run detail page, click **Compare cost** to see what the same request (same token counts) would have cost across 300+ models, sorted cheapest to most expensive. Useful for deciding whether a cheaper model can replace your current one.

---

## Budget alerts

Set a daily spend threshold in Settings. Torrix fires a webhook when you exceed it. Fires once per day. Slack webhook URLs (`https://hooks.slack.com/...`) are automatically formatted as native Slack Block Kit messages.

---

## Run comparison

Pick any two runs from the Evals page or the runs list and compare them side-by-side: model, cost, tokens, latency, prompt, and response. Useful for A/B testing prompts or models on the same input.

---

## Run scoring and LLM judge

Rate any response as good or bad from the run detail page using the thumbs up / thumbs down buttons. Add an optional note. Scored runs show a green or red badge in the runs list.

### Manual scoring

Click the thumbs up or thumbs down button on any run detail page. Add an optional note to record why.

### Auto-score with an AI judge

Click **Auto-score with AI** on any run detail page to let an LLM evaluate the response automatically.

1. Select the provider: **OpenAI-compatible** (OpenAI, Groq, Mistral, DeepSeek, Ollama, and any provider using the `/v1/chat/completions` format) or **Anthropic**
2. Paste your API key for that provider
3. For OpenAI-compatible providers, optionally enter a custom base URL (defaults to `https://api.openai.com`). Examples:
   - Groq: `https://api.groq.com/openai`
   - Ollama: `http://localhost:11434`
   - Mistral: `https://api.mistral.ai`
4. Optionally enter a custom model (defaults to `gpt-4o-mini` for OpenAI-compatible, `claude-haiku-4-5-20251001` for Anthropic)
5. Optionally enter custom evaluation criteria
6. Click **Auto-score**

The judge sends the run's prompt and response to the selected LLM, which returns a good or bad verdict with a one-sentence reason. The result is saved as the run's score and note, the same as a manual score.

Default criteria when none is specified: "Evaluate correctness, helpfulness, and clarity."

Custom criteria example: "The response should be concise and under 100 words. Penalise any hallucinated facts."

**Note:** Your API key is sent directly to OpenAI or Anthropic for the judge call. It is not stored in Torrix.

### Filtering and export

**Filtering:** Use the **Score** dropdown on the Runs page to show only good runs, only bad runs, or only unscored runs.

**Evals page:** Shows a summary of all scored runs with counts and a table linking back to each one. The Run B compare panel can be filtered to show only good runs.

**Export:** Export to CSV to get your scored dataset for offline eval pipelines. The CSV includes `score` and `score_note` columns.

---

## CSV export

Click **Export CSV** on the Runs page to download all currently filtered runs. The file includes: run ID, name, provider, model, status, input tokens, output tokens, cost, latency, finish reason, source, score, score note, and timestamp.

Apply filters first to export a specific subset — for example, only good runs from a specific model.

---

## Thinking and reasoning capture

Captures chain-of-thought reasoning tokens from models that expose them:

- OpenAI o1, o3, o4 series
- DeepSeek R1
- Claude extended thinking
- Gemini 2.5
- Ollama Qwen3

Reasoning steps appear in the Event Timeline on the run detail page alongside the final response. Reasoning tokens are tracked separately where the model reports them, so you can see the true cost of extended thinking.

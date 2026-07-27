# SLM Pipeline

Torrix provides a complete workflow for reducing LLM costs using fine-tuned small models (SLMs). The pipeline has five stages: identify candidates, export training data, fine-tune externally, route traffic, and monitor quality.

## Stage 1: Identify SLM candidates

Go to the **Analytics** page and click **Analyse runs** in the SLM Opportunities card.

Torrix scores each repeated prompt pattern across your last 30 days of runs against 7 signals:

| Signal | Weight | Reasoning |
|---|---|---|
| Repeated 5 or more times in 7 days | +3 | Templated task |
| Average input tokens below 200 | +2 | Short structured prompt |
| Structured output (JSON or tool calling) | +2 | Schema-constrained response |
| Average output tokens below 150 | +1 | Formulaic response |
| Zero errors | +1 | Consistent and predictable |
| All scored runs passing | +1 | Quality already validated |
| Cache hit rate above 50% | +1 | Deterministic enough for an SLM |

Verdicts:

- **Strong** (score 6 or above): high confidence SLM candidate with estimated monthly savings
- **Possible** (score 3 to 5): likely candidate, worth reviewing
- Tasks scoring below 3 are not shown

## Stage 2: Export training data

Click **Export training data (JSONL)** in the SLM Opportunities card.

Each line is one training example in the standard fine-tuning format:

```json
{"messages": [
  {"role": "system", "content": "Classify as: billing, technical, or account."},
  {"role": "user", "content": "I was charged twice for my subscription"},
  {"role": "assistant", "content": "billing"}
]}
```

The export includes:
- System prompt (if present in the original run)
- User message
- Assistant response

Only runs with `score = 1` (passing) are included. If fewer than 50% of candidate runs have been scored, the card shows an amber warning recommending you enable Online Evals first.

### Improve training data quality

Enable Online Evals in **Settings > AI Judge** per project before exporting. Torrix will auto-score every incoming run using your configured judge. After a few days, re-export and all included examples will be quality-validated.

## Stage 3: Fine-tune externally

Take the JSONL file to any fine-tuning platform:

| Platform | Notes |
|---|---|
| OpenAI fine-tuning | Upload at platform.openai.com/fine-tuning. Creates a custom `ft:gpt-4o-mini:...` model. |
| Mistral fine-tuning | Upload at console.mistral.ai/fine-tuning |
| Axolotl | `axolotl train config.yml` on a GPU instance |
| Unsloth | Faster training on a single GPU, works with Llama and Mistral base models |
| Ollama | `ollama create my-model -f Modelfile` after fine-tuning |

Typical fine-tuning time for a classification task with 200 to 500 examples: 30 minutes to 2 hours.

## Stage 4: Deploy and route

Deploy your fine-tuned model on any OpenAI-compatible server:

```bash
# Ollama
ollama serve
# vLLM
python -m vllm.entrypoints.openai.api_server --model ./my-slm --port 8000
# llama.cpp
./server -m my-slm.gguf --port 8080
```

Then create a routing rule in **Settings > Routing**, expand **Advanced: SLM endpoint routing**:

- **From model**: the original model (e.g. `gpt-4o`)
- **To model**: your SLM name (e.g. `my-classifier`)
- **SLM endpoint URL**: base URL of your SLM server (e.g. `http://localhost:11434/v1`)
- **Quality threshold**: optional, 0 to 1 (e.g. `0.8`)

Torrix appends the original path (`/v1/chat/completions`) to your endpoint URL and forwards the request. No code change is required in your application.

### Via API

```http
POST /api/routing-rules
Authorization: Bearer <your-torrix-api-key>
Content-Type: application/json

{
  "name": "classify-to-slm",
  "from_model": "gpt-4o",
  "to_model": "my-classifier",
  "condition_type": "model",
  "target_url": "http://localhost:11434/v1",
  "quality_threshold": 0.8,
  "fallback_model": "gpt-4o"
}
```

## Stage 5: Monitor quality

Every run routed to your SLM is tagged with `{ "slm_routed": true }` and appears in the Runs list with a normal run entry.

If Online Evals is enabled for the project, each routed run is auto-scored. Torrix checks the last 5 scored runs every 60 seconds. If the average score drops below the configured `quality_threshold`:

1. The routing rule is automatically disabled
2. All traffic reverts to the original LLM immediately
3. A webhook alert fires with event type `slm_quality_degraded`

### Webhook payload

```json
{
  "event": "slm_quality_degraded",
  "rule_id": "abc-123",
  "rule_name": "classify-to-slm",
  "from_model": "gpt-4o",
  "slm_model": "my-classifier",
  "slm_endpoint": "http://localhost:11434/v1",
  "avg_score": 0.4,
  "threshold": 0.8,
  "scored_runs": 5,
  "action_taken": "routing_rule_disabled",
  "timestamp": "2026-07-27T10:00:00Z"
}
```

## SLM Performance card

The **SLM Performance** card on the Analytics page loads automatically when at least one routing rule has a custom endpoint. It shows per rule:

- Total calls routed to the SLM
- Actual SLM cost
- Estimated LLM cost (based on original model average cost per call)
- Savings amount and percentage
- Average quality score with colour coding (green: above threshold, amber: borderline, red: degraded)

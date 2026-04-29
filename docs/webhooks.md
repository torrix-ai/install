# Outbound Webhooks

Torrix can fire HTTP webhooks on three events: budget threshold breaches, LLM request errors, and weekly cost digests.

## Setup

Go to **Settings > Budget Controls** and fill in:

- **Webhook URL**: any `https://` POST endpoint
- **Enable budget alert webhook**: fires when your daily spend crosses the threshold
- **Also fire webhook on LLM request errors**: fires on any upstream HTTP 4xx or 5xx
- **Send weekly cost digest**: fires every 7 days with a full cost summary (Pro only)

## Slack

Webhook URLs starting with `https://hooks.slack.com/` are automatically formatted as native Slack messages using the Block Kit format.

## Payload format

### Budget alert

```json
{
  "event": "budget_exceeded",
  "period": "daily",
  "spent_usd": 5.24,
  "threshold_usd": 5.00,
  "timestamp": "2026-04-27T14:00:00.000Z"
}
```

### LLM error alert

```json
{
  "event": "run.error",
  "run_id": "abc-123",
  "model": "gpt-4o",
  "status": 401,
  "error": null,
  "timestamp": "2026-04-27T14:00:00.000Z"
}
```

### Weekly cost digest (Pro)

```json
{
  "event": "weekly_digest",
  "period_start": "2026-04-22T00:00:00.000Z",
  "period_end": "2026-04-29T00:00:00.000Z",
  "total_cost_usd": 12.45,
  "total_runs": 342,
  "error_count": 5,
  "error_rate_pct": 1.46,
  "top_models": [
    { "model": "gpt-4o", "cost_usd": 8.20, "runs": 180 },
    { "model": "gpt-4o-mini", "cost_usd": 2.10, "runs": 120 }
  ],
  "vs_prior_week": {
    "cost_change_pct": 15.2,
    "runs_change_pct": 8.0
  }
}
```

## PagerDuty

To route Torrix alerts to PagerDuty, set up a Generic Webhook integration in your PagerDuty service and paste the endpoint URL into the Torrix webhook field. Use the `run.error`, `budget_exceeded`, or `weekly_digest` event field to write routing rules in PagerDuty.

## Notes

- Webhook failures are silent. Torrix does not retry or log failed deliveries.
- Budget alerts fire at most once per day per user to avoid alert fatigue.
- Error alerts fire on every request that returns HTTP 4xx or 5xx from the upstream LLM provider.
- Weekly digests fire every 7 days. Torrix checks hourly and fires if 7+ days have passed since the last digest.

## Security

All webhook payloads include a `x-torrix-signature` header containing an HMAC-SHA256 signature of the request body, keyed with your Torrix API key. Verify the signature on your receiving endpoint to ensure authenticity.

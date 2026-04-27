# Outbound Webhooks

Torrix can fire HTTP webhooks on two events: budget threshold breaches and LLM request errors.

## Setup

Go to **Settings > Budget Controls** and fill in:

- **Webhook URL**: any `https://` POST endpoint
- **Enable budget alert webhook**: fires when your daily spend crosses the threshold
- **Also fire webhook on LLM request errors**: fires on any upstream HTTP 4xx or 5xx

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

## PagerDuty

To route Torrix alerts to PagerDuty, set up a Generic Webhook integration in your PagerDuty service and paste the endpoint URL into the Torrix webhook field. Use the `run.error` or `budget_exceeded` event field to write routing rules in PagerDuty.

## Notes

- Webhook failures are silent. Torrix does not retry or log failed deliveries.
- Budget alerts fire at most once per day per user to avoid alert fatigue.
- Error alerts fire on every request that returns HTTP 4xx or 5xx from the upstream LLM provider.

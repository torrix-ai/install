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

## Verifying webhook signatures (HMAC)

Every outbound webhook includes an `x-torrix-signature` header containing an HMAC-SHA256 signature of the request body. This lets you confirm the webhook genuinely came from your Torrix instance and was not forged by a third party.

**Header format:** `x-torrix-signature: sha256=<hex>`

**Your signing secret** is available via the API:

```bash
curl -s http://localhost:8088/api/alert-settings \
  -H "Authorization: Bearer trxk_YOUR_KEY" | jq .webhook_secret
```

**Verification example (Node.js):**

```javascript
const crypto = require('crypto');

function verifySignature(rawBody, signature, secret) {
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(rawBody)
    .digest('hex');
  return signature === expected;
}

// In your webhook handler:
const isValid = verifySignature(req.body, req.headers['x-torrix-signature'], YOUR_SECRET);
if (!isValid) return res.status(401).send('Invalid signature');
```

**Verification example (bash):**

```bash
echo -n "$BODY" | openssl dgst -sha256 -hmac "$WEBHOOK_SECRET"
```

The secret is generated automatically on first webhook send and persists across restarts. It is unique to your Torrix instance.

## PagerDuty

To route Torrix alerts to PagerDuty, set up a Generic Webhook integration in your PagerDuty service and paste the endpoint URL into the Torrix webhook field. Use the `run.error` or `budget_exceeded` event field to write routing rules in PagerDuty.

## Notes

- Webhook failures are silent. Torrix does not retry or log failed deliveries.
- Budget alerts fire at most once per day per user to avoid alert fatigue.
- Error alerts fire on every request that returns HTTP 4xx or 5xx from the upstream LLM provider.

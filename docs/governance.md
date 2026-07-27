# AI Governance

Torrix includes a governance layer that sits between your application and the LLM. Every proxied request is evaluated against your governance policies before it is forwarded. Blocked requests never reach the model.

## Governance policies

A governance policy is a named rule with three parts:

- **Condition type** - what to evaluate
- **Condition value** - the value to compare against
- **Action** - what to do when the condition matches (block or flag)

### Condition types

| Condition type | What it checks | Example value |
|---|---|---|
| `model_not_in_list` | Request model is not in the approved list | `["gpt-4o-mini","claude-haiku-4-5-20251001"]` |
| `prompt_contains` | Prompt text contains a substring (case-insensitive) | `confidential` |
| `prompt_regex` | Prompt text matches a regular expression | `\bpassword\b` |
| `cost_above` | Estimated per-request cost exceeds a threshold in USD | `0.05` |

### Actions

- **block** - the proxy returns HTTP 403 with `{ "error": "governance_policy_blocked", "policy": "<name>" }`. The request never reaches the model.
- **flag** - the request proceeds normally but the run is tagged with `{ "flagged_policy": "<name>" }`. Visible in the Runs list and exportable in compliance reports.

## Setting up policies

Go to **Settings** and open the **Governance** tab.

1. Enter a policy name.
2. Choose a condition type and enter the condition value.
   - For `model_not_in_list`, enter model names separated by commas. Torrix converts them to a JSON array automatically.
3. Choose an action (Block or Flag).
4. Click **Add Policy**.

Policies are evaluated in creation order. You can enable and disable individual policies using the toggle without deleting them.

## API

Governance policies are also manageable via the REST API.

### List policies

```http
GET /api/governance/policies
Authorization: Bearer <your-torrix-api-key>
```

### Create a policy

```http
POST /api/governance/policies
Authorization: Bearer <your-torrix-api-key>
Content-Type: application/json

{
  "name": "Block unapproved models",
  "condition_type": "model_not_in_list",
  "condition_value": "[\"gpt-4o-mini\",\"claude-haiku-4-5-20251001\"]",
  "action": "block"
}
```

### Delete a policy

```http
DELETE /api/governance/policies/:id
Authorization: Bearer <your-torrix-api-key>
```

### Enable or disable a policy

```http
PATCH /api/governance/policies/:id/toggle
Authorization: Bearer <your-torrix-api-key>
Content-Type: application/json

{ "enabled": 1 }
```

## Compliance report export

The Compliance page at `/ui/compliance` lets you download a dated CSV covering all AI activity and audit events for a selected period.

### What the report contains

**Section 1 - AI activity log**

One row per AI call: `id`, `created_at`, `model`, `provider`, `project_id`, `end_user_id`, `input_tokens`, `output_tokens`, `cost_usd`, `latency_ms`, `status`, `finish_reason`, `is_anomaly`, `source`, `score`, `api_key_id`, `trace_id`.

**Section 2 - Audit log**

One row per settings or admin action: `created_at`, `action`, `target`, `detail`, `ip`, `email`.

### Generating a report

1. Go to `/ui/compliance`.
2. Set the date range and optionally filter by project.
3. Click **Load Summary** to preview counts before downloading.
4. Click **Download CSV** to get the full report.

### Via API

```http
GET /api/compliance/report.csv?from=2026-06-01&to=2026-06-30
Authorization: Bearer <your-torrix-api-key>
```

Add `&project_id=<uuid>` to scope to a single project.

### Summary endpoint

```http
GET /api/compliance/summary?from=2026-06-01&to=2026-06-30
Authorization: Bearer <your-torrix-api-key>
```

Returns:

```json
{
  "totalRuns": 4812,
  "flaggedRuns": 3,
  "anomalyRuns": 1,
  "errorRuns": 12,
  "totalCost": 9.4821,
  "auditActions": 47,
  "from": "2026-06-01",
  "to": "2026-06-30"
}
```

## Compliance frameworks

| Requirement | How Torrix helps |
|---|---|
| GDPR Art. 30 (records of processing) | Compliance CSV covers every AI call with full metadata and audit trail |
| EU AI Act (transparency and logging) | Every high-risk AI call is logged with model, inputs, outputs, and cost |
| SOC 2 Type II (change management, access) | Audit log captures all settings changes, key lifecycle events, and user actions |
| HIPAA (audit controls) | PII masking removes PHI before storage; audit log records every access event |

---

## SSO configuration via API

### Get current SSO config

```http
GET /api/sso/config
Authorization: Bearer <your-torrix-api-key>
```

Returns:

```json
{
  "issuer": "https://accounts.google.com",
  "client_id": "your-client-id",
  "configured": true
}
```

### Save SSO config

```http
POST /api/sso/config
Authorization: Bearer <your-torrix-api-key>
Content-Type: application/json

{
  "issuer": "https://accounts.google.com",
  "client_id": "your-client-id",
  "client_secret": "your-client-secret"
}
```

Admin access required. The client secret is stored server-side and never returned in GET responses.

### Remove SSO config

```http
DELETE /api/sso/config
Authorization: Bearer <your-torrix-api-key>
```

Removes all SSO configuration. Users will no longer see the Sign in with SSO button.

### Check SSO status (unauthenticated)

```http
GET /auth/sso/status
```

Returns `{ "configured": true, "issuer": "https://accounts.google.com" }`. Used by the login page to decide whether to show the SSO button. Safe to call without authentication.

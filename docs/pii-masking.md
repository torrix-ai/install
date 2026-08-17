# PII Detection and Masking

Torrix can automatically detect and redact sensitive data from prompts and responses before they are stored in SQLite. Available in all editions.

---

## What gets masked

| Category | Example input | Stored as |
|---|---|---|
| Email | `user@example.com` | `[EMAIL]` |
| Phone | `+1-555-123-4567` | `[PHONE]` |
| Credit card | `4111 1111 1111 1111` | `[CREDIT_CARD]` |
| IP address | `192.168.1.1` | `[IP_ADDRESS]` |

Masking applies to: prompt text, response text, request body JSON, thinking text, and tool call arguments.

---

## What is NOT detected

PII masking uses regex patterns. It reliably catches **structured PII** (emails, phone numbers, card numbers, IPs). It does **not** detect:

- Personal names
- Physical addresses
- Custom identifiers (account numbers, employee IDs, internal codes)
- Freeform text that describes personal information without a recognisable pattern

For those cases, use governance policies to block or flag prompts containing sensitive keywords before they reach the LLM.

---

## How to enable

1. Open Torrix at `http://localhost:8088`
2. Go to **Settings** and click the **Privacy** tab
3. Toggle **Enable PII Masking** on
4. Select which categories to detect (email, phone, credit card, IP)
5. Click **Save**

Masking takes effect immediately for new runs. Existing stored data is not modified retroactively.

---

## Before and after

**Stored prompt (masking off):**
```
User email is alice@example.com, phone is 555-867-5309.
```

**Stored prompt (masking on, email + phone enabled):**
```
User email is [EMAIL], phone is [PHONE].
```

The actual request forwarded to the LLM provider is **not modified** — masking applies only to what Torrix writes to the database.

---

## Custom patterns (Pro / Enterprise)

Pro and Enterprise installations can add custom regex patterns to catch organisation-specific identifiers:

1. Go to **Settings → Privacy → Custom Patterns**
2. Add a regex and a replacement label, e.g.:
   - Pattern: `EMP-\d{6}`
   - Label: `[EMPLOYEE_ID]`

Custom patterns are applied after the built-in categories.

---

## Performance impact

PII detection runs synchronously after the upstream response is received, before writing to SQLite. For typical prompt/response sizes (< 10 KB) the overhead is under 1 ms. For very long responses (> 100 KB), expect 2-5 ms additional latency per run.

---

## Verify masking is working

After enabling masking, send a test prompt containing an email address, then:

1. Open the **Runs** tab in the Torrix dashboard
2. Click the run you just created
3. In the **Prompt** section, the email should appear as `[EMAIL]`

If the raw email is still visible, check that the **Email** category is checked under Settings → Privacy.

---

## Notes

- Each category can be enabled or disabled independently.
- Changes are recorded in the audit log (**Settings → Audit Log**, action: `settings.update`).
- PII masking does not affect the HTTP response returned to your application — only what is stored.
- If you need to redact data from existing runs, use the bulk delete API and re-ingest clean data.

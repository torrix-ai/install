# PII Detection and Masking

Torrix can automatically detect and redact sensitive data from prompts and responses before they are stored. Available in all editions.

## What gets masked

| Category | Example input | Stored as |
|---|---|---|
| Email | user@example.com | [EMAIL] |
| Phone | +1-555-123-4567 | [PHONE] |
| Credit card | 4111 1111 1111 1111 | [CREDIT_CARD] |
| IP address | 192.168.1.1 | [IP_ADDRESS] |

Masking applies to: prompt text, response text, request body, thinking text, and tool call arguments.

## How to enable

1. Open Torrix at http://localhost:8088
2. Go to Settings and click the **Privacy** tab
3. Toggle **Enable PII Masking** on
4. Select which categories to detect
5. Click **Save**

Masking takes effect immediately for new runs. Existing stored data is not modified.

## Notes

- Detection uses regex patterns. It catches structured PII reliably. Names and addresses are not detected.
- Each category can be enabled or disabled independently.
- Changes are recorded in the audit log (Settings > Audit Log).

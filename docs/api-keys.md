# API Key Management

API key management is available on Torrix Pro and Enterprise editions. Community edition uses a single default key per user.

## Overview

Torrix supports multiple named API keys per user. Each key can be:

- **Scoped** to all projects or specific projects only
- **Permissioned** as full access (read + write) or read-only
- **Revoked** instantly without affecting other keys
- **Tracked** with automatic last-used timestamps

Keys are SHA-256 hashed before storage. The full key is shown only once at creation time.

## Creating Keys

### Via UI

1. Navigate to Settings > API Keys
2. Click "+ Create Key"
3. Enter a name (e.g., `production-sdk`, `ci-pipeline`, `grafana-readonly`)
4. Choose permission level and scope
5. Click "Generate Key"
6. Copy the full key immediately (it will not be shown again)

### Via API

```bash
curl -X POST http://localhost:8088/api/keys \
  -H "Authorization: Bearer <your-api-key>" \
  -H "Content-Type: application/json" \
  -d '{"name": "production-sdk", "scope": "full", "permission": "read_write"}'
```

Response (key shown only in this response):
```json
{
  "id": "uuid",
  "key": "trxk_550e8400e29b41d4a716446655440000",
  "prefix": "trxk_550e",
  "name": "production-sdk",
  "scope": "full",
  "projects": null,
  "permission": "read_write"
}
```

### Scoped keys

To restrict a key to specific projects:

```bash
curl -X POST http://localhost:8088/api/keys \
  -H "Authorization: Bearer <your-api-key>" \
  -H "Content-Type: application/json" \
  -d '{"name": "chatbot-only", "scope": "projects", "projects": ["my-chatbot"], "permission": "read_write"}'
```

### Read-only keys

Read-only keys can access dashboard, analytics, and run data but cannot send requests through the proxy or ingest new data:

```bash
curl -X POST http://localhost:8088/api/keys \
  -H "Authorization: Bearer <your-api-key>" \
  -H "Content-Type: application/json" \
  -d '{"name": "grafana-reader", "scope": "full", "permission": "read_only"}'
```

## Listing Keys

List your own keys:

```bash
curl http://localhost:8088/api/keys \
  -H "Authorization: Bearer <your-api-key>"
```

Admin users can list all keys across all users:

```bash
curl "http://localhost:8088/api/keys?all=true" \
  -H "Authorization: Bearer <admin-api-key>"
```

Response includes key prefix (never the full key), scope, permission, and timestamps:

```json
[
  {
    "id": "uuid",
    "name": "production-sdk",
    "key_prefix": "trxk_550e",
    "scope": "full",
    "projects": null,
    "permission": "read_write",
    "created_at": "2026-05-03T10:00:00.000Z",
    "last_used_at": "2026-05-03T21:30:00.000Z",
    "revoked_at": null
  }
]
```

## Revoking Keys

Revoke a key by ID. The key stops working immediately:

```bash
curl -X DELETE http://localhost:8088/api/keys/<key-id> \
  -H "Authorization: Bearer <your-api-key>"
```

Response:
```json
{"ok": true}
```

Revoked keys remain visible to admins in the full listing (with a non-null `revoked_at` timestamp) for audit purposes but are filtered from the regular user view.

## Scope Enforcement

| Key Permission | Allowed Operations |
|---------------|-------------------|
| **Full access** | Proxy requests, SDK ingest, dashboard reads, key management |
| **Read only** | Dashboard reads, analytics, run/event viewing only |

| Key Scope | Behavior |
|-----------|----------|
| **All projects** | No project restriction applied |
| **Specific projects** | Proxy requests must include `x-torrix-project` header matching an allowed project |

If a read-only key attempts a write operation:
```json
{"error": "API key does not have write access to this project"}
```

## Security Best Practices

1. **Name keys by integration** so you know which to revoke if compromised (e.g., `n8n-prod`, `ci-tests`, `monitoring`)
2. **Use read-only keys** for dashboards, Grafana, and monitoring tools that only need to read data
3. **Scope keys to projects** when different teams or services use different projects
4. **Rotate periodically** by creating a new key, updating your integration, then revoking the old one
5. **Never share keys** across environments (use separate keys for dev, staging, production)

## Migration from Single Key

When upgrading to Pro, your existing API key is automatically migrated into the new system with the name "Default", full scope, and read-write permission. It continues to work without any changes to your integrations.

## API Reference

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/keys` | List own keys (admin: add `?all=true` for all users) |
| POST | `/api/keys` | Create a new key |
| DELETE | `/api/keys/:id` | Revoke a key |
| GET | `/api/settings/permissions` | Get visible tabs and role info |

## Community Edition

On Community edition, API key management endpoints return:

```json
{"error": "API key management requires Pro or Enterprise"}
```

The single default key per user continues to work normally.

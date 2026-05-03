# Team Management

Team management is available on Torrix Pro and Enterprise editions. Community edition is single-user only.

## Overview

Torrix supports per-project role-based access control with three roles:

| Role | Permissions |
|------|------------|
| **Owner** | Manage project members, delete project, plus all Editor permissions |
| **Editor** | Create runs (proxy/SDK), score runs, add notes, bulk operations |
| **Viewer** | Read-only access to dashboard, analytics, runs, and events |

The **instance admin** (first user created on a fresh instance) bypasses all role checks and can manage all projects and users.

## Creating Users

Only the instance admin can create new user accounts. This is done from the Team page in the UI or via the API.

### Via UI

1. Navigate to `/ui/team`
2. Fill in email and temporary password (minimum 8 characters)
3. Click "Add User"
4. The new user will be prompted to change their password on first login

### Via API

```bash
curl -X POST http://localhost:8088/api/admin/users \
  -H "Authorization: Bearer <admin-api-key>" \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "temppass123"}'
```

Response:
```json
{
  "id": "uuid",
  "email": "user@example.com",
  "api_key": "trxk_...",
  "must_change_password": true
}
```

## Managing Project Members

### Add a member to a project

```bash
curl -X POST http://localhost:8088/api/projects/<project-id>/members \
  -H "Authorization: Bearer <admin-api-key>" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "<user-id>", "role": "editor"}'
```

### Change a member's role

```bash
curl -X PUT http://localhost:8088/api/projects/<project-id>/members/<user-id> \
  -H "Authorization: Bearer <admin-api-key>" \
  -H "Content-Type: application/json" \
  -d '{"role": "viewer"}'
```

### Remove a member

```bash
curl -X DELETE http://localhost:8088/api/projects/<project-id>/members/<user-id> \
  -H "Authorization: Bearer <admin-api-key>"
```

### List project members

```bash
curl http://localhost:8088/api/projects/<project-id>/members \
  -H "Authorization: Bearer <admin-api-key>"
```

## Proxy Authorization

When sending requests through the proxy with a project header, the user must have at least **Editor** role on that project:

```bash
curl -X POST http://localhost:8088/proxy \
  -H "Authorization: Bearer <user-api-key>" \
  -H "x-target-url: https://api.openai.com/v1/chat/completions" \
  -H "x-upstream-authorization: Bearer <openai-key>" \
  -H "x-torrix-project: my-chatbot" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello"}]}'
```

If the user lacks Editor access, the proxy returns:
```json
{"error": "No write access to this project"}
```

## Password Change

Users created by an admin must change their password on first login. The login response includes a `must_change_password` flag, and the UI shows a password change form automatically.

To change password via API:

```bash
curl -X PUT http://localhost:8088/auth/change-password \
  -H "Authorization: Bearer <user-api-key>" \
  -H "Content-Type: application/json" \
  -d '{"current_password": "temppass123", "new_password": "newsecurepass"}'
```

## Admin API Reference

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/admin/users` | List all users |
| POST | `/api/admin/users` | Create a new user |
| DELETE | `/api/admin/users/:id` | Delete a user |
| GET | `/api/projects/:id/members` | List project members |
| POST | `/api/projects/:id/members` | Add a member |
| PUT | `/api/projects/:id/members/:uid` | Change role |
| DELETE | `/api/projects/:id/members/:uid` | Remove a member |

## Community Edition

On Community edition, team features are disabled. The single user is automatically an admin with full access to all projects. Attempting to access team endpoints returns:

```json
{"error": "Team features require Pro or Enterprise"}
```

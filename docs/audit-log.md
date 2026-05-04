# Audit Log

The audit log records every write action taken in your Torrix instance. It gives administrators a tamper-visible history of who changed what and when.

## Where to find it

Open **Settings** and click the **Audit Log** tab. The tab is visible to admin users only.

## What is logged

Every event includes:

| Field | Description |
|-------|-------------|
| Timestamp | Exact date and time (UTC) |
| Action | Category and action, e.g. `key.create` or `project.delete` |
| User | Email of the user who triggered the action |
| Target | The affected resource name or identifier |
| IP address | The IP of the request |

### Tracked actions

| Category | Actions |
|----------|---------|
| User | Signup, login, password change |
| API key | Create, revoke |
| Project | Create, delete, member add, member role update, member remove |
| Admin | User create, user delete |
| Settings | Sampling update, alert update |
| License | Activate, deactivate |
| Routing | Rule create, rule delete |

## Filtering

Use the category dropdown at the top of the tab to filter by action type: All, User, Key, Project, Admin, Settings, License, or Routing.

## Access control

The Audit Log tab is visible and accessible to admin users only. Non-admin accounts receive a 403 response from the `/api/audit-logs` endpoint. In the Community edition, the single user is always the admin.

## Pagination

The log loads 50 entries at a time. Click **Load more** to fetch older entries.

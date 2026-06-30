---
title: Team
description: Create, update, delete and list team members (staff users) via the Grow CRM API.
---

# Team

The Team API provides CRUD access to team members (staff users).

> New to the API? Start with **[Getting started](/getting-started)** (base URL, response
> envelope, errors, pagination) and **[Authentication](/authentication)** (API keys).

> **Identifier note:** team members are addressed by their numeric **user `id`**.

> **A team member is a staff user** (`type='team'`) with a role (Administrator / Staff). Creating
> one provisions a login (a random password is generated) and a personal workspace; the welcome
> email is opt-in.

> **Scope:** per-user preferences, avatar upload and role management (a separate concern) are out
> of API scope. Web events are not fired.

## The team member object

```json
{
  "id": 280,
  "first_name": "Jane",
  "last_name": "Doe",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "phone": "12345",
  "job_position": "Developer",
  "role": { "id": 3, "name": "Staff" },
  "status": "active",
  "dashboard_access": "yes",
  "social": { "facebook": null, "twitter": null, "linkedin": null, "github": null, "dribbble": null },
  "dates": { "created": "...", "updated": "..." }
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | User id. |
| `first_name` / `last_name` / `name` | string | |
| `email` | string | Unique across all users. |
| `role` | object | `{ id, name }` (Administrator=1, Staff=3). |
| `status` | string | `active` / `suspended` / `deleted`. |
| `job_position` / `phone` | string | Optional. |

---

## List / search team members

```
GET /api/team
```

Query params: `status`, `role_id`, `search`, `sort` (`first_name`,`last_name`,`email`,`created`),
`order`, `limit`, `page`. Soft-deleted members are excluded by default.

```bash
curl -G https://your-domain/api/team -H "Authorization: Bearer YOUR_API_KEY" -d role_id=3
```

---

## Get a team member

```
GET /api/team/{id}
```

---

## Create a team member

```
POST /api/team
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `first_name` | string | yes | |
| `last_name` | string | yes | |
| `email` | string | yes | Unique across all users. |
| `role_id` | integer | yes | A team role (Administrator=1 / Staff=3). |
| `phone` | string | no | |
| `position` | string | no | |
| `send_email` | string | no | `yes` to send the welcome email (with login details). Default no. |

```bash
curl -X POST https://your-domain/api/team -H "Authorization: Bearer YOUR_API_KEY" \
  -d first_name=Jane -d last_name=Doe -d email=jane@example.com -d role_id=3 -d send_email=yes
```

Returns the new team member (`201`, status `active`).

---

## Update a team member

```
PATCH /api/team/{id}
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `first_name` / `last_name` | string | yes | |
| `email` | string | yes | Unique (ignoring this user). |
| `role_id` | integer | no | |
| `phone` / `position` | string | no | |
| `password` | string | no | New password (min 5), applied if supplied. |

---

## Delete a team member

```
DELETE /api/team/{id}
```

Soft-deletes the account (status `deleted`, login cleared) and removes the member's project/task/
lead assignments, project-manager rows and calendar events.

---

## Errors

See [Getting started](/getting-started#errors). Team-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The team member id does not exist. |
| `422 Unprocessable Entity` | Validation failed (missing name/email/role, duplicate email, or an invalid role). |

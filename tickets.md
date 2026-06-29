---
title: Tickets
description: Create, update, delete, list, status, archive, tag and reply to support tickets via the Grow CRM API.
---

# Tickets

The Tickets API provides CRUD access to support tickets plus status, archive/restore, tags and
the reply conversation.

> New to the API? Start with **[Getting started](getting-started.md)** (base URL, response
> envelope, errors, pagination) and **[Authentication](authentication.md)** (API keys).

> **Identifier note:** tickets are addressed by their numeric **`ticket_id`**.

> **A ticket belongs to a department and a client.** The department is a category of type
> `ticket`. Tickets are created in **client mode** (an existing client). The web's email/IMAP
> "new user" ingestion is out of API scope.

> **Silent semantics.** Creating tickets and posting replies records the data and updates the
> ticket, but does **not** fire the web's events/notifications, send the event-driven client
> emails, or trigger web→IMAP conversion (consistent with the rest of the API).

> **Scope:** file attachments (upload/download), canned responses, sticky notes, pinning, bulk
> actions, inbound IMAP/email threading and the history view are out of API scope.

## The ticket object

```json
{
  "id": 32,
  "subject": "Cannot log in",
  "message": "I get an error on the login page.",
  "status": { "id": 1, "title": "Open" },
  "priority": "high",
  "source": "web",
  "active_state": "active",
  "department": { "id": 72, "name": "Sales" },
  "client": { "id": 1, "name": "Acme Inc" },
  "project_id": null,
  "creator_id": 1,
  "dates": { "created": "...", "last_updated": "...", "status_changed": null },
  "replies_count": 2,
  "tags": ["billing"]
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Ticket id. |
| `subject` / `message` | string | The opening message. |
| `status` | object | `{ id, title }` (1 Open, 2 Closed, 3 On Hold, 4 Answered, 5 Customer Replied). |
| `priority` | string | `normal`, `high` or `urgent`. |
| `source` | string | `web` or `email` (set automatically). |
| `active_state` | string | `active` or `archived`. |
| `department` | object | The category (type `ticket`). |
| `client` / `project_id` | object/int | Links. |
| `replies_count` | int | Number of replies. |
| `tags` | array | Tag titles. |

---

## List / search tickets

```
GET /api/tickets
```

Query params: `status`, `priority`, `category_id`, `client_id`, `project_id`, `active_state`,
`search`, `tags[]`, `sort` (`ticket_created`,`ticket_last_updated`,`ticket_priority`,`ticket_status`),
`order`, `limit`, `page`.

```bash
curl -G https://your-domain/api/tickets -H "Authorization: Bearer YOUR_API_KEY" -d status=1
```

---

## Get a ticket

```
GET /api/tickets/{id}
```

---

## Create a ticket

```
POST /api/tickets
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `ticket_clientid` | integer | yes | Existing client. |
| `ticket_categoryid` | integer | yes | A department (category of type `ticket`). |
| `ticket_projectid` | integer | no | Must belong to the client. |
| `ticket_subject` | string | yes | |
| `ticket_message` | string | yes | |
| `ticket_priority` | string | yes | `normal`, `high` or `urgent`. |
| custom fields | mixed | varies | Any enabled ticket custom fields (required ones are enforced). |

```bash
curl -X POST https://your-domain/api/tickets -H "Authorization: Bearer YOUR_API_KEY" \
  -d ticket_clientid=1 -d ticket_categoryid=72 -d ticket_subject="Cannot log in" \
  -d ticket_message="I get an error" -d ticket_priority=high
```

Returns the new ticket (`201`).

---

## Update a ticket

```
PATCH /api/tickets/{id}
```

Edits the department/project/subject/message/priority (status has its own setter).

| Parameter | Type | Required |
|---|---|---|
| `ticket_categoryid` | integer | yes |
| `ticket_projectid` | integer | no |
| `ticket_subject` | string | yes |
| `ticket_message` | string | yes |
| `ticket_priority` | string | yes |

---

## Delete a ticket

```
DELETE /api/tickets/{id}
```

---

## Change status

```
PUT /api/tickets/{id}/status
```

| Parameter | Type | Notes |
|---|---|---|
| `status` | integer | A `ticketstatus_id` (1 Open, 2 Closed, 3 On Hold, 4 Answered, 5 Customer Replied). |

```bash
curl -X PUT https://your-domain/api/tickets/32/status -H "Authorization: Bearer YOUR_API_KEY" -d status=2
```

---

## Archive / restore

```
PUT /api/tickets/{id}/archive
PUT /api/tickets/{id}/restore
```

---

## Set tags

```
PUT /api/tickets/{id}/tags
```

Full-set replace (empty clears).

---

## The conversation (replies)

```
GET    /api/tickets/{id}/replies            list the conversation
POST   /api/tickets/{id}/replies            post a reply
DELETE /api/tickets/{id}/replies/{reply_id} delete a reply
```

Post a reply:

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `ticketreply_text` | string | yes | The message. |
| `ticketreply_type` | string | no | `reply` (default) or `note`. |

```bash
curl -X POST https://your-domain/api/tickets/32/replies -H "Authorization: Bearer YOUR_API_KEY" \
  -d ticketreply_text="We are looking into this for you."
```

Returns the new reply (`201`). Posting a reply updates the ticket's last-updated timestamp but
does not send a client email (silent semantics).

---

## Errors

See [Getting started](getting-started.md#errors). Ticket-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The ticket (or reply) id does not exist. |
| `422 Unprocessable Entity` | Validation failed (missing fields, non-ticket department, invalid priority/status, or a project that does not belong to the client). |

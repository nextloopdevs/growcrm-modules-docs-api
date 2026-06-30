---
title: Notes
description: Create, update, delete, list and tag notes via the CRM API.
---

# Notes

The Notes API provides CRUD access to notes plus tags.

> New to the API? Start with **[Getting started](/getting-started)** (base URL, response
> envelope, errors, pagination) and **[Authentication](/authentication)** (API keys).

> **Identifier note:** notes are addressed by their numeric **`note_id`**.

> **A note has a title + description** and may optionally be **attached to a resource**
> (client/lead/project/task/user). Visibility is `public` or `private`.

> **Scope:** per-user **sticky-notes** are excluded (the API never returns or creates them),
> as are file attachments, starring and the "my notes" feature. Web events are not fired.

## The note object

```json
{
  "id": 74,
  "title": "Follow-up call",
  "description": "Discuss the renewal terms.",
  "visibility": "public",
  "resource": { "type": "client", "id": 1 },
  "creator_id": 1,
  "tags": ["sales"],
  "dates": { "created": "...", "updated": "..." }
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Note id. |
| `title` / `description` | string | |
| `visibility` | string | `public` or `private`. |
| `resource` | object | The attached resource (`type`, `id`), or `null`/`null` if standalone. |
| `creator_id` | int | The API admin. |
| `tags` | array | Tag titles. |

---

## List / search notes

```
GET /api/notes
```

Query params: `resource_type`, `resource_id`, `visibility`, `search`,
`sort` (`note_title`,`note_created`,`note_updated`), `order`, `limit`, `page`.
Sticky-notes are always excluded.

```bash
curl -G https://your-domain/api/notes -H "Authorization: Bearer YOUR_API_KEY" -d resource_type=client -d resource_id=1
```

---

## Get a note

```
GET /api/notes/{id}
```

---

## Create a note

```
POST /api/notes
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `note_title` | string | yes | |
| `note_description` | string | yes | |
| `note_visibility` | string | no | `public` (default) or `private`. |
| `noteresource_type` | string | no | `client`, `lead`, `project`, `task` or `user`. |
| `noteresource_id` | integer | no | Required if a type is given; validated to exist. |

```bash
curl -X POST https://your-domain/api/notes -H "Authorization: Bearer YOUR_API_KEY" \
  -d note_title="Follow-up call" -d note_description="Discuss renewal" \
  -d noteresource_type=client -d noteresource_id=1
```

Returns the new note (`201`).

---

## Update a note

```
PATCH /api/notes/{id}
```

Edits the title, description and visibility (the resource attachment is immutable).

| Parameter | Type | Required |
|---|---|---|
| `note_title` | string | yes |
| `note_description` | string | yes |
| `note_visibility` | string | no |

---

## Delete a note

```
DELETE /api/notes/{id}
```

---

## Set tags

```
PUT /api/notes/{id}/tags
```

Full-set replace (empty clears).

---

## Errors

See [Getting started](/getting-started#errors). Note-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The note id does not exist (or is a sticky-note). |
| `422 Unprocessable Entity` | Validation failed (missing title/description, invalid resource type, or a resource that does not exist). |

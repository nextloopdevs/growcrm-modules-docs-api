---
title: Checklists
description: Create, update, delete, list, reorder and comment on checklists via the CRM API.
---

# Checklists

The Checklists API manages checklist items that belong to a **parent resource** (for example a
task or a project), plus their status, ordering and comments.

> New to the API? Start with **[Getting started](/getting-started)** (base URL, response
> envelope, errors, pagination) and **[Authentication](/authentication)** (API keys). Those
> conventions apply to every endpoint below and are not repeated here.

> **Identifier note:** checklist items are addressed by their numeric **`checklist_id`**. Each
> item belongs to a parent identified by `checklistresource_type` + `checklistresource_id`
> (e.g. `task` / `123`). Listing is always scoped to a parent.

## The checklist object

```json
{
  "id": 45,
  "text": "Draft the proposal",
  "status": "pending",
  "position": 1,
  "resource": { "type": "task", "id": 123 },
  "comments_count": 2,
  "dates": {
    "created": "2026-06-17T09:30:00.000000Z",
    "updated": "2026-06-17T09:30:00.000000Z"
  }
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Checklist item id. Used in all URLs. |
| `text` | string | The checklist item text. |
| `status` | string | `pending` or `completed`. |
| `position` | integer | Manual sort order within the parent. |
| `resource` | object | The parent: `{ type, id }`. |
| `comments_count` | integer | Number of comments on the item. |
| `dates` | object | `created`, `updated`. |

---

## List / search checklists

```
GET /api/checklists
```

Always scoped to a parent — `resource_type` and `resource_id` are **required**. The list
response includes a `meta.progress` summary for the parent's full checklist set.

### Query parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `resource_type` | string | yes | Parent type (e.g. `task`, `project`). |
| `resource_id` | integer | yes | Parent id. |
| `sort` | string | no | `checklist_position` (default), `checklist_status`, `checklist_created`. |
| `order` | string | no | `asc` (default) or `desc`. |
| `limit` | integer | no | Results per page (max 100). |
| `page` | integer | no | Page number. |

### Example request

```bash
curl -G https://your-domain/api/checklists \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d resource_type=task \
  -d resource_id=123
```

```php
<?php
$query = http_build_query(['resource_type' => 'task', 'resource_id' => 123]);
$ch = curl_init("https://your-domain/api/checklists?$query");
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `200 OK`

```json
{
  "data": [ { "id": 45, "text": "Draft the proposal", "status": "pending" } ],
  "meta": {
    "current_page": 1, "per_page": 25, "total": 1, "last_page": 1,
    "progress": { "total": 1, "completed": 0, "percentage": 0 }
  },
  "message": "Checklists retrieved successfully."
}
```

---

## Get a checklist

```
GET /api/checklists/{id}
```

```bash
curl https://your-domain/api/checklists/45 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/checklists/45');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the checklist (`200`) with `message` "Checklist retrieved successfully."

---

## Create a checklist item

```
POST /api/checklists
```

New items are appended to the end of the parent's list (position is assigned automatically).

### Body parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `checklistresource_type` | string | yes | Parent type (e.g. `task`, `project`). |
| `checklistresource_id` | integer | yes | Parent id. |
| `checklist_text` | string | yes | The item text. |

### Example request

```bash
curl -X POST https://your-domain/api/checklists \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d checklistresource_type=task \
  -d checklistresource_id=123 \
  -d checklist_text="Draft the proposal"
```

```php
<?php
$ch = curl_init('https://your-domain/api/checklists');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'checklistresource_type' => 'task',
        'checklistresource_id'   => 123,
        'checklist_text'         => 'Draft the proposal',
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `201 Created`

```json
{
  "data": { "id": 45, "text": "Draft the proposal", "status": "pending", "position": 1 },
  "message": "Checklist created successfully."
}
```

---

## Update a checklist item

```
PATCH /api/checklists/{id}
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `checklist_text` | string | yes | The new item text. |

```bash
curl -X PATCH https://your-domain/api/checklists/45 \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d checklist_text="Draft and review the proposal"
```

```php
<?php
$ch = curl_init('https://your-domain/api/checklists/45');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PATCH',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['checklist_text' => 'Draft and review the proposal']),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the checklist (`200`) with `message` "Checklist updated successfully."

---

## Delete a checklist item

```
DELETE /api/checklists/{id}
```

Deletes the item and all of its comments.

```bash
curl -X DELETE https://your-domain/api/checklists/45 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/checklists/45');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'DELETE',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `200 OK`

```json
{
  "data": { "id": 45 },
  "message": "Checklist deleted successfully."
}
```

---

## Change status

```
PUT /api/checklists/{id}/status
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `status` | string | yes | `pending` or `completed`. |

```bash
curl -X PUT https://your-domain/api/checklists/45/status \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d status=completed
```

```php
<?php
$ch = curl_init('https://your-domain/api/checklists/45/status');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['status' => 'completed']),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the checklist (`200`) with `message` "Checklist status updated successfully."

---

## Reorder checklist items

```
PUT /api/checklists/positions
```

Sets each item's position to match the order of the supplied id array. This is a bulk action —
the ids are in the body.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `checklist_ids` | array of integers | yes | Checklist ids in the desired order. |

```bash
curl -X PUT https://your-domain/api/checklists/positions \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d "checklist_ids[]=47" -d "checklist_ids[]=45" -d "checklist_ids[]=46"
```

```php
<?php
$ch = curl_init('https://your-domain/api/checklists/positions');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['checklist_ids' => [47, 45, 46]]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `200 OK`

```json
{
  "data": { "checklist_ids": [47, 45, 46] },
  "message": "Checklist positions updated successfully."
}
```

---

## Comments

A checklist item can have comments.

### List comments

```
GET /api/checklists/{id}/comments
```

```bash
curl https://your-domain/api/checklists/45/comments \
  -H "Authorization: Bearer YOUR_API_KEY"
```

Returns `{ "data": [ { "id": 9, "text": "...", "creator": { "id": 1, "name": "Admin" } } ], "message": "..." }`.

### Add a comment

```
POST /api/checklists/{id}/comments
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `comment` | string | yes | The comment text. |

```bash
curl -X POST https://your-domain/api/checklists/45/comments \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d comment="Looks good to me"
```

```php
<?php
$ch = curl_init('https://your-domain/api/checklists/45/comments');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['comment' => 'Looks good to me']),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the comment (`201`) with `message` "Checklist comment added successfully."

### Delete a comment

```
DELETE /api/checklists/{id}/comments/{comment_id}
```

```bash
curl -X DELETE https://your-domain/api/checklists/45/comments/9 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

Returns `{ "data": { "id": 9 }, "message": "Checklist comment deleted successfully." }`.

---

## Errors

See [Getting started](/getting-started#errors) for the shared error format. Checklist-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The checklist (or comment) id does not exist. |
| `422 Unprocessable Entity` | Validation failed (e.g. missing `resource_type`/`resource_id` on list, missing `checklist_text`, or an invalid `status`). |

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "resource_type": ["The resource type field is required."],
    "resource_id": ["The resource id field is required."]
  }
}
```

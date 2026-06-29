---
title: Tasks
description: Create, update, delete, list, clone and manage project tasks via the Grow CRM API.
---

# Tasks

The Tasks API provides CRUD access to project tasks, plus ancillary actions for status,
assignment, tags, archive/activate, clone, comments and timers.

> New to the API? Start with **[Getting started](getting-started.md)** (base URL, response
> envelope, errors, pagination) and **[Authentication](authentication.md)** (API keys). Those
> conventions apply to every endpoint below and are not repeated here.

> **Identifier note:** tasks are addressed by their numeric **`task_id`** and always belong to a
> project (`task_projectid`). `task_status` is a `taskstatus_id` and `task_priority` is a
> `taskpriority_id`.

> **Scope:** starting/stopping individual timers, recurring settings, task dependencies, per-user
> notes, attachments/cover image, kanban position, bulk actions and pinning are managed in-app
> and are **not** part of this API. Checklists on a task are handled by the
> **[Checklists](checklists.md)** API (`resource_type=task`). Timers are read-only here, plus a
> stop-all command.

## The task object

```json
{
  "id": 120,
  "title": "Design homepage",
  "description": "First draft.",
  "status": { "id": 1, "title": "To do", "color": "#4f8cff" },
  "priority": { "id": 2, "title": "High", "color": "#ff5b5b" },
  "active_state": "active",
  "client_visibility": "no",
  "billable": "yes",
  "project": { "id": 7, "title": "Website Redesign" },
  "client": { "id": 12, "name": "Acme Inc" },
  "milestone": { "id": 21, "name": "Uncategorised" },
  "dates": { "start": "2026-06-01", "due": "2026-06-30", "created": "...", "updated": "..." },
  "time_logged": 3600,
  "assigned": [ { "id": 53, "name": "Sam Lee" } ],
  "tags": ["priority"]
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Task id. |
| `title` | string | |
| `description` | string\|null | |
| `status` | object | `{ id, title, color }` — `id` is the `taskstatus_id`. |
| `priority` | object | `{ id, title, color }` — `id` is the `taskpriority_id`. |
| `active_state` | string | `active` or `archived`. |
| `client_visibility` | string | `yes`/`no` (visible to the client). |
| `billable` | string | `yes`/`no`. |
| `project` | object | `{ id, title }`. |
| `client` | object | `{ id, name }` (derived from the project). |
| `milestone` | object | `{ id, name }`. |
| `dates` | object | `start`, `due`, `created`, `updated`. |
| `time_logged` | integer | Total logged time (seconds). |
| `assigned` | array | Assigned team members — set via the **assigned** action. |
| `tags` | array | Tag titles. |

---

## List / search tasks

```
GET /api/tasks
```

| Parameter | Type | Description |
|---|---|---|
| `project_id` | integer | Filter by project. |
| `client_id` | integer | Filter by client. |
| `status` | integer | Filter by `taskstatus_id`. |
| `priority` | integer | Filter by `taskpriority_id`. |
| `milestone_id` | integer | Filter by milestone. |
| `active_state` | string | `active` or `archived`. |
| `assigned_user_id` | integer | Filter by an assigned team user. |
| `search` | string | Free-text on the title. |
| `sort` | string | `task_title`, `task_date_due`, `task_created`, `task_position`. |
| `order` | string | `asc` or `desc` (default `desc`). |
| `limit` / `page` | integer | Pagination (limit max 100). |

```bash
curl -G https://your-domain/api/tasks \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d project_id=7 -d limit=25
```

```php
<?php
$query = http_build_query(['project_id' => 7, 'limit' => 25]);
$ch = curl_init("https://your-domain/api/tasks?$query");
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns `{ data, meta, message }` (`200`).

---

## Get a task

```
GET /api/tasks/{id}
```

```bash
curl https://your-domain/api/tasks/120 -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/tasks/120');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the task (`200`), message "Task retrieved successfully."

---

## Create a task

```
POST /api/tasks
```

The client is derived from the project; the milestone defaults to the project's uncategorised
milestone when omitted. Assignment is set separately via the **assigned** action.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `task_projectid` | integer | yes | Existing project id. |
| `task_title` | string | yes | |
| `task_status` | integer | yes | A `taskstatus_id`. |
| `task_priority` | integer | yes | A `taskpriority_id`. |
| `task_milestoneid` | integer | no | Must belong to the project. Defaults to uncategorised. |
| `task_date_start` / `task_date_due` | date | no | Due must be on/after start. |
| `task_description` | string | no | |
| `task_client_visibility` | string | no | `on` to make visible to the client. |
| `task_billable` | string | no | `on` to mark billable. |

```bash
curl -X POST https://your-domain/api/tasks \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d task_projectid=7 -d task_title="Design homepage" \
  -d task_status=1 -d task_priority=2
```

```php
<?php
$ch = curl_init('https://your-domain/api/tasks');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'task_projectid' => 7,
        'task_title'     => 'Design homepage',
        'task_status'    => 1,
        'task_priority'  => 2,
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the new task (`201`), message "Task created successfully."

---

## Update a task

```
PATCH /api/tasks/{id}
```

Full update of the editable fields (same parameters as **Create**, except the project is fixed).
Assignment is managed via the **assigned** action.

```bash
curl -X PATCH https://your-domain/api/tasks/120 \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d task_title="Design homepage v2" -d task_status=1 -d task_priority=2
```

```php
<?php
$ch = curl_init('https://your-domain/api/tasks/120');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PATCH',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'task_title'    => 'Design homepage v2',
        'task_status'   => 1,
        'task_priority' => 2,
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the task (`200`), message "Task updated successfully."

---

## Delete a task

```
DELETE /api/tasks/{id}
```

```bash
curl -X DELETE https://your-domain/api/tasks/120 -H "Authorization: Bearer YOUR_API_KEY"
```

Returns `{ "data": { "id": 120 }, "message": "Task deleted successfully." }`.

---

## Change status

```
PUT /api/tasks/{id}/status
```

Sets the status (and stamps the status-change date; marks the completer when set to the completed
status).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `status` | integer | yes | A `taskstatus_id`. |

```bash
curl -X PUT https://your-domain/api/tasks/120/status \
  -H "Authorization: Bearer YOUR_API_KEY" -d status=2
```

Returns the task (`200`), message "Task status updated successfully."

---

## Set assigned team members

```
PUT /api/tasks/{id}/assigned
```

Full-set replace (empty array clears).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `assigned` | array of integers | yes | Team user ids. |

```bash
curl -X PUT https://your-domain/api/tasks/120/assigned \
  -H "Authorization: Bearer YOUR_API_KEY" -d "assigned[]=53"
```

Returns the task (`200`), message "Task assignees updated successfully."

---

## Set tags

```
PUT /api/tasks/{id}/tags
```

Full-set replace (empty array clears).

```bash
curl -X PUT https://your-domain/api/tasks/120/tags \
  -H "Authorization: Bearer YOUR_API_KEY" -d "tags[]=priority"
```

---

## Archive / activate

```
PUT /api/tasks/{id}/archive
PUT /api/tasks/{id}/activate
```

No body. Toggles `active_state` between `archived` and `active`.

```bash
curl -X PUT https://your-domain/api/tasks/120/archive -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Clone a task

```
POST /api/tasks/{id}/clone
```

Clones the task into a target project + milestone.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `task_title` | string | yes | |
| `task_status` | integer | yes | A `taskstatus_id`. |
| `project_id` | integer | yes | Target project. |
| `task_milestoneid` | integer | yes | Must belong to the target project. |
| `copy_checklist` | string | no | `on` to copy checklist items. |
| `copy_files` | string | no | `on` to copy files. |

```bash
curl -X POST https://your-domain/api/tasks/120/clone \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d task_title="Design homepage (copy)" -d task_status=1 \
  -d project_id=7 -d task_milestoneid=21
```

Returns the **new** task (`201`), message "Task cloned successfully."

---

## Timers

Timers are read-only via the API, plus a stop-all command.

### List timers

```
GET /api/tasks/{id}/timers
```

Returns the task's time entries: `[{ id, user: {id,name}, started, stopped, time, status, billing_status }]`.

```bash
curl https://your-domain/api/tasks/120/timers -H "Authorization: Bearer YOUR_API_KEY"
```

### Stop all running timers

```
PUT /api/tasks/{id}/stop-timers
```

No body. Stops any running timers on the task. Returns `{ "data": { "id": 120 }, "message": "..." }`.

```bash
curl -X PUT https://your-domain/api/tasks/120/stop-timers -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Comments

### List comments

```
GET /api/tasks/{id}/comments
```

### Add a comment

```
POST /api/tasks/{id}/comments
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `comment` | string | yes | The comment text. |

```bash
curl -X POST https://your-domain/api/tasks/120/comments \
  -H "Authorization: Bearer YOUR_API_KEY" -d comment="Looks good"
```

### Delete a comment

```
DELETE /api/tasks/{id}/comments/{comment_id}
```

---

## Errors

See [Getting started](getting-started.md#errors) for the shared error format. Task-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The task (or comment) id does not exist. |
| `409 Conflict` | Invalid milestone for the project, or the project has no default milestone. |
| `422 Unprocessable Entity` | Validation failed (e.g. missing project/title/status/priority). |

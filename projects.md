---
title: Projects
description: Create, update, delete, list and search projects via the Grow CRM API.
---

# Projects

The Projects API provides CRUD access to projects, plus ancillary actions for assigning team
members and setting a manager.

> New to the API? Start with **[Getting started](getting-started.md)** (base URL, response
> envelope, errors, pagination) and **[Authentication](authentication.md)** (API keys). Those
> conventions apply to every endpoint below and are not repeated here.

> **Identifier note:** projects are addressed by their numeric **`project_id`**. Wherever `{id}`
> appears below it is the `project_id`.

## The project object

All endpoints that return a project use this shape:

```json
{
  "id": 42,
  "title": "Website Redesign",
  "description": "Full redesign of the marketing website.",
  "reference": null,
  "status": "in_progress",
  "active_state": "active",
  "progress": 40,
  "progress_manually": "no",
  "client": { "id": 12, "name": "Acme Inc" },
  "category": { "id": 3, "name": "Development" },
  "dates": {
    "start": "2026-06-01",
    "due": "2026-08-30",
    "created": "2026-06-15T09:30:00.000000Z",
    "updated": "2026-06-15T09:30:00.000000Z"
  },
  "billing": {
    "type": "hourly",
    "rate": "120.00",
    "estimated_hours": 80,
    "costs_estimate": "0.00"
  },
  "permissions": {
    "clientperm_tasks_view": "yes",
    "clientperm_tasks_collaborate": "yes",
    "clientperm_tasks_create": "yes",
    "clientperm_timesheets_view": "yes",
    "clientperm_expenses_view": "no",
    "assignedperm_tasks_collaborate": "yes"
  },
  "assigned": [ { "id": 5, "name": "Jane Doe" } ],
  "managers": [ { "id": 2, "name": "John Smith" } ],
  "tags": ["priority", "q3"]
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Project id. Used in all URLs. |
| `title` | string | Project title. |
| `description` | string\|null | Project description. |
| `reference` | string\|null | Optional external reference. |
| `status` | string | `not_started`, `in_progress`, `on_hold`, `cancelled`, `completed`. |
| `active_state` | string | `active` or `archive`. |
| `progress` | integer | Completion percentage (0–100). |
| `progress_manually` | string | `yes` if progress is set manually, otherwise `no`. |
| `client` | object | `{ id, name }`. |
| `category` | object | `{ id, name }`. |
| `dates` | object | `start`, `due`, `created`, `updated`. |
| `billing` | object | `type` (`hourly`\|`fixed`), `rate`, `estimated_hours`, `costs_estimate`. |
| `permissions` | object | Client/assignee permission flags (`yes`\|`no`). |
| `assigned` | array | Assigned team members `[{ id, name }]` — set via the **assign** action. |
| `managers` | array | Project manager(s) `[{ id, name }]` — set via the **manager** action. |
| `tags` | array | Tag titles. |

---

## List / search projects

```
GET /api/projects
```

### Query parameters

| Parameter | Type | Description |
|---|---|---|
| `status` | string | Filter by project status. |
| `client_id` | integer | Filter by client id. |
| `category_id` | integer | Filter by category id. |
| `search` | string | Free-text search on the project title. |
| `sort` | string | `project_title`, `project_status`, `project_date_start`, `project_date_due`, `project_created`, `project_progress`. |
| `order` | string | `asc` or `desc` (default `desc`). |
| `limit` | integer | Results per page (max 100). |
| `page` | integer | Page number. |

### Example request

```bash
curl -G https://your-domain/api/projects \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d status=in_progress \
  -d limit=25
```

```php
<?php
$query = http_build_query(['status' => 'in_progress', 'limit' => 25]);
$ch = curl_init("https://your-domain/api/projects?$query");
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
  "data": [ { "id": 42, "title": "Website Redesign" } ],
  "meta": { "current_page": 1, "per_page": 25, "total": 1, "last_page": 1 },
  "message": "Projects retrieved successfully."
}
```

---

## Get a project

```
GET /api/projects/{id}
```

`{id}` is the project's id.

### Example request

```bash
curl https://your-domain/api/projects/42 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/projects/42');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the project (`200`) with `message` "Project retrieved successfully."

---

## Create a project

```
POST /api/projects
```

Assignment is **not** set here — create the project, then use the **assign** and **manager**
actions below.

### Body parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `project_clientid` | integer | yes | Must be an existing client id. |
| `project_creatorid` | integer | no | Existing user id to attribute the project to. Defaults to the main administrator. |
| `project_title` | string | yes | |
| `project_date_start` | date | yes | `YYYY-MM-DD`. |
| `project_date_due` | date | no | Must be on/after the start date. |
| `project_categoryid` | integer | yes | Must be an existing category id. |
| `project_description` | string | no | |
| `project_billing_type` | string | no | `hourly` or `fixed`. |
| `project_billing_rate` | numeric | no | |
| `project_billing_estimated_hours` | numeric | no | |
| `project_billing_costs_estimate` | numeric | no | |
| `project_progress_manually` | string | no | `on` to enable manual progress. |
| `clientperm_tasks_view` | string | no | `on` to enable. |
| `clientperm_tasks_collaborate` | string | no | `on` to enable. |
| `clientperm_tasks_create` | string | no | `on` to enable. |
| `clientperm_timesheets_view` | string | no | `on` to enable. |
| `clientperm_expenses_view` | string | no | `on` to enable. |
| `assignedperm_tasks_collaborate` | string | no | `on` to enable. |

### Example request

```bash
curl -X POST https://your-domain/api/projects \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d project_clientid=12 \
  -d project_title="Website Redesign" \
  -d project_categoryid=3 \
  -d project_date_start=2026-06-01
```

```php
<?php
$ch = curl_init('https://your-domain/api/projects');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'project_clientid'   => 12,
        'project_title'      => 'Website Redesign',
        'project_categoryid' => 3,
        'project_date_start' => '2026-06-01',
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `201 Created`

```json
{
  "data": { "id": 42, "title": "Website Redesign" },
  "message": "Project created successfully."
}
```

---

## Update a project

```
PATCH /api/projects/{id}
```

`{id}` is the project's id. Same body parameters as **Create**, except
`project_clientid` cannot be changed. Assignment is managed via the **assign**/**manager**
actions, not here.

### Example request

```bash
curl -X PATCH https://your-domain/api/projects/42 \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d project_title="Website Redesign v2" \
  -d project_categoryid=3 \
  -d project_date_start=2026-06-01
```

```php
<?php
$ch = curl_init('https://your-domain/api/projects/42');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PATCH',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'project_title'      => 'Website Redesign v2',
        'project_categoryid' => 3,
        'project_date_start' => '2026-06-01',
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `200 OK`

```json
{
  "data": { "id": 42, "title": "Website Redesign v2" },
  "message": "Project updated successfully."
}
```

---

## Delete a project

```
DELETE /api/projects/{id}
```

Deletes the project and all of its related records (tasks, milestones, files, invoices,
estimates, comments, etc.).

### Example request

```bash
curl -X DELETE https://your-domain/api/projects/42 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/projects/42');
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
  "data": { "id": 42 },
  "message": "Project deleted successfully."
}
```

---

## Assign team members

```
PUT /api/projects/{id}/assign
```

Sets the project's assigned team members. The `assigned` array is the **full set** — it
replaces any existing assignment. Send an empty array to clear all assignees.

### Body parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `assigned` | array of integers | yes | Team user ids. Empty array clears all. |

### Example request

```bash
curl -X PUT https://your-domain/api/projects/42/assign \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d "assigned[]=5" \
  -d "assigned[]=8"
```

```php
<?php
$ch = curl_init('https://your-domain/api/projects/42/assign');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['assigned' => [5, 8]]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `200 OK`

```json
{
  "data": { "id": 42, "assigned": [ { "id": 5, "name": "Jane Doe" }, { "id": 8, "name": "Sam Lee" } ] },
  "message": "Project assignees updated successfully."
}
```

---

## Set the manager

```
PUT /api/projects/{id}/manager
```

Sets the project's manager (a single team user). Replaces any existing manager. Send an empty
value to remove the manager.

### Body parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `manager` | integer\|null | yes | Team user id, or empty/null to remove the manager. |

### Example request

```bash
curl -X PUT https://your-domain/api/projects/42/manager \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d manager=5
```

```php
<?php
$ch = curl_init('https://your-domain/api/projects/42/manager');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['manager' => 5]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `200 OK`

```json
{
  "data": { "id": 42, "managers": [ { "id": 5, "name": "Jane Doe" } ] },
  "message": "Project manager updated successfully."
}
```

---

## Change status

```
PUT /api/projects/{id}/status
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `status` | string | yes | `not_started`, `in_progress`, `on_hold`, `cancelled`, `completed`. |

```bash
curl -X PUT https://your-domain/api/projects/42/status \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d status=in_progress
```

```php
<?php
$ch = curl_init('https://your-domain/api/projects/42/status');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['status' => 'in_progress']),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the project (`200`) with `message` "Project status updated successfully."

---

## Update progress

```
PUT /api/projects/{id}/progress
```

Sets the project to **manual progress** mode and applies the given percentage.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `progress` | integer | yes | 0–100. |

```bash
curl -X PUT https://your-domain/api/projects/42/progress \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d progress=50
```

```php
<?php
$ch = curl_init('https://your-domain/api/projects/42/progress');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['progress' => 50]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

---

## Set tags

```
PUT /api/projects/{id}/tags
```

Sets the project's tags. The `tags` array is the **full set** — it replaces existing tags;
an empty array clears them.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `tags` | array of strings | yes | Tag titles. Empty array clears all. |

```bash
curl -X PUT https://your-domain/api/projects/42/tags \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d "tags[]=priority" -d "tags[]=q3"
```

```php
<?php
$ch = curl_init('https://your-domain/api/projects/42/tags');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['tags' => ['priority', 'q3']]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

---

## Archive / restore

```
PUT /api/projects/{id}/archive
PUT /api/projects/{id}/restore
```

No body. `archive` sets `active_state` to `archived`; `restore` sets it back to `active`.

```bash
curl -X PUT https://your-domain/api/projects/42/archive \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/projects/42/archive');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

---

## Stop timers

```
PUT /api/projects/{id}/stop-timers
```

Stops all running timers on the project. No body. Returns `{ "data": { "id": "..." }, "message": ... }`.

```bash
curl -X PUT https://your-domain/api/projects/42/stop-timers \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/projects/42/stop-timers');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

---

## Clone a project

```
POST /api/projects/{id}/clone
```

Creates a new project from an existing one. Fields not supplied default to the source project.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `project_title` | string | yes | Title of the new project. |
| `project_clientid` | integer | no | Defaults to the source project's client. |
| `project_categoryid` | integer | no | Defaults to the source project's category. |
| `project_date_start` | date | no | Defaults to the source project's start date. |
| `project_date_due` | date | no | Defaults to the source project's due date. |
| `copy_milestones` | string | no | `on` to copy milestones. |
| `copy_tasks` | string | no | `on` to copy tasks. |
| `copy_tasks_files` | string | no | `on` to copy task files. |
| `copy_tasks_checklist` | string | no | `on` to copy task checklists. |
| `copy_invoices` | string | no | `on` to copy invoices. |
| `copy_estimates` | string | no | `on` to copy estimates. |
| `copy_files` | string | no | `on` to copy project files. |

```bash
curl -X POST https://your-domain/api/projects/42/clone \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d project_title="Website Redesign (copy)" \
  -d copy_tasks=on
```

```php
<?php
$ch = curl_init('https://your-domain/api/projects/42/clone');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'project_title' => 'Website Redesign (copy)',
        'copy_tasks'    => 'on',
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the **new** project (`201`) with `message` "Project cloned successfully."

---

## Errors

See [Getting started](getting-started.md#errors) for the shared error format. Project-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The project id does not exist. |
| `422 Unprocessable Entity` | Validation failed (see below). |

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "project_title": ["The project title field is required."],
    "project_clientid": ["The selected project clientid is invalid."]
  }
}
```

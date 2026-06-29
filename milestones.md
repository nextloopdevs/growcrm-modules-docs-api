---
title: Milestones
description: Create, update, delete, list and reorder project milestones via the Grow CRM API.
---

# Milestones

The Milestones API provides CRUD access to project milestones plus bulk reordering.

> New to the API? Start with **[Getting started](getting-started.md)** (base URL, response
> envelope, errors, pagination) and **[Authentication](authentication.md)** (API keys).

> **Identifier note:** milestones are addressed by their numeric **`milestone_id`**.

> **Milestones belong to a project.** They are grouping buckets that tasks are assigned to.
> Listing requires a `project_id`. Each project has a system **`uncategorised`** default
> milestone (the catch-all for tasks without one) which cannot be deleted.

> **Scope:** the global milestone-category templates (`settings/milestones`) and web events are
> out of API scope.

## The milestone object

```json
{
  "id": 690,
  "title": "Phase 1",
  "project_id": 159,
  "type": "categorised",
  "color": "success",
  "position": 1,
  "creator_id": 1,
  "tasks_count": 4,
  "dates": { "created": "...", "updated": "..." }
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Milestone id. |
| `title` | string | |
| `project_id` | int | The owning project. |
| `type` | string | `categorised`, or `uncategorised` (the system default). |
| `color` | string | A colour key (e.g. `default`, `success`, `warning`, `danger`). |
| `position` | int | Ordering position. |
| `tasks_count` | int | Number of tasks in the milestone. |

---

## List milestones

```
GET /api/milestones?project_id={project_id}
```

`project_id` is **required**. Other query params: `type`, `search`, `sort`
(`milestone_title`,`milestone_position`,`milestone_created`), `order`, `limit`, `page`.

```bash
curl -G https://your-domain/api/milestones -H "Authorization: Bearer YOUR_API_KEY" -d project_id=159
```

---

## Get a milestone

```
GET /api/milestones/{id}
```

---

## Create a milestone

```
POST /api/milestones
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `milestone_projectid` | integer | yes | The project. |
| `milestone_title` | string | yes | Must be unique within the project. |
| `milestone_color` | string | yes | A colour key. |

```bash
curl -X POST https://your-domain/api/milestones -H "Authorization: Bearer YOUR_API_KEY" \
  -d milestone_projectid=159 -d milestone_title="Phase 1" -d milestone_color=success
```

Returns the new milestone (`201`, type `categorised`).

---

## Update a milestone

```
PATCH /api/milestones/{id}
```

Only the title and colour are editable (the project and type are immutable).

| Parameter | Type | Required |
|---|---|---|
| `milestone_title` | string | yes |
| `milestone_color` | string | yes |

---

## Delete a milestone

```
DELETE /api/milestones/{id}
```

The system `uncategorised` default cannot be deleted (`409`). By default the milestone's tasks
are reassigned to the project's default milestone; pass `delete_tasks=yes` to delete the tasks
instead (along with their attachments/comments/timers).

| Parameter | Type | Notes |
|---|---|---|
| `delete_tasks` | string | `yes` to delete the tasks; otherwise they are reassigned. |

---

## Reorder milestones

```
PUT /api/milestones/positions
```

Sets each milestone's position from the supplied ordered list of ids (1..N).

| Parameter | Type | Required |
|---|---|---|
| `milestones[]` | array of ints | yes |

```bash
curl -X PUT https://your-domain/api/milestones/positions -H "Authorization: Bearer YOUR_API_KEY" \
  -d "milestones[]=12" -d "milestones[]=10" -d "milestones[]=15"
```

---

## Errors

See [Getting started](getting-started.md#errors). Milestone-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The milestone id does not exist. |
| `409 Conflict` | Duplicate title in the project, or attempting to delete the system default. |
| `422 Unprocessable Entity` | Validation failed (missing `project_id` on list, missing fields, or an invalid project). |

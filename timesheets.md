---
title: Timesheets
description: Record, update, delete and list time entries (timesheets) via the CRM API.
---

# Timesheets

The Timesheets API provides CRUD access to time entries — time logged against a task.

> New to the API? Start with **[Getting started](/getting-started)** (base URL, response
> envelope, errors, pagination) and **[Authentication](/authentication)** (API keys).

> **Identifier note:** timesheets are addressed by their numeric **`timer_id`**.

> **A timesheet entry logs time against a task.** You supply the `task_id`, the `user_id` who
> performed the work, and the duration (`hours` + `minutes`); the project and client are derived
> from the task. The duration must total at least one minute.

> **Scope:** live timer **start/stop** is on the [Tasks](/tasks) API (task-timers list +
> stop-all). Bulk delete, pinning, settings and the stats widget are out of scope. The
> **billing status** is managed by invoicing, not editable here.

## The timesheet object

```json
{
  "id": 27,
  "task_id": 43,
  "project_id": 44,
  "client_id": 20,
  "worked_by_id": 5,
  "recorded_by_id": 1,
  "time_seconds": 5400,
  "time_readable": "1h 30m",
  "status": "stopped",
  "billing": { "status": "not_invoiced", "invoice_id": null },
  "dates": { "created": "2026-06-20", "started": null, "stopped": null }
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Timer/timesheet id. |
| `task_id` | int | The task the time is logged against. |
| `project_id` / `client_id` | int | Derived from the task. |
| `worked_by_id` | int | The user who performed the work. |
| `recorded_by_id` | int | The user who recorded the entry (the API admin). |
| `time_seconds` | int | Duration in seconds. |
| `time_readable` | string | Human-readable duration (e.g. `1h 30m`). |
| `status` | string | `stopped` (manual entries) or `running`. |
| `billing` | object | `status` (`not_invoiced`/`invoiced`) + `invoice_id`. Read-only. |

---

## List / search timesheets

```
GET /api/timesheets
```

Query params: `task_id`, `project_id`, `client_id`, `user_id` (worked-by), `status`,
`billing_status`, `date_start`, `date_end`, `sort` (`timer_created`,`timer_time`), `order`,
`limit`, `page`.

```bash
curl -G https://your-domain/api/timesheets -H "Authorization: Bearer YOUR_API_KEY" -d project_id=44
```

---

## Get a timesheet

```
GET /api/timesheets/{id}
```

---

## Record a timesheet entry

```
POST /api/timesheets
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `task_id` | integer | yes | The task (project/client derived from it). |
| `user_id` | integer | yes | The user who performed the work. |
| `hours` | numeric | yes | |
| `minutes` | numeric | yes | |
| `timer_created` | date | yes | The date the work was done. |

The combined `hours` + `minutes` must total at least one minute.

```bash
curl -X POST https://your-domain/api/timesheets -H "Authorization: Bearer YOUR_API_KEY" \
  -d task_id=43 -d user_id=5 -d hours=1 -d minutes=30 -d timer_created=2026-06-20
```

Returns the new entry (`201`, status `stopped`).

---

## Update a timesheet entry

```
PATCH /api/timesheets/{id}
```

Edits the logged time only (the task, date and user are immutable).

| Parameter | Type | Required |
|---|---|---|
| `hours` | numeric | yes |
| `minutes` | numeric | yes |

```bash
curl -X PATCH https://your-domain/api/timesheets/27 -H "Authorization: Bearer YOUR_API_KEY" \
  -d hours=2 -d minutes=0
```

---

## Delete a timesheet entry

```
DELETE /api/timesheets/{id}
```

---

## Errors

See [Getting started](/getting-started#errors). Timesheet-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The timesheet (or task) id does not exist. |
| `422 Unprocessable Entity` | Validation failed (missing task/user/date, or a total time under one minute). |

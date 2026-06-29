---
title: Expenses
description: Create, update, delete, list, tag, attach and clone expenses via the Grow CRM API.
---

# Expenses

The Expenses API provides CRUD access to expenses, tags, the attach/detach action and clone.

> New to the API? Start with **[Getting started](getting-started.md)** (base URL, response
> envelope, errors, pagination) and **[Authentication](authentication.md)** (API keys).

> **Identifier note:** expenses are addressed by their numeric **`expense_id`**.

> **Project & client are both optional and independent.** When a project is linked, the client
> is forced to that project's client; a conflicting client is rejected (`422`).

> **Scope:** file attachments (upload/download), recurring schedules, per-user sticky notes &
> pinning, bulk change-category and invoice-integration are out of API scope. Existing
> attachments are returned read-only.

## The expense object

```json
{
  "id": 59,
  "description": "Server hosting",
  "date": "2026-06-20",
  "amount": "200.00",
  "type": null,
  "billable": "billable",
  "billing_status": "not_invoiced",
  "category": { "id": 7, "name": "Default" },
  "client": { "id": 4, "name": "Acme Inc" },
  "project_id": 191,
  "creator_id": 1,
  "dates": { "date": "2026-06-20", "created": "..." },
  "recurring": { "is_recurring": false },
  "attachments": [ { "uniqueid": "abc123", "filename": "receipt.pdf" } ],
  "tags": ["travel"]
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Expense id. |
| `description` / `date` / `amount` | string/date/numeric | Core fields. |
| `billable` | string | `billable` or `not_billable`. |
| `billing_status` | string | `not_invoiced` or `invoiced`. Read-only. |
| `category` | object | The expense category (`id`, `name`). |
| `client` / `project_id` | object/int | Optional links (kept consistent). |
| `creator_id` | int | The API admin. |
| `recurring` | object | `is_recurring`. Read-only (recurring is out of scope). |
| `attachments` | array | Existing attachments, read-only. |
| `tags` | array | Tag titles. |

---

## List / search expenses

```
GET /api/expenses
```

Query params: `category_id`, `client_id`, `project_id`, `billable` (`billable`/`not_billable`),
`billing_status`, `amount_min`, `amount_max`, `date_start`, `date_end`, `search`, `tags[]`,
`sort` (`expense_date`,`expense_amount`,`expense_created`), `order`, `limit`, `page`.

```bash
curl -G https://your-domain/api/expenses -H "Authorization: Bearer YOUR_API_KEY" -d category_id=7
```

---

## Get an expense

```
GET /api/expenses/{id}
```

---

## Create an expense

```
POST /api/expenses
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `expense_description` | string | yes | |
| `expense_date` | date | yes | |
| `expense_amount` | numeric | yes | |
| `expense_categoryid` | integer | yes | Must be a category of type **expense**. |
| `expense_projectid` | integer | no | When set, the client is forced to the project's client. |
| `expense_clientid` | integer | no | Must belong to the linked project (if any). |
| `expense_billable` | boolean | no | `yes`/`true` marks the expense billable. |

```bash
curl -X POST https://your-domain/api/expenses -H "Authorization: Bearer YOUR_API_KEY" \
  -d expense_description="Server hosting" -d expense_date=2026-06-20 -d expense_amount=200 \
  -d expense_categoryid=7 -d expense_projectid=191 -d expense_billable=yes
```

Returns the new expense (`201`).

---

## Update an expense

```
PATCH /api/expenses/{id}
```

Same fields as create. The `billable` flag is not changed once the expense has been
`invoiced`. Project/client consistency is enforced.

```bash
curl -X PATCH https://your-domain/api/expenses/59 -H "Authorization: Bearer YOUR_API_KEY" \
  -d expense_description="Server hosting" -d expense_date=2026-06-20 -d expense_amount=200 \
  -d expense_categoryid=7
```

---

## Delete an expense

```
DELETE /api/expenses/{id}
```

---

## Set tags

```
PUT /api/expenses/{id}/tags
```

Full-set replace (empty clears).

---

## Attach / detach (project & client)

```
PUT /api/expenses/{id}/attach
```

Links the expense to a project and/or client. Either or both may be supplied; an empty value
detaches. When a project is supplied the client is forced to the project's client, and a
conflicting client is rejected (`422`).

| Parameter | Type | Notes |
|---|---|---|
| `expense_projectid` | integer | Empty to detach. |
| `expense_clientid` | integer | Empty to detach. Must belong to the linked project. |

```bash
# attach to a project (client auto-derived)
curl -X PUT https://your-domain/api/expenses/59/attach -H "Authorization: Bearer YOUR_API_KEY" \
  -d expense_projectid=191

# detach both
curl -X PUT https://your-domain/api/expenses/59/attach -H "Authorization: Bearer YOUR_API_KEY" \
  -d expense_projectid= -d expense_clientid=
```

---

## Clone

```
POST /api/expenses/{id}/clone
```

Copies the source expense into a new one (`billing_status` reset to `not_invoiced`; recurring
schedules and tags are not copied). `expense_date` is required; the other fields optionally
override the source.

| Parameter | Type | Required |
|---|---|---|
| `expense_date` | date | yes |
| `expense_description` / `expense_amount` / `expense_categoryid` | mixed | no |
| `expense_projectid` / `expense_clientid` | integer | no |

Returns the **new** expense (`201`).

---

## Errors

See [Getting started](getting-started.md#errors). Expense-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The expense id does not exist. |
| `422 Unprocessable Entity` | Validation failed (missing fields, non-expense category, or a client that does not belong to the linked project). |

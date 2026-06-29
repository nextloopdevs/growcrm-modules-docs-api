---
title: Leads
description: Create, update, delete, list, convert, clone and manage leads via the Grow CRM API.
---

# Leads

The Leads API provides CRUD access to leads, plus ancillary actions for status, assignment,
tags, archive/activate, convert-to-client, clone and comments.

> New to the API? Start with **[Getting started](getting-started.md)** (base URL, response
> envelope, errors, pagination) and **[Authentication](authentication.md)** (API keys). Those
> conventions apply to every endpoint below and are not repeated here.

> **Identifier note:** leads are addressed by their numeric **`lead_id`**. Wherever `{id}`
> appears below it is the `lead_id`. A lead's status is a **`leadstatus_id`** (from the lead
> statuses configured in the CRM).

> **Scope:** attachments/cover image, per-user notes, activity logs, kanban position, bulk
> actions and pinning are managed in-app and are **not** part of this API. Checklists on a lead
> are handled by the **[Checklists](checklists.md)** API (`resource_type=lead`).

## The lead object

```json
{
  "id": 59,
  "title": "Acme – new website",
  "value": "2500.00",
  "status": { "id": 1, "title": "New", "color": "#4f8cff" },
  "active_state": "active",
  "category": { "id": 3, "name": "Web" },
  "source": "Website",
  "last_contacted": "2026-06-15",
  "description": "Inbound enquiry.",
  "contact": {
    "first_name": "Jane", "last_name": "Doe", "email": "jane@acme.example",
    "phone": "+1 555 0100", "job_position": "CTO", "company_name": "Acme Inc",
    "website": "https://acme.example"
  },
  "address": { "street": null, "city": null, "state": null, "zip": null, "country": null },
  "converted": { "is_converted": false, "client_id": null, "date": null },
  "assigned": [ { "id": 53, "name": "Sam Lee" } ],
  "tags": ["priority"],
  "dates": { "created": "2026-06-18T09:30:00.000000Z", "updated": "2026-06-18T09:30:00.000000Z" }
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Lead id. Used in all URLs. |
| `title` | string | Lead title. |
| `value` | string\|null | Estimated value. |
| `status` | object | `{ id, title, color }` — `id` is the `leadstatus_id`. |
| `active_state` | string | `active` or `archived`. |
| `category` | object | `{ id, name }`. |
| `source` | string\|null | Lead source. |
| `last_contacted` | date\|null | |
| `description` | string\|null | |
| `contact` | object | Contact person + company fields. |
| `address` | object | Address fields. |
| `converted` | object | Read-only: `is_converted`, `client_id`, `date`. |
| `assigned` | array | Assigned team members `[{ id, name }]` — set via the **assigned** action. |
| `tags` | array | Tag titles. |
| `dates` | object | `created`, `updated`. |

---

## List / search leads

```
GET /api/leads
```

### Query parameters

| Parameter | Type | Description |
|---|---|---|
| `status` | integer | Filter by `leadstatus_id`. |
| `category_id` | integer | Filter by category id. |
| `source` | string | Filter by source. |
| `active_state` | string | `active` or `archived`. |
| `assigned_user_id` | integer | Filter by an assigned team user. |
| `search` | string | Free-text on title / company / email. |
| `sort` | string | `lead_title`, `lead_value`, `lead_created`, `lead_last_contacted`. |
| `order` | string | `asc` or `desc` (default `desc`). |
| `limit` | integer | Results per page (max 100). |
| `page` | integer | Page number. |

### Example request

```bash
curl -G https://your-domain/api/leads \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d status=1 -d limit=25
```

```php
<?php
$query = http_build_query(['status' => 1, 'limit' => 25]);
$ch = curl_init("https://your-domain/api/leads?$query");
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
  "data": [ { "id": 59, "title": "Acme – new website", "status": { "id": 1, "title": "New" } } ],
  "meta": { "current_page": 1, "per_page": 25, "total": 1, "last_page": 1 },
  "message": "Leads retrieved successfully."
}
```

---

## Get a lead

```
GET /api/leads/{id}
```

```bash
curl https://your-domain/api/leads/59 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/leads/59');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the lead (`200`) with `message` "Lead retrieved successfully."

---

## Create a lead

```
POST /api/leads
```

### Body parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `lead_title` | string | yes | |
| `lead_status` | integer | yes | A `leadstatus_id`. |
| `lead_categoryid` | integer | yes | Existing (lead) category id. |
| `lead_firstname` / `lead_lastname` | string | no | Contact name. |
| `lead_email` | email | no | |
| `lead_phone`, `lead_company_name`, `lead_job_position`, `lead_website` | string | no | |
| `lead_value` | numeric | no | |
| `lead_source` | string | no | |
| `lead_last_contacted` | date | no | |
| `lead_description` | string | no | |
| `lead_street`, `lead_city`, `lead_state`, `lead_zip`, `lead_country` | string | no | Address. |

> Required client **custom fields** of type `leads` must also be supplied (by field name).
> Assignment is set separately — see the **assigned** action.

### Example request

```bash
curl -X POST https://your-domain/api/leads \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d lead_title="Acme – new website" \
  -d lead_status=1 \
  -d lead_categoryid=3
```

```php
<?php
$ch = curl_init('https://your-domain/api/leads');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'lead_title'      => 'Acme – new website',
        'lead_status'     => 1,
        'lead_categoryid' => 3,
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `201 Created`

```json
{
  "data": { "id": 59, "title": "Acme – new website", "status": { "id": 1 } },
  "message": "Lead created successfully."
}
```

---

## Update a lead

```
PATCH /api/leads/{id}
```

Full update of the editable fields — send all fields you want to keep (same parameters as
**Create**). Assignment is managed via the **assigned** action, not here.

```bash
curl -X PATCH https://your-domain/api/leads/59 \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d lead_title="Acme – website + branding" \
  -d lead_status=1 \
  -d lead_categoryid=3
```

```php
<?php
$ch = curl_init('https://your-domain/api/leads/59');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PATCH',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'lead_title'      => 'Acme – website + branding',
        'lead_status'     => 1,
        'lead_categoryid' => 3,
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the lead (`200`) with `message` "Lead updated successfully."

---

## Delete a lead

```
DELETE /api/leads/{id}
```

Deletes the lead and its related records.

```bash
curl -X DELETE https://your-domain/api/leads/59 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Example response — `200 OK`

```json
{ "data": { "id": 59 }, "message": "Lead deleted successfully." }
```

---

## Change status

```
PUT /api/leads/{id}/status
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `status` | integer | yes | A `leadstatus_id`. |

```bash
curl -X PUT https://your-domain/api/leads/59/status \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d status=3
```

Returns the lead (`200`) with `message` "Lead status updated successfully."

---

## Set assigned team members

```
PUT /api/leads/{id}/assigned
```

The `assigned` array is the **full set** — it replaces any existing assignment; an empty array
clears all.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `assigned` | array of integers | yes | Team user ids. Empty array clears all. |

```bash
curl -X PUT https://your-domain/api/leads/59/assigned \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d "assigned[]=53" -d "assigned[]=8"
```

Returns the lead (`200`) with `message` "Lead assignees updated successfully."

---

## Set tags

```
PUT /api/leads/{id}/tags
```

Full-set replace (empty array clears).

```bash
curl -X PUT https://your-domain/api/leads/59/tags \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d "tags[]=priority"
```

Returns the lead (`200`) with `message` "Lead tags updated successfully."

---

## Archive / activate

```
PUT /api/leads/{id}/archive
PUT /api/leads/{id}/activate
```

No body. `archive` sets `active_state` to `archived`; `activate` sets it back to `active`.

```bash
curl -X PUT https://your-domain/api/leads/59/archive \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Convert to client

```
POST /api/leads/{id}/convert
```

Creates a **client** (and its primary user) from the lead. The lead's proposals are moved to the
new client. By default the lead is kept and marked converted; pass `delete_lead=yes` to remove
it. A welcome email is sent only if `send_welcome_email=yes`. Returns the **new client**.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `first_name` | string | yes | Primary user's first name. |
| `last_name` | string | yes | Primary user's last name. |
| `email` | email | yes | Unique among client/team users. |
| `client_company_name` | string | no | |
| `client_billing_invoice_due_days` | integer | no | |
| `send_welcome_email` | string | no | `yes` to email the new client. Default `no`. |
| `delete_lead` | string | no | `yes` to delete the lead after converting. Default `no`. |

```bash
curl -X POST https://your-domain/api/leads/59/convert \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d first_name=Jane -d last_name=Doe -d email=jane@acme.example \
  -d client_company_name="Acme Inc"
```

```php
<?php
$ch = curl_init('https://your-domain/api/leads/59/convert');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'first_name'          => 'Jane',
        'last_name'           => 'Doe',
        'email'               => 'jane@acme.example',
        'client_company_name' => 'Acme Inc',
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the **new client** (`201`) with `message` "Lead converted to client successfully."

---

## Clone a lead

```
POST /api/leads/{id}/clone
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `lead_title` | string | yes | Title of the new lead. |
| `lead_status` | integer | yes | A `leadstatus_id`. |
| `lead_firstname`, `lead_lastname`, `lead_email`, `lead_phone`, `lead_company_name`, `lead_website` | string | no | |
| `lead_value` | numeric | no | |
| `copy_checklist` | string | no | `on` to copy checklist items. |
| `copy_files` | string | no | `on` to copy files. |

```bash
curl -X POST https://your-domain/api/leads/59/clone \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d lead_title="Acme – new website (copy)" -d lead_status=1
```

Returns the **new** lead (`201`) with `message` "Lead cloned successfully."

---

## Comments

### List comments

```
GET /api/leads/{id}/comments
```

Returns `{ "data": [ { "id": 9, "text": "...", "creator": { "id": 1, "name": "Admin" } } ], "message": "..." }`.

### Add a comment

```
POST /api/leads/{id}/comments
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `comment` | string | yes | The comment text. |

```bash
curl -X POST https://your-domain/api/leads/59/comments \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d comment="Called the prospect, following up next week"
```

Returns the comment (`201`) with `message` "Lead comment added successfully."

### Delete a comment

```
DELETE /api/leads/{id}/comments/{comment_id}
```

Returns `{ "data": { "id": 9 }, "message": "Lead comment deleted successfully." }`.

---

## Errors

See [Getting started](getting-started.md#errors) for the shared error format. Lead-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The lead (or comment) id does not exist. |
| `409 Conflict` | The lead has already been converted to a client. |
| `422 Unprocessable Entity` | Validation failed (e.g. missing title/status/category, or convert email already taken). |

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "lead_title": ["The lead title field is required."],
    "lead_status": ["The selected lead status is invalid."]
  }
}
```

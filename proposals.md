---
title: Proposals
description: Create, update, delete, list, clone and manage proposals via the CRM API.
---

# Proposals

The Proposals API provides CRUD access to proposals (documents sent to a client or a lead), plus
ancillary actions for status, tags and cloning.

> New to the API? Start with **[Getting started](/getting-started)** (base URL, response
> envelope, errors, pagination) and **[Authentication](/authentication)** (API keys). Those
> conventions apply to every endpoint below and are not repeated here.

> **Identifier note:** proposals are addressed by their numeric **`doc_id`**. Wherever `{id}`
> appears below it is the `doc_id`.

> **Scope:** the signature accept flow, publishing & emailing, automation, estimate generation
> and the rich document editor are managed in-app and are **not** part of this API. Client accept
> details are exposed read-only; a proposal can still be marked declined/accepted via the status
> action.

## The proposal object

A proposal belongs to a **client** or a **lead** — the unused side is `null`.

```json
{
  "id": 210,
  "title": "Website Proposal",
  "status": "draft",
  "body": "<p>Proposal details…</p>",
  "client": { "id": 12, "name": "Acme Inc" },
  "lead": { "id": null, "name": null },
  "category": { "id": 11, "name": "Standard" },
  "project_id": null,
  "dates": {
    "start": "2026-06-01",
    "end": "2026-12-31",
    "created": "2026-06-18T09:30:00.000000Z",
    "published": null
  },
  "signed": { "date": null, "first_name": null, "last_name": null },
  "tags": ["priority"]
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Proposal id. Used in all URLs. |
| `title` | string | Proposal title. |
| `status` | string | `draft`, `new`, `accepted`, `declined`, `revised`. |
| `body` | string\|null | Proposal body (HTML). |
| `client` | object | `{ id, name }` (null when the proposal is for a lead). |
| `lead` | object | `{ id, name }` (null when the proposal is for a client). |
| `category` | object | `{ id, name }`. |
| `project_id` | integer\|null | Attached project id (managed in-app). |
| `dates` | object | `start`, `end`, `created`, `published`. |
| `signed` | object | Read-only client accept details: `date`, `first_name`, `last_name`. |
| `tags` | array | Tag titles. |

---

## List / search proposals

```
GET /api/proposals
```

### Query parameters

| Parameter | Type | Description |
|---|---|---|
| `status` | string | Filter by status. |
| `client_id` | integer | Filter by client id. |
| `lead_id` | integer | Filter by lead id. |
| `category_id` | integer | Filter by category id. |
| `search` | string | Free-text search on the title. |
| `sort` | string | `doc_title`, `doc_status`, `doc_date_start`, `doc_created`. |
| `order` | string | `asc` or `desc` (default `desc`). |
| `limit` | integer | Results per page (max 100). |
| `page` | integer | Page number. |

### Example request

```bash
curl -G https://your-domain/api/proposals \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d status=draft -d limit=25
```

```php
<?php
$query = http_build_query(['status' => 'draft', 'limit' => 25]);
$ch = curl_init("https://your-domain/api/proposals?$query");
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
  "data": [ { "id": 210, "title": "Website Proposal", "status": "draft" } ],
  "meta": { "current_page": 1, "per_page": 25, "total": 1, "last_page": 1 },
  "message": "Proposals retrieved successfully."
}
```

---

## Get a proposal

```
GET /api/proposals/{id}
```

```bash
curl https://your-domain/api/proposals/210 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/proposals/210');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the proposal (`200`) with `message` "Proposal retrieved successfully."

---

## Create a proposal

```
POST /api/proposals
```

A proposal is created for a **client** or a **lead** (`customer_type`). Provide `doc_body` to set
the content directly, or `proposal_template` to seed the body from a template. Creates a backing
estimate record.

### Body parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `customer_type` | string | yes | `client` or `lead`. |
| `doc_client_id` | integer | required if `customer_type=client` | Existing client id. |
| `doc_lead_id` | integer | required if `customer_type=lead` | Existing lead id. |
| `doc_title` | string | yes | |
| `doc_date_start` | date | yes | `YYYY-MM-DD`. |
| `doc_categoryid` | integer | yes | Existing (proposal) category id. |
| `doc_date_end` | date | no | Must be on/after the start date. |
| `doc_body` | string (HTML) | no | Proposal content. Overrides `proposal_template`. |
| `proposal_template` | integer | no | Template id used to seed the body when `doc_body` is omitted. |

### Example request

```bash
curl -X POST https://your-domain/api/proposals \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d customer_type=client \
  -d doc_client_id=12 \
  -d doc_title="Website Proposal" \
  -d doc_date_start=2026-06-01 \
  -d doc_categoryid=11
```

```php
<?php
$ch = curl_init('https://your-domain/api/proposals');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'customer_type'  => 'client',
        'doc_client_id'  => 12,
        'doc_title'      => 'Website Proposal',
        'doc_date_start' => '2026-06-01',
        'doc_categoryid' => 11,
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `201 Created`

```json
{
  "data": { "id": 210, "title": "Website Proposal", "status": "draft" },
  "message": "Proposal created successfully."
}
```

---

## Update a proposal

```
PATCH /api/proposals/{id}
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `doc_title` | string | yes | |
| `doc_date_start` | date | yes | |
| `doc_categoryid` | integer | yes | Existing category id. |
| `doc_date_end` | date | no | Must be on/after the start date. |
| `doc_body` | string (HTML) | no | Replaces the proposal content. |

```bash
curl -X PATCH https://your-domain/api/proposals/210 \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d doc_title="Website Proposal v2" \
  -d doc_date_start=2026-06-01 \
  -d doc_categoryid=11
```

```php
<?php
$ch = curl_init('https://your-domain/api/proposals/210');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PATCH',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'doc_title'      => 'Website Proposal v2',
        'doc_date_start' => '2026-06-01',
        'doc_categoryid' => 11,
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the proposal (`200`) with `message` "Proposal updated successfully."

---

## Delete a proposal

```
DELETE /api/proposals/{id}
```

```bash
curl -X DELETE https://your-domain/api/proposals/210 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Example response — `200 OK`

```json
{
  "data": { "id": 210 },
  "message": "Proposal deleted successfully."
}
```

---

## Change status

```
PUT /api/proposals/{id}/status
```

Changing the status clears any recorded client acceptance signature.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `status` | string | yes | `accepted`, `declined`, `revised`, `draft`, `new`. |

```bash
curl -X PUT https://your-domain/api/proposals/210/status \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d status=accepted
```

```php
<?php
$ch = curl_init('https://your-domain/api/proposals/210/status');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['status' => 'accepted']),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the proposal (`200`) with `message` "Proposal status updated successfully."

---

## Set tags

```
PUT /api/proposals/{id}/tags
```

The `tags` array is the **full set** — it replaces existing tags; an empty array clears them.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `tags` | array of strings | yes | Tag titles. Empty array clears all. |

```bash
curl -X PUT https://your-domain/api/proposals/210/tags \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d "tags[]=priority"
```

```php
<?php
$ch = curl_init('https://your-domain/api/proposals/210/tags');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['tags' => ['priority']]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the proposal (`200`) with `message` "Proposal tags updated successfully."

---

## Clone a proposal

```
POST /api/proposals/{id}/clone
```

Creates a new proposal from an existing one.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `doc_title` | string | yes | Title of the new proposal. |
| `customer_type` | string | yes | `client` or `lead`. |
| `doc_client_id` | integer | required if `customer_type=client` | |
| `doc_lead_id` | integer | required if `customer_type=lead` | |
| `doc_categoryid` | integer | yes | |
| `doc_date_start` | date | yes | |
| `doc_date_end` | date | no | |

```bash
curl -X POST https://your-domain/api/proposals/210/clone \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d doc_title="Website Proposal (copy)" \
  -d customer_type=client \
  -d doc_client_id=12 \
  -d doc_categoryid=11 \
  -d doc_date_start=2026-06-01
```

```php
<?php
$ch = curl_init('https://your-domain/api/proposals/210/clone');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'doc_title'      => 'Website Proposal (copy)',
        'customer_type'  => 'client',
        'doc_client_id'  => 12,
        'doc_categoryid' => 11,
        'doc_date_start' => '2026-06-01',
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the **new** proposal (`201`) with `message` "Proposal cloned successfully."

---

## Errors

See [Getting started](/getting-started#errors) for the shared error format. Proposal-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The proposal id does not exist. |
| `422 Unprocessable Entity` | Validation failed (e.g. missing customer/title/date/category, or an invalid status). |

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "customer_type": ["The customer type field is required."],
    "doc_title": ["The doc title field is required."]
  }
}
```

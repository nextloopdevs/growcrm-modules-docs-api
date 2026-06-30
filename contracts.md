---
title: Contracts
description: Create, update, delete, list, clone and manage contracts via the CRM
---

# Contracts

The Contracts API provides CRUD access to contracts (signable documents), plus ancillary actions
for tags and cloning.

> New to the API? Start with **[Getting started](/getting-started)** (base URL, response
> envelope, errors, pagination) and **[Authentication](/authentication)** (API keys). Those
> conventions apply to every endpoint below and are not repeated here.

> **Identifier note:** contracts are addressed by their numeric **`doc_id`**. Wherever `{id}`
> appears below it is the `doc_id`.

> **Scope:** signing (team/client/guest/in-person), publishing & emailing, automation, estimate
> generation and the rich document editor are managed in-app and are **not** part of this API.
> Signature/automation state is exposed read-only where relevant.

## The contract object

```json
{
  "id": 136,
  "title": "Service Agreement",
  "status": "draft",
  "value": "5000.00",
  "body": "<p>Agreement terms…</p>",
  "client": { "id": 12, "name": "Acme Inc" },
  "category": { "id": 6, "name": "Standard" },
  "project_id": null,
  "dates": {
    "start": "2026-06-01",
    "end": "2026-12-31",
    "created": "2026-06-17T09:30:00.000000Z",
    "published": null
  },
  "signing": { "client_status": "unsigned", "provider_status": "unsigned" },
  "tags": ["priority"]
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Contract id. Used in all URLs. |
| `title` | string | Contract title. |
| `status` | string | **Read-only**, system-derived: `draft`, `awaiting_signatures`, `active`, `expired` (driven by signatures + dates). |
| `value` | string\|null | Contract value. |
| `body` | string\|null | Contract body (HTML). |
| `client` | object | `{ id, name }`. |
| `category` | object | `{ id, name }`. |
| `project_id` | integer\|null | Attached project id (managed in-app). |
| `dates` | object | `start`, `end`, `created`, `published`. |
| `signing` | object | Read-only signing state: `client_status`, `provider_status`. |
| `tags` | array | Tag titles. |

---

## List / search contracts

```
GET /api/contracts
```

### Query parameters

| Parameter | Type | Description |
|---|---|---|
| `status` | string | Filter by status. |
| `client_id` | integer | Filter by client id. |
| `category_id` | integer | Filter by category id. |
| `search` | string | Free-text search on the title. |
| `sort` | string | `doc_title`, `doc_status`, `doc_date_start`, `doc_created`, `doc_value`. |
| `order` | string | `asc` or `desc` (default `desc`). |
| `limit` | integer | Results per page (max 100). |
| `page` | integer | Page number. |

### Example request

```bash
curl -G https://your-domain/api/contracts \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d status=draft -d limit=25
```

```php
<?php
$query = http_build_query(['status' => 'draft', 'limit' => 25]);
$ch = curl_init("https://your-domain/api/contracts?$query");
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
  "data": [ { "id": 136, "title": "Service Agreement", "status": "draft" } ],
  "meta": { "current_page": 1, "per_page": 25, "total": 1, "last_page": 1 },
  "message": "Contracts retrieved successfully."
}
```

---

## Get a contract

```
GET /api/contracts/{id}
```

```bash
curl https://your-domain/api/contracts/136 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/contracts/136');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the contract (`200`) with `message` "Contract retrieved successfully."

---

## Create a contract

```
POST /api/contracts
```

Creates a contract and its backing estimate record. Provide `doc_body` to set the content
directly, or `contract_template` to seed the body from a template. (Automation is not
configurable via the API; new contracts are created with automation disabled.)

### Body parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `doc_client_id` | integer | yes | Existing client id. |
| `doc_title` | string | yes | |
| `doc_date_start` | date | yes | `YYYY-MM-DD`. |
| `doc_categoryid` | integer | yes | Existing (contract) category id. |
| `doc_date_end` | date | no | Must be on/after the start date. |
| `doc_value` | numeric | no | |
| `doc_project_id` | integer | no | Existing project id. |
| `doc_lead_id` | integer | no | Existing lead id. |
| `doc_body` | string (HTML) | no | Contract content. Overrides `contract_template`. |
| `contract_template` | integer | no | Template id used to seed the body when `doc_body` is omitted. |

### Example request

```bash
curl -X POST https://your-domain/api/contracts \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d doc_client_id=12 \
  -d doc_title="Service Agreement" \
  -d doc_date_start=2026-06-01 \
  -d doc_categoryid=6
```

```php
<?php
$ch = curl_init('https://your-domain/api/contracts');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'doc_client_id'  => 12,
        'doc_title'      => 'Service Agreement',
        'doc_date_start' => '2026-06-01',
        'doc_categoryid' => 6,
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `201 Created`

```json
{
  "data": { "id": 136, "title": "Service Agreement", "status": "draft" },
  "message": "Contract created successfully."
}
```

---

## Update a contract

```
PATCH /api/contracts/{id}
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `doc_title` | string | yes | |
| `doc_date_start` | date | yes | |
| `doc_categoryid` | integer | yes | Existing category id. |
| `doc_date_end` | date | no | Must be on/after the start date. |
| `doc_value` | numeric | no | |
| `doc_body` | string (HTML) | no | Replaces the contract content. |

```bash
curl -X PATCH https://your-domain/api/contracts/136 \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d doc_title="Service Agreement v2" \
  -d doc_date_start=2026-06-01 \
  -d doc_categoryid=6
```

```php
<?php
$ch = curl_init('https://your-domain/api/contracts/136');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PATCH',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'doc_title'      => 'Service Agreement v2',
        'doc_date_start' => '2026-06-01',
        'doc_categoryid' => 6,
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the contract (`200`) with `message` "Contract updated successfully."

---

## Delete a contract

```
DELETE /api/contracts/{id}
```

```bash
curl -X DELETE https://your-domain/api/contracts/136 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Example response — `200 OK`

```json
{
  "data": { "id": 136 },
  "message": "Contract deleted successfully."
}
```

---

## Set tags

```
PUT /api/contracts/{id}/tags
```

The `tags` array is the **full set** — it replaces existing tags; an empty array clears them.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `tags` | array of strings | yes | Tag titles. Empty array clears all. |

```bash
curl -X PUT https://your-domain/api/contracts/136/tags \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d "tags[]=priority"
```

```php
<?php
$ch = curl_init('https://your-domain/api/contracts/136/tags');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['tags' => ['priority']]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the contract (`200`) with `message` "Contract tags updated successfully."

---

## Clone a contract

```
POST /api/contracts/{id}/clone
```

Creates a new contract from an existing one.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `doc_title` | string | yes | Title of the new contract. |
| `doc_client_id` | integer | yes | Client for the new contract. |
| `doc_categoryid` | integer | yes | Category for the new contract. |
| `doc_date_start` | date | yes | |
| `doc_date_end` | date | no | |
| `doc_project_id` | integer | no | |
| `doc_value` | numeric | no | |

```bash
curl -X POST https://your-domain/api/contracts/136/clone \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d doc_title="Service Agreement (copy)" \
  -d doc_client_id=12 \
  -d doc_categoryid=6 \
  -d doc_date_start=2026-06-01
```

```php
<?php
$ch = curl_init('https://your-domain/api/contracts/136/clone');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'doc_title'      => 'Service Agreement (copy)',
        'doc_client_id'  => 12,
        'doc_categoryid' => 6,
        'doc_date_start' => '2026-06-01',
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the **new** contract (`201`) with `message` "Contract cloned successfully."

---

## Errors

See [Getting started](/getting-started#errors) for the shared error format. Contract-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The contract id does not exist. |
| `422 Unprocessable Entity` | Validation failed (e.g. missing client/title/date/category, or an invalid status). |

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "doc_client_id": ["The selected doc client id is invalid."],
    "doc_title": ["The doc title field is required."]
  }
}
```

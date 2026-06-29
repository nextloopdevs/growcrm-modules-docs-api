---
title: Clients
description: Create, update, delete, list and search clients via the Grow CRM API.
---

# Clients

The Clients API provides CRUD access to clients, plus ancillary actions for setting tags,
changing the status, and changing the account owner.

> New to the API? Start with **[Getting started](getting-started.md)** (base URL, response
> envelope, errors, pagination) and **[Authentication](authentication.md)** (API keys). Those
> conventions apply to every endpoint below and are not repeated here.

> **Identifier note:** unlike most resources, clients are addressed by their **numeric
> `client_id`** (the clients table has no unique-id column). Wherever `{id}` appears below it is
> the `client_id`.

## The client object

All endpoints that return a client use this shape:

```json
{
  "id": 12,
  "company_name": "Acme Inc",
  "description": "Key account.",
  "status": "active",
  "phone": "+1 555 0100",
  "website": "https://acme.example",
  "vat": "GB123456789",
  "category": { "id": 2, "name": "Standard" },
  "owner": { "id": 34, "name": "Jane Doe" },
  "billing": {
    "street": "1 Market St",
    "city": "San Francisco",
    "state": "CA",
    "zip": "94105",
    "country": "United States",
    "invoice_due_days": "7"
  },
  "shipping": {
    "street": null,
    "city": null,
    "state": null,
    "zip": null,
    "country": null
  },
  "dates": {
    "created": "2026-06-15T09:30:00.000000Z",
    "updated": "2026-06-15T09:30:00.000000Z"
  },
  "tags": ["priority", "retainer"]
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Client id. Used in all URLs. |
| `company_name` | string | Client company name. |
| `description` | string\|null | Client description. |
| `status` | string | `active` or `suspended`. |
| `phone` | string\|null | |
| `website` | string\|null | |
| `vat` | string\|null | VAT / tax number. |
| `category` | object | `{ id, name }`. |
| `owner` | object | Primary account owner `{ id, name }` — set via the **owner** action. |
| `billing` | object | Billing address + `invoice_due_days`. |
| `shipping` | object | Shipping address. |
| `dates` | object | `created`, `updated`. |
| `tags` | array | Tag titles. |

---

## List / search clients

```
GET /api/clients
```

### Query parameters

| Parameter | Type | Description |
|---|---|---|
| `status` | string | Filter by client status (`active`\|`suspended`). |
| `category_id` | integer | Filter by category id. |
| `search` | string | Free-text search on the company name. |
| `sort` | string | `client_company_name`, `client_status`, `client_created`, `client_updated`. |
| `order` | string | `asc` or `desc` (default `desc`). |
| `limit` | integer | Results per page (max 100). |
| `page` | integer | Page number. |

### Example request

```bash
curl -G https://your-domain/api/clients \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d status=active \
  -d limit=25
```

```php
<?php
$query = http_build_query(['status' => 'active', 'limit' => 25]);
$ch = curl_init("https://your-domain/api/clients?$query");
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
  "data": [ { "id": 12, "company_name": "Acme Inc" } ],
  "meta": { "current_page": 1, "per_page": 25, "total": 1, "last_page": 1 },
  "message": "Clients retrieved successfully."
}
```

---

## Get a client

```
GET /api/clients/{id}
```

### Example request

```bash
curl https://your-domain/api/clients/12 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/clients/12');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the client (`200`) with `message` "Client retrieved successfully."

---

## Create a client

```
POST /api/clients
```

Creating a client also provisions its **primary account-owner user** from `first_name`,
`last_name` and `email` (these are required). If a `contact`-type user already exists with that
email, it is promoted to the client's primary user. A welcome email is **not** sent unless you
pass `send_email=yes`. The account owner can later be changed with the **owner** action.

### Body parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `first_name` | string | yes | Primary user's first name. |
| `last_name` | string | yes | Primary user's last name. |
| `email` | string | yes | Primary user's email. Must be unique among client/team users. |
| `client_company_name` | string | no | Defaults to "first_name last_name" if omitted. |
| `client_categoryid` | integer | no | Existing category id. Defaults to the system default category. |
| `client_description` | string | no | |
| `client_phone` | string | no | |
| `client_website` | string | no | |
| `client_vat` | string | no | |
| `client_billing_street` | string | no | |
| `client_billing_city` | string | no | |
| `client_billing_state` | string | no | |
| `client_billing_zip` | string | no | |
| `client_billing_country` | string | no | |
| `client_billing_invoice_due_days` | integer | no | |
| `tags` | array of strings | no | Tag titles to attach on create. |
| `send_email` | string | no | `yes` to send the welcome email. Default `no`. |

> Required **custom fields** configured for clients must also be supplied (using their field
> name). Missing required custom fields return a `422`.

### Example request

```bash
curl -X POST https://your-domain/api/clients \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d first_name=Jane \
  -d last_name=Doe \
  -d email=jane@acme.example \
  -d client_company_name="Acme Inc" \
  -d client_categoryid=2
```

```php
<?php
$ch = curl_init('https://your-domain/api/clients');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'first_name'          => 'Jane',
        'last_name'           => 'Doe',
        'email'               => 'jane@acme.example',
        'client_company_name' => 'Acme Inc',
        'client_categoryid'   => 2,
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `201 Created`

```json
{
  "data": { "id": 12, "company_name": "Acme Inc" },
  "message": "Client created successfully."
}
```

---

## Update a client

```
PATCH /api/clients/{id}
```

`client_company_name` is required. The primary user / account owner is **not** changed here —
use the **owner** action.

### Body parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `client_company_name` | string | yes | |
| `client_categoryid` | integer | no | Existing category id. |
| `client_description` | string | no | |
| `client_phone` | string | no | |
| `client_website` | string | no | |
| `client_vat` | string | no | |
| `client_billing_street` | string | no | |
| `client_billing_city` | string | no | |
| `client_billing_state` | string | no | |
| `client_billing_zip` | string | no | |
| `client_billing_country` | string | no | |
| `client_billing_invoice_due_days` | integer | no | |

> As with create, required client **custom fields** must be supplied.

### Example request

```bash
curl -X PATCH https://your-domain/api/clients/12 \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d client_company_name="Acme Incorporated" \
  -d client_categoryid=2
```

```php
<?php
$ch = curl_init('https://your-domain/api/clients/12');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PATCH',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'client_company_name' => 'Acme Incorporated',
        'client_categoryid'   => 2,
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `200 OK`

```json
{
  "data": { "id": 12, "company_name": "Acme Incorporated" },
  "message": "Client updated successfully."
}
```

---

## Delete a client

```
DELETE /api/clients/{id}
```

Deletes the client and all of its related records (users, projects, invoices, files, etc.).

### Example request

```bash
curl -X DELETE https://your-domain/api/clients/12 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/clients/12');
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
  "data": { "id": 12 },
  "message": "Client deleted successfully."
}
```

---

## Set tags

```
PUT /api/clients/{id}/tags
```

Sets the client's tags. The `tags` array is the **full set** — it replaces existing tags; an
empty array clears them.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `tags` | array of strings | yes | Tag titles. Empty array clears all. |

```bash
curl -X PUT https://your-domain/api/clients/12/tags \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d "tags[]=priority" -d "tags[]=retainer"
```

```php
<?php
$ch = curl_init('https://your-domain/api/clients/12/tags');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['tags' => ['priority', 'retainer']]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the client (`200`) with `message` "Client tags updated successfully."

---

## Change status

```
PUT /api/clients/{id}/status
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `status` | string | yes | `active` or `suspended`. |

```bash
curl -X PUT https://your-domain/api/clients/12/status \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d status=suspended
```

```php
<?php
$ch = curl_init('https://your-domain/api/clients/12/status');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['status' => 'suspended']),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the client (`200`) with `message` "Client status updated successfully."

---

## Change account owner

```
PUT /api/clients/{id}/owner
```

Sets the client's primary account owner. The `owner` must be the id of a user that **already
belongs to this client**; it replaces the previous account owner.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `owner` | integer | yes | Id of a user belonging to this client. |

```bash
curl -X PUT https://your-domain/api/clients/12/owner \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d owner=34
```

```php
<?php
$ch = curl_init('https://your-domain/api/clients/12/owner');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['owner' => 34]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the client (`200`) with `message` "Client account owner updated successfully."

---

## Errors

See [Getting started](getting-started.md#errors) for the shared error format. Client-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The client id does not exist. |
| `422 Unprocessable Entity` | Validation failed (see below). |

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "email": ["The email has already been taken."],
    "first_name": ["The first name field is required."]
  }
}
```

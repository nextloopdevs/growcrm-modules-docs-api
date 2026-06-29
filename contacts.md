---
title: Contacts
description: Create, update, delete, list and search contacts (client users) via the Grow CRM API.
---

# Contacts

The Contacts API provides CRUD access to contacts — the users that belong to a client account
(`type = client`) — plus an ancillary action for setting a contact as the client's account owner.

> New to the API? Start with **[Getting started](getting-started.md)** (base URL, response
> envelope, errors, pagination) and **[Authentication](authentication.md)** (API keys). Those
> conventions apply to every endpoint below and are not repeated here.

> **Identifier note:** contacts are addressed by their numeric **`id`**. Wherever `{id}` appears
> below it is the contact `id`.

## The contact object

All endpoints that return a contact use this shape:

```json
{
  "id": 45,
  "first_name": "Jane",
  "last_name": "Doe",
  "name": "Jane Doe",
  "email": "jane@acme.example",
  "phone": "+1 555 0100",
  "job_position": "Procurement Lead",
  "account_owner": "no",
  "status": "active",
  "client": { "id": 12, "company_name": "Acme Inc" },
  "role": { "id": 2, "name": "Client" },
  "social": {
    "facebook": null,
    "twitter": null,
    "linkedin": null,
    "github": null,
    "dribbble": null
  },
  "dates": {
    "created": "2026-06-15T09:30:00.000000Z",
    "updated": "2026-06-15T09:30:00.000000Z"
  }
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Contact id. Used in all URLs. |
| `first_name` | string | |
| `last_name` | string | |
| `name` | string | Full name (first + last). |
| `email` | string | Unique across all users. |
| `phone` | string\|null | |
| `job_position` | string\|null | |
| `account_owner` | string | `yes` or `no`. Set via the **account-owner** action. |
| `status` | string | Contact status (e.g. `active`). |
| `client` | object | The client the contact belongs to `{ id, company_name }`. |
| `role` | object | `{ id, name }`. |
| `social` | object | Social profile handles. |
| `dates` | object | `created`, `updated`. |

---

## List / search contacts

```
GET /api/contacts
```

### Query parameters

| Parameter | Type | Description |
|---|---|---|
| `status` | string | Filter by contact status. By default, soft-deleted contacts are excluded. |
| `client_id` | integer | Filter by the client the contact belongs to. |
| `account_owner` | string | Filter by account owner flag (`yes`\|`no`). |
| `search` | string | Free-text search on first/last name, email and phone. |
| `sort` | string | `first_name`, `last_name`, `email`, `created`, `updated`. |
| `order` | string | `asc` or `desc` (default `asc`). |
| `limit` | integer | Results per page (max 100). |
| `page` | integer | Page number. |

### Example request

```bash
curl -G https://your-domain/api/contacts \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d client_id=12 \
  -d limit=25
```

```php
<?php
$query = http_build_query(['client_id' => 12, 'limit' => 25]);
$ch = curl_init("https://your-domain/api/contacts?$query");
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
  "data": [ { "id": 45, "name": "Jane Doe" } ],
  "meta": { "current_page": 1, "per_page": 25, "total": 1, "last_page": 1 },
  "message": "Contacts retrieved successfully."
}
```

---

## Get a contact

```
GET /api/contacts/{id}
```

### Example request

```bash
curl https://your-domain/api/contacts/45 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/contacts/45');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the contact (`200`) with `message` "Contact retrieved successfully."

---

## Create a contact

```
POST /api/contacts
```

Creates a contact (client-type user) under an existing client account. A password is generated
automatically. A welcome email is **not** sent unless you pass `send_email=yes`. The email must
be unique across all users — a duplicate returns a `422`.

### Body parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `first_name` | string | yes | |
| `last_name` | string | yes | |
| `email` | string | yes | Must be unique across all users. |
| `clientid` | integer | yes | Existing client id the contact belongs to. |
| `phone` | string | no | |
| `position` | string | no | Job position. |
| `send_email` | string | no | `yes` to send the welcome email. Default `no`. |

### Example request

```bash
curl -X POST https://your-domain/api/contacts \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d first_name=Jane \
  -d last_name=Doe \
  -d email=jane@acme.example \
  -d clientid=12
```

```php
<?php
$ch = curl_init('https://your-domain/api/contacts');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'first_name' => 'Jane',
        'last_name'  => 'Doe',
        'email'      => 'jane@acme.example',
        'clientid'   => 12,
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `201 Created`

```json
{
  "data": { "id": 45, "name": "Jane Doe" },
  "message": "Contact created successfully."
}
```

---

## Update a contact

```
PATCH /api/contacts/{id}
```

`first_name`, `last_name` and `email` are required. The account owner is **not** changed here —
use the **account-owner** action.

### Body parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `first_name` | string | yes | |
| `last_name` | string | yes | |
| `email` | string | yes | Must remain unique (the current contact is ignored). |
| `phone` | string | no | |
| `position` | string | no | Job position. |

### Example request

```bash
curl -X PATCH https://your-domain/api/contacts/45 \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d first_name=Jane \
  -d last_name=Smith \
  -d email=jane@acme.example
```

```php
<?php
$ch = curl_init('https://your-domain/api/contacts/45');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PATCH',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'first_name' => 'Jane',
        'last_name'  => 'Smith',
        'email'      => 'jane@acme.example',
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `200 OK`

```json
{
  "data": { "id": 45, "name": "Jane Smith" },
  "message": "Contact updated successfully."
}
```

---

## Delete a contact

```
DELETE /api/contacts/{id}
```

Soft-deletes the contact (the account is marked deleted and its email/password are cleared),
mirroring the web application's delete behaviour. Once deleted, the contact is treated as gone:
it no longer appears in listings and any subsequent request for it by id (get, update, delete,
account-owner) returns `404`.

### Example request

```bash
curl -X DELETE https://your-domain/api/contacts/45 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/contacts/45');
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
  "message": "Contact deleted successfully."
}
```

---

## Set as account owner

```
PUT /api/contacts/{id}/account-owner
```

Promotes the contact to the account owner of its client. This replaces the client's previous
account owner. No request body is required.

```bash
curl -X PUT https://your-domain/api/contacts/45/account-owner \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/contacts/45/account-owner');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the contact (`200`) with `account_owner` now `yes` and `message` "Contact set as account
owner successfully."

---

## Errors

See [Getting started](getting-started.md#errors) for the shared error format. Contact-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The contact id does not exist, or the contact has been deleted. |
| `409 Conflict` | The contact does not belong to a client account (account-owner action). |
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

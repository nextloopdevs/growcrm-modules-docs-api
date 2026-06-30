---
title: Getting started
description: Base URL, conventions, responses and errors for the CRM API.
---

# Getting started

The CRM API is a JSON REST API. This page covers the conventions shared by every endpoint.
For credentials, see **[Authentication](/authentication)**.

## Base URL

```
https://your-domain/api
```

On the multitenant (SaaS) edition this is your own account domain — see
[Authentication](/authentication).

## Making a request

Send the API key as a Bearer token. A minimal first request:

```bash
curl https://your-domain/api/projects \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/projects');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

## Response envelope

Successful responses use a consistent envelope:

```json
{
  "data": { },
  "meta": { },
  "message": "..."
}
```

- `data` — the resource object, or an array of objects for list endpoints.
- `meta` — pagination info, present on list endpoints only.
- `message` — a human-readable summary.

## Errors

Errors return the matching HTTP status with a JSON body:

```json
{ "message": "...", "errors": { "field": ["..."] } }
```

`errors` is present only for validation failures.

| Status | Meaning |
|---|---|
| `401 Unauthorized` | Missing or invalid API key. |
| `404 Not Found` | The resource id does not exist. |
| `422 Unprocessable Entity` | Validation failed. |

## Identifiers

Resources are addressed by their **unique id** (an opaque string), not a sequential number.
The unique id is the `id` field on each object and is what you put in URLs, e.g.
`/api/projects/{id}`.

## Pagination, filtering & sorting (list endpoints)

| Parameter | Description |
|---|---|
| `page` | Page number. |
| `limit` | Results per page (max 100). |
| `sort` | Sort column (whitelisted per resource). |
| `order` | `asc` or `desc` (default `desc`). |

Resource-specific filters are documented on each resource's page.

## Sub-resource actions

Ancillary actions on a resource live under its id and carry only the action's data in the body:

```
PUT /api/projects/{id}/assign     body: { "assigned": [5, 8, 12] }
PUT /api/projects/{id}/manager    body: { "manager": 5 }
```

The id is always in the **path**; the body never repeats it.

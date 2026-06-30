---
title: Categories
description: Create, update, delete, list and search categories via the Grow CRM API.
---

# Categories

The Categories API provides CRUD access to categories, plus ancillary actions for assigning team
members to a category and migrating a category's resources into another category.

> New to the API? Start with **[Getting started](/getting-started)** (base URL, response
> envelope, errors, pagination) and **[Authentication](/authentication)** (API keys). Those
> conventions apply to every endpoint below and are not repeated here.

> **Identifier note:** categories are addressed by their numeric **`category_id`**. Wherever
> `{id}` appears below it is the `category_id`.

## The category object

All endpoints that return a category use this shape:

```json
{
  "id": 3,
  "name": "Development",
  "description": "Software development work.",
  "type": "project",
  "visibility": "everyone",
  "icon": "ti ti-folder",
  "system_default": "no",
  "slug": "3-development",
  "count": 12,
  "users_count": 4,
  "dates": {
    "created": "2026-06-15T09:30:00.000000Z",
    "updated": "2026-06-15T09:30:00.000000Z"
  }
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Category id. Used in all URLs. |
| `name` | string | Category name. |
| `description` | string\|null | Category description. |
| `type` | string | Resource type the category belongs to (e.g. `project`, `client`, `ticket`, `invoice`, …). Set on create, immutable thereafter. |
| `visibility` | string\|null | Category visibility. |
| `icon` | string\|null | Icon class. |
| `system_default` | string | `yes` if this is a system-default category (cannot be deleted). |
| `slug` | string | URL slug (auto-generated). |
| `count` | integer | Number of resources of this category's type currently in the category. |
| `users_count` | integer | Number of team members assigned to the category. |
| `dates` | object | `created`, `updated`. |

---

## List / search categories

```
GET /api/categories
```

### Query parameters

| Parameter | Type | Description |
|---|---|---|
| `type` | string | Filter by category type. |
| `visibility` | string | Filter by visibility. |
| `search` | string | Free-text search on the category name. |
| `sort` | string | `category_name`, `category_type`, `category_created`. |
| `order` | string | `asc` or `desc` (default `desc`). |
| `limit` | integer | Results per page (max 100). |
| `page` | integer | Page number. |

### Example request

```bash
curl -G https://your-domain/api/categories \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d type=project \
  -d limit=25
```

```php
<?php
$query = http_build_query(['type' => 'project', 'limit' => 25]);
$ch = curl_init("https://your-domain/api/categories?$query");
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
  "data": [ { "id": 3, "name": "Development", "type": "project" } ],
  "meta": { "current_page": 1, "per_page": 25, "total": 1, "last_page": 1 },
  "message": "Categories retrieved successfully."
}
```

---

## Get a category

```
GET /api/categories/{id}
```

### Example request

```bash
curl https://your-domain/api/categories/3 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/categories/3');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the category (`200`) with `message` "Category retrieved successfully."

---

## Create a category

```
POST /api/categories
```

### Body parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `category_name` | string | yes | Must be unique within the same `category_type`. |
| `category_type` | string | yes | Resource type (e.g. `project`, `client`, `ticket`, …). Not changeable later. |
| `category_visibility` | string | no | |
| `category_description` | string | no | |
| `category_icon` | string | no | Icon class. |

### Example request

```bash
curl -X POST https://your-domain/api/categories \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d category_name="Development" \
  -d category_type=project
```

```php
<?php
$ch = curl_init('https://your-domain/api/categories');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'category_name' => 'Development',
        'category_type' => 'project',
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `201 Created`

```json
{
  "data": { "id": 3, "name": "Development", "type": "project" },
  "message": "Category created successfully."
}
```

---

## Update a category

```
PATCH /api/categories/{id}
```

`category_name` is required and must be unique within the category's type. `category_type`
**cannot** be changed.

### Body parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `category_name` | string | yes | |
| `category_visibility` | string | no | |
| `category_description` | string | no | |
| `category_icon` | string | no | |

### Example request

```bash
curl -X PATCH https://your-domain/api/categories/3 \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d category_name="Dev & Engineering"
```

```php
<?php
$ch = curl_init('https://your-domain/api/categories/3');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PATCH',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['category_name' => 'Dev & Engineering']),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Example response — `200 OK`

```json
{
  "data": { "id": 3, "name": "Dev & Engineering", "type": "project" },
  "message": "Category updated successfully."
}
```

---

## Delete a category

```
DELETE /api/categories/{id}
```

A category can only be deleted when it is **empty** (`count` is `0`) and is **not** a
system-default category. Otherwise a `409` is returned. To empty a non-empty category first,
use the **migrate** action below. Deleting also removes the category's team assignments.

### Example request

```bash
curl -X DELETE https://your-domain/api/categories/3 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```php
<?php
$ch = curl_init('https://your-domain/api/categories/3');
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
  "data": { "id": 3 },
  "message": "Category deleted successfully."
}
```

---

## Set team members

```
PUT /api/categories/{id}/team
```

Sets the team members assigned to the category. The `users` array is the **full set** — it
replaces any existing assignment; an empty array clears all. Category-based project permissions
are re-synced automatically.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `users` | array of integers | yes | Team user ids. Empty array clears all. |

```bash
curl -X PUT https://your-domain/api/categories/3/team \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d "users[]=5" -d "users[]=8"
```

```php
<?php
$ch = curl_init('https://your-domain/api/categories/3/team');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['users' => [5, 8]]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the category (`200`) with `message` "Category team updated successfully."

---

## Migrate resources

```
PUT /api/categories/{id}/migrate
```

Moves **all** of this category's resources (projects, clients, invoices, estimates, leads,
tickets, items, expenses, contracts) into another category, and re-syncs category-based project
permissions. Use this to empty a category before deleting it. Returns the (now-emptied) source
category.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `target_id` | integer | yes | Destination category id. Must exist and differ from `{id}`. |

```bash
curl -X PUT https://your-domain/api/categories/3/migrate \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d target_id=4
```

```php
<?php
$ch = curl_init('https://your-domain/api/categories/3/migrate');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query(['target_id' => 4]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the source category (`200`) with `message` "Category resources migrated successfully."

---

## Errors

See [Getting started](/getting-started#errors) for the shared error format. Category-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The category id does not exist. |
| `409 Conflict` | Cannot delete a system-default or non-empty category. |
| `422 Unprocessable Entity` | Validation failed (e.g. missing name/type, or a duplicate name within the type). |

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "category_name": ["A category with this name already exists."],
    "category_type": ["The category type field is required."]
  }
}
```

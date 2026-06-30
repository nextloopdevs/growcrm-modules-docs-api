---
title: Items (Products)
description: Create, update, delete and list catalog items (products/services) via the CRM API.
---

# Items

The Items API provides CRUD access to the product/service catalog. Catalog items populate
invoice and estimate line items.

> New to the API? Start with **[Getting started](/getting-started)** (base URL, response
> envelope, errors, pagination) and **[Authentication](/authentication)** (API keys).

> **Identifier note:** items are addressed by their numeric **`item_id`**.

> **An item sits under an item category** (a category of type `item`) and carries a **unit**, a
> **rate** and a **default tax rate**.

> **Scope:** production tasks, bulk change-category, pinning, import and the purchasing/stock
> module are out of API scope. Items have no tags.

## The item object

```json
{
  "id": 28,
  "type": "standard",
  "description": "Logo Design",
  "notes": null,
  "rate": "200.00",
  "tax_status": "taxable",
  "default_tax": { "id": 1, "name": "VAT" },
  "unit": { "id": 4, "name": "Pcs" },
  "category": { "id": 8, "name": "Default" },
  "creator_id": 1,
  "estimation_notes": null,
  "custom_fields": [ { "id": 1, "name": "ISBN", "value": "ISBN-978-3-16-148410-0" } ],
  "dates": { "created": "...", "updated": "..." }
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Item id. |
| `description` / `notes` | string | |
| `rate` | numeric | Unit rate. |
| `default_tax` | object | The default tax rate (`id`, `name`). |
| `unit` | object | The unit of measure (`id`, `name`). |
| `category` | object | The item category. |
| `estimation_notes` | string | Optional estimation note. |
| `custom_fields` | array | Enabled item custom fields with their values (`id`, `name`, `value`). |

---

## List / search items

```
GET /api/items
```

Query params: `category_id`, `unit_id`, `tax_rate_id`, `rate_min`, `rate_max`, `search`,
`sort` (`item_description`,`item_rate`,`item_created`), `order`, `limit`, `page`.

```bash
curl -G https://your-domain/api/items -H "Authorization: Bearer YOUR_API_KEY" -d category_id=8
```

---

## Get an item

```
GET /api/items/{id}
```

---

## Create an item

```
POST /api/items
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `item_description` | string | yes | |
| `item_rate` | numeric | yes | |
| `item_categoryid` | integer | yes | A category of type `item`. |
| `item_default_tax` | integer | yes | A `taxrate_id` (use the "No Tax" rate for tax-free items). |
| `item_unit` | mixed | yes | An existing `unit_id`, or a **new unit name** (auto-created). |
| `item_notes` | string | no | |
| `item_notes_estimatation` | string | no | |
| `item_custom_field_1..10` | mixed | no | Values for enabled item custom fields. |

```bash
curl -X POST https://your-domain/api/items -H "Authorization: Bearer YOUR_API_KEY" \
  -d item_description="Logo Design" -d item_rate=200 -d item_categoryid=8 \
  -d item_default_tax=1 -d item_unit=Hours -d item_custom_field_1="ISBN-001"
```

Returns the new item (`201`).

---

## Update an item

```
PATCH /api/items/{id}
```

Same fields as create.

```bash
curl -X PATCH https://your-domain/api/items/28 -H "Authorization: Bearer YOUR_API_KEY" \
  -d item_description="Logo Design" -d item_rate=250 -d item_categoryid=8 \
  -d item_default_tax=1 -d item_unit=4
```

---

## Delete an item

```
DELETE /api/items/{id}
```

---

## Errors

See [Getting started](/getting-started#errors). Item-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The item id does not exist. |
| `422 Unprocessable Entity` | Validation failed (missing fields, non-item category, invalid tax rate, or an invalid unit name). |

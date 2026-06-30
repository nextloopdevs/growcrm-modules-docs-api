---
title: Invoices
description: Create, update, delete, list, line-item, publish and email invoices via the Grow CRM API.
---

# Invoices

The Invoices API provides CRUD access to invoices, line-item management, status, tags, clone,
publish and email.

> New to the API? Start with **[Getting started](/getting-started)** (base URL, response
> envelope, errors, pagination) and **[Authentication](/authentication)** (API keys).

> **Identifier note:** invoices are addressed by their numeric **`bill_invoiceid`**.

> **Totals are computed server-side.** You never send totals. You set the line items (and, for
> summary mode, the tax rates) and the header discount/adjustment; the API computes the subtotal,
> discount, tax and final amounts to match the CRM exactly.

> **Scope:** payment recording/gateways (a future Payments resource), recurring, file
> attachments and bulk actions are out of API scope.

## The invoice object

```json
{
  "id": 295,
  "status": { "id": 1, "title": "Draft", "color": "default" },
  "client": { "id": 12, "name": "Acme Inc" },
  "project_id": null,
  "category": { "id": 4, "name": "Default" },
  "dates": { "date": "2026-06-19", "due_date": "2026-07-19", "created": "..." },
  "notes": null,
  "terms": null,
  "tax_type": "summary",
  "totals": {
    "subtotal": "250.00", "subtotal_before_discount": "0.00",
    "discount_type": "fixed", "discount_percentage": "0.00", "discount_amount": "25.00",
    "amount_before_tax": "225.00", "tax_total_percentage": "10.00", "tax_total_amount": "22.50",
    "adjustment_description": null, "adjustment_amount": "0.00", "final_amount": "247.50"
  },
  "line_items": [
    { "id": 1, "description": "Item A", "unit": "hours", "quantity": "2.00", "rate": "100.00",
      "total": "200.00", "type": "plain", "tax_status": "no",
      "discount": { "type": "none", "value": "0.00", "amount": "0.00" } }
  ],
  "taxes": [ { "name": "VAT", "rate": "10.00", "lineitem_id": null } ],
  "tags": ["priority"]
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Invoice id. |
| `status` | object | `{ id, title, color }` (`invoicestatus_id`: 1 Draft, 2 Due, 3 Overdue, 4 Part Paid, 5 Paid, 20 Cancelled). |
| `client` / `project_id` / `category` | object/int | Links. |
| `dates` | object | `date`, `due_date`, `created`. |
| `tax_type` | string | `summary` or `inline`. |
| `totals` | object | All amounts — **read-only**, computed by the server. |
| `line_items` | array | The line items. |
| `taxes` | array | Tax rate records (summary: `lineitem_id` null; inline: per line). |
| `tags` | array | Tag titles. |

---

## List / search invoices

```
GET /api/invoices
```

Query params: `status`, `client_id`, `project_id`, `category_id`, `search`, `sort`
(`bill_date`,`bill_due_date`,`bill_final_amount`,`bill_created`), `order`, `limit`, `page`.

```bash
curl -G https://your-domain/api/invoices -H "Authorization: Bearer YOUR_API_KEY" -d client_id=12
```

---

## Get an invoice

```
GET /api/invoices/{id}
```

Returns the invoice with line items + computed totals.

---

## Create an invoice

```
POST /api/invoices
```

Creates an empty invoice header. Add line items next via the **items** action.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `bill_clientid` | integer | yes | Existing client. |
| `bill_categoryid` | integer | yes | Existing category. |
| `bill_date` | date | yes | |
| `bill_due_date` | date | no | Defaults to client/system due days. |
| `bill_projectid` | integer | no | |
| `bill_tax_type` | string | no | `summary` (default) or `inline`. |
| `bill_notes` / `bill_terms` | string | no | |

```bash
curl -X POST https://your-domain/api/invoices -H "Authorization: Bearer YOUR_API_KEY" \
  -d bill_clientid=12 -d bill_categoryid=4 -d bill_date=2026-06-19 -d bill_tax_type=summary
```

Returns the new invoice (`201`, status Draft).

---

## Update an invoice (header) + recalculate

```
PATCH /api/invoices/{id}
```

Sets header fields, including the **discount**, **tax mode** and **adjustment** that drive the
totals; the invoice is recalculated.

| Parameter | Type | Notes |
|---|---|---|
| `bill_categoryid` | integer | required |
| `bill_date` | date | required |
| `bill_due_date` | date | optional |
| `bill_projectid` | integer | optional |
| `bill_notes` / `bill_terms` | string | optional |
| `bill_tax_type` | string | `summary`/`inline` |
| `bill_discount_type` | string | `none`/`fixed`/`percentage` |
| `bill_discount_percentage` | numeric | for percentage discounts |
| `bill_discount_amount` | numeric | for fixed discounts |
| `bill_adjustment_description` | string | |
| `bill_adjustment_amount` | numeric | may be negative |

```bash
curl -X PATCH https://your-domain/api/invoices/295 -H "Authorization: Bearer YOUR_API_KEY" \
  -d bill_categoryid=4 -d bill_date=2026-06-19 -d bill_discount_type=fixed -d bill_discount_amount=25
```

---

## Set line items + recalculate

```
PUT /api/invoices/{id}/items
```

Replaces the invoice's line items and recomputes all totals (matching the CRM UI to the cent).
The behaviour follows the invoice's `tax_type`:
- **summary** — pass invoice-level `tax_rate_ids[]`; the tax applies to the whole invoice.
- **inline** — give each item its own `tax_rate_id` (and optional per-line `discount_type`/`discount_value`).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `items[]` | array | yes | The full set of line items (replaces existing). |
| `items[].description` | string | yes | |
| `items[].quantity` | numeric | yes | |
| `items[].rate` | numeric | yes | |
| `items[].unit` | string | no | |
| `items[].long_description` | string | no | |
| `items[].tax_rate_id` | integer | no | Inline mode: a tax rate id for the line. |
| `items[].discount_type` / `items[].discount_value` | string/numeric | no | Inline mode: per-line discount. |
| `tax_rate_ids[]` | array of ints | no | Summary mode: invoice-level tax rate ids. |

```bash
curl -X PUT https://your-domain/api/invoices/295/items \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d "items[0][description]=Design" -d "items[0][unit]=hours" -d "items[0][quantity]=2" -d "items[0][rate]=100" \
  -d "items[1][description]=Hosting" -d "items[1][quantity]=1" -d "items[1][rate]=50" \
  -d "tax_rate_ids[]=1"
```

```php
<?php
$ch = curl_init('https://your-domain/api/invoices/295/items');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CUSTOMREQUEST  => 'PUT',
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'items' => [
            ['description' => 'Design', 'unit' => 'hours', 'quantity' => 2, 'rate' => 100],
            ['description' => 'Hosting', 'quantity' => 1, 'rate' => 50],
        ],
        'tax_rate_ids' => [1],
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the invoice with recomputed `totals` (`200`).

---

## Delete an invoice

```
DELETE /api/invoices/{id}
```

---

## Change status

```
PUT /api/invoices/{id}/status
```

| Parameter | Type | Notes |
|---|---|---|
| `status` | integer | An `invoicestatus_id`. Set manually; it sticks (use 20 to cancel). |

```bash
curl -X PUT https://your-domain/api/invoices/295/status -H "Authorization: Bearer YOUR_API_KEY" -d status=2
```

---

## Set tags

```
PUT /api/invoices/{id}/tags
```

Full-set replace (empty clears).

---

## Clone

```
POST /api/invoices/{id}/clone
```

| Parameter | Type | Required |
|---|---|---|
| `bill_clientid` | integer | yes |
| `bill_date` | date | yes |
| `bill_due_date` | date | yes |
| `bill_projectid` | integer | no |

Returns the **new** invoice (`201`).

---

## Publish

```
PUT /api/invoices/{id}/publish
```

Publishes a **draft** invoice (moves it out of draft and queues the invoice email to the client).
No body. `409` if already published.

```bash
curl -X PUT https://your-domain/api/invoices/295/publish -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Email the client

```
POST /api/invoices/{id}/send-email
```

Queues the invoice email to the client (as the UI does). The invoice must not be a draft
(`409` otherwise). No body.

```bash
curl -X POST https://your-domain/api/invoices/295/send-email -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Errors

See [Getting started](/getting-started#errors). Invoice-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The invoice id does not exist. |
| `409 Conflict` | Already published, or emailing a draft. |
| `422 Unprocessable Entity` | Validation failed (e.g. missing client/category/date, invalid status/tax rate). |

---
title: Estimates
description: Create, update, delete, list, line-item, publish, email and convert estimates via the CRM API.
---

# Estimates

The Estimates API provides CRUD access to estimates, line-item management, status, tags, clone,
publish, email and convert-to-invoice.

> New to the API? Start with **[Getting started](/getting-started)** (base URL, response
> envelope, errors, pagination) and **[Authentication](/authentication)** (API keys).

> **Identifier note:** estimates are addressed by their numeric **`bill_estimateid`**.

> **Totals are computed server-side.** You never send totals. You set the line items (and, for
> summary mode, the tax rates) and the header discount/adjustment; the API computes the subtotal,
> discount, tax and final amounts to match the CRM exactly.

> **Scope:** file attachments and bulk actions are out of API scope.

## The estimate object

```json
{
  "id": 100,
  "status": "accepted",
  "client": { "id": 12, "name": "Acme Inc" },
  "project_id": null,
  "category": { "id": 4, "name": "Default" },
  "dates": { "date": "2026-06-20", "expiry_date": "2026-07-20", "created": "..." },
  "notes": null,
  "terms": null,
  "tax_type": "inline",
  "totals": {
    "subtotal": "247.50", "subtotal_before_discount": "250.00",
    "discount_type": "none", "discount_percentage": "0.00", "discount_amount": "25.00",
    "amount_before_tax": "247.50", "tax_total_percentage": "0.00", "tax_total_amount": "22.50",
    "adjustment_description": null, "adjustment_amount": "0.00", "final_amount": "247.50"
  },
  "converted": { "is_converted": false, "invoice_id": null },
  "line_items": [
    { "id": 1, "description": "Item A", "unit": "hours", "quantity": "2.00", "rate": "100.00",
      "total": "198.00", "type": "plain", "tax_status": "yes",
      "discount": { "type": "fixed", "value": "20.00", "amount": "20.00" } }
  ],
  "taxes": [ { "name": "VAT", "rate": "10.00", "lineitem_id": 1627 } ],
  "tags": ["priority"]
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Estimate id. |
| `status` | string | `draft`, `new`, `accepted`, `declined`, `revised`, `expired`. |
| `client` / `project_id` / `category` | object/int | Links. |
| `dates` | object | `date`, `expiry_date`, `created`. |
| `tax_type` | string | `summary` or `inline`. |
| `totals` | object | All amounts — **read-only**, computed by the server. |
| `converted` | object | `is_converted` and the resulting `invoice_id` (once converted). |
| `line_items` | array | The line items. |
| `taxes` | array | Tax rate records (summary: `lineitem_id` null; inline: per line). |
| `tags` | array | Tag titles. |

---

## List / search estimates

```
GET /api/estimates
```

Query params: `status`, `client_id`, `project_id`, `category_id`, `search`, `sort`
(`bill_date`,`bill_expiry_date`,`bill_final_amount`,`bill_created`), `order`, `limit`, `page`.

```bash
curl -G https://your-domain/api/estimates -H "Authorization: Bearer YOUR_API_KEY" -d client_id=12
```

---

## Get an estimate

```
GET /api/estimates/{id}
```

Returns the estimate with line items + computed totals.

---

## Create an estimate

```
POST /api/estimates
```

Creates an empty estimate header. Add line items next via the **items** action.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `bill_clientid` | integer | yes | Existing client. |
| `bill_categoryid` | integer | yes | Existing category. |
| `bill_date` | date | yes | |
| `bill_expiry_date` | date | no | Optional. |
| `bill_projectid` | integer | no | |
| `bill_tax_type` | string | no | `summary` (default) or `inline`. |
| `bill_notes` / `bill_terms` | string | no | |

```bash
curl -X POST https://your-domain/api/estimates -H "Authorization: Bearer YOUR_API_KEY" \
  -d bill_clientid=12 -d bill_categoryid=4 -d bill_date=2026-06-20 -d bill_tax_type=summary
```

Returns the new estimate (`201`, status `draft`).

---

## Update an estimate (header) + recalculate

```
PATCH /api/estimates/{id}
```

Sets header fields, including the **discount**, **tax mode** and **adjustment** that drive the
totals; the estimate is recalculated. The status is preserved.

| Parameter | Type | Notes |
|---|---|---|
| `bill_categoryid` | integer | required |
| `bill_date` | date | required |
| `bill_expiry_date` | date | optional |
| `bill_projectid` | integer | optional |
| `bill_notes` / `bill_terms` | string | optional |
| `bill_tax_type` | string | `summary`/`inline` |
| `bill_discount_type` | string | `none`/`fixed`/`percentage` |
| `bill_discount_percentage` | numeric | for percentage discounts |
| `bill_discount_amount` | numeric | for fixed discounts |
| `bill_adjustment_description` | string | |
| `bill_adjustment_amount` | numeric | may be negative |

```bash
curl -X PATCH https://your-domain/api/estimates/100 -H "Authorization: Bearer YOUR_API_KEY" \
  -d bill_categoryid=4 -d bill_date=2026-06-20 -d bill_discount_type=fixed -d bill_discount_amount=25
```

---

## Set line items + recalculate

```
PUT /api/estimates/{id}/items
```

Replaces the estimate's line items and recomputes all totals (matching the CRM UI to the cent).
The behaviour follows the estimate's `tax_type`:
- **summary** — pass estimate-level `tax_rate_ids[]`; the tax applies to the whole estimate.
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
| `tax_rate_ids[]` | array of ints | no | Summary mode: estimate-level tax rate ids. |

```bash
curl -X PUT https://your-domain/api/estimates/100/items \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d "items[0][description]=Design" -d "items[0][unit]=hours" -d "items[0][quantity]=2" -d "items[0][rate]=100" \
  -d "items[1][description]=Hosting" -d "items[1][quantity]=1" -d "items[1][rate]=50" \
  -d "tax_rate_ids[]=1"
```

```php
<?php
$ch = curl_init('https://your-domain/api/estimates/100/items');
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

Returns the estimate with recomputed `totals` (`200`).

---

## Delete an estimate

```
DELETE /api/estimates/{id}
```

---

## Change status

```
PUT /api/estimates/{id}/status
```

| Parameter | Type | Notes |
|---|---|---|
| `status` | string | One of `draft`, `new`, `accepted`, `declined`, `revised`. |

```bash
curl -X PUT https://your-domain/api/estimates/100/status -H "Authorization: Bearer YOUR_API_KEY" -d status=accepted
```

---

## Set tags

```
PUT /api/estimates/{id}/tags
```

Full-set replace (empty clears).

---

## Clone

```
POST /api/estimates/{id}/clone
```

| Parameter | Type | Required |
|---|---|---|
| `bill_clientid` | integer | yes |
| `bill_categoryid` | integer | yes |
| `bill_date` | date | yes |
| `bill_expiry_date` | date | no |
| `bill_projectid` | integer | no |

Returns the **new** estimate (`201`, status `draft`).

---

## Publish

```
PUT /api/estimates/{id}/publish
```

Publishes a **draft** estimate (moves it out of draft and queues the estimate email to the client).
No body. `409` if already published.

```bash
curl -X PUT https://your-domain/api/estimates/100/publish -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Email the client

```
POST /api/estimates/{id}/send-email
```

Queues the estimate email to the client (as the UI does). The estimate must not be a draft
(`409` otherwise). No body.

```bash
curl -X POST https://your-domain/api/estimates/100/send-email -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Convert to invoice

```
POST /api/estimates/{id}/convert-to-invoice
```

Creates an invoice from the estimate (carrying over the line items and totals) and links the two.
No body. Returns the **new invoice** object (`201`) — see [Invoices](/invoices).

```bash
curl -X POST https://your-domain/api/estimates/100/convert-to-invoice -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Errors

See [Getting started](/getting-started#errors). Estimate-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The estimate id does not exist. |
| `409 Conflict` | Already published, or emailing a draft. |
| `422 Unprocessable Entity` | Validation failed (e.g. missing client/category/date, invalid status/tax rate). |

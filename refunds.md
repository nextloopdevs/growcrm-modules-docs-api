---
title: Refunds
description: Create, update, delete, list and tag payment refunds via the CRM API.
---

# Refunds

The Refunds API provides CRUD access to refunds plus tags.

> New to the API? Start with **[Getting started](/getting-started)** (base URL, response
> envelope, errors, pagination) and **[Authentication](/authentication)** (API keys).

> **Identifier note:** refunds are addressed by their numeric **`refund_id`**.

> **A refund belongs to a payment.** You supply `refund_paymentid`; the amount, client and
> invoice are derived from that payment (a refund is for the **full** payment amount — there are
> no partials, and one refund per payment). Creating a refund marks the payment `refunded`;
> deleting it marks the payment `paid` again.

> **Same flow as Payments.** A refund can also be created/removed from the payment directly via
> `POST` / `DELETE /payments/{id}/refund` — both paths behave identically.

## The refund object

```json
{
  "id": 13,
  "formatted_id": "#000013",
  "amount": "80.00",
  "date": "2026-06-20",
  "notes": "Customer cancelled order",
  "payment": { "id": 150 },
  "invoice_id": 294,
  "client": { "id": 1, "name": "Acme Inc" },
  "creator_id": 1,
  "dates": { "date": "2026-06-20", "created": "..." },
  "tags": ["chargeback"]
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Refund id. |
| `formatted_id` | string | Display id (e.g. `#000013`). |
| `amount` | numeric | The full payment amount. Read-only (derived from the payment). |
| `date` / `notes` | date/string | Editable. |
| `payment` | object | The refunded payment (`id`). |
| `invoice_id` | int | Derived from the payment. |
| `client` | object | Derived from the payment. |
| `creator_id` | int | The API admin. |
| `tags` | array | Tag titles. |

---

## List / search refunds

```
GET /api/refunds
```

Query params: `payment_id`, `client_id`, `invoice_id`, `date_start`, `date_end`, `search`,
`tags[]`, `sort` (`refund_date`,`refund_amount`,`refund_created`), `order`, `limit`, `page`.

```bash
curl -G https://your-domain/api/refunds -H "Authorization: Bearer YOUR_API_KEY" -d client_id=1
```

---

## Get a refund

```
GET /api/refunds/{id}
```

---

## Create a refund

```
POST /api/refunds
```

Refunds a payment (full amount). The payment must not already have a refund (`409` otherwise).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `refund_paymentid` | integer | yes | The payment to refund. |
| `refund_date` | date | yes | |
| `refund_notes` | string | no | |

```bash
curl -X POST https://your-domain/api/refunds -H "Authorization: Bearer YOUR_API_KEY" \
  -d refund_paymentid=150 -d refund_date=2026-06-20 -d refund_notes="Customer cancelled order"
```

Returns the new refund (`201`); the payment is marked `refunded`.

---

## Update a refund

```
PATCH /api/refunds/{id}
```

Only the date and notes are editable (amount/payment/client/invoice are immutable — to change
the amount, delete the refund and create a new one).

| Parameter | Type | Required |
|---|---|---|
| `refund_date` | date | yes |
| `refund_notes` | string | no |

```bash
curl -X PATCH https://your-domain/api/refunds/13 -H "Authorization: Bearer YOUR_API_KEY" \
  -d refund_date=2026-06-20 -d refund_notes="Refund processed"
```

---

## Delete a refund

```
DELETE /api/refunds/{id}
```

Deletes the refund and marks the linked payment `paid` again.

---

## Set tags

```
PUT /api/refunds/{id}/tags
```

Full-set replace (empty clears).

---

## Errors

See [Getting started](/getting-started#errors). Refund-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The refund (or payment) id does not exist. |
| `409 Conflict` | The payment already has a refund. |
| `422 Unprocessable Entity` | Validation failed (missing/invalid payment, missing date). |

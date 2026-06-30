---
title: Payments
description: Record, update, delete, list, tag and refund invoice payments via the Grow CRM API.
---

# Payments

The Payments API provides CRUD access to payments, tags, and the refund sub-resource.

> New to the API? Start with **[Getting started](/getting-started)** (base URL, response
> envelope, errors, pagination) and **[Authentication](/authentication)** (API keys).

> **Identifier note:** payments are addressed by their numeric **`payment_id`**.

> **A payment always belongs to an invoice.** You supply `payment_invoiceid`; the client and
> project are derived from that invoice (never sent). Recording, updating or deleting a payment
> refreshes the parent invoice (paid amount / status).

> **Scope:** online gateway payment flows (Stripe/PayPal/etc.), public thank-you/webhook
> handlers, per-user pinning and bulk actions are out of API scope. The linked refund is
> created/removed here; the **Refunds** resource provides read access.

## The payment object

```json
{
  "id": 146,
  "status": "paid",
  "amount": "150.00",
  "date": "2026-06-20",
  "gateway": "bank",
  "transaction_id": "TXN-1001",
  "notes": null,
  "invoice": { "id": 294, "formatted_id": "INV-0294" },
  "client": { "id": 1, "name": "Acme Inc" },
  "project_id": null,
  "creator_id": 1,
  "dates": { "date": "2026-06-20", "created": "..." },
  "refund": { "has_refund": false, "id": null, "amount": null, "date": null, "notes": null },
  "tags": ["priority"]
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Payment id. |
| `status` | string | `paid` or `refunded`. |
| `amount` | numeric | Payment amount. |
| `date` | date | Payment date. |
| `gateway` | string | Payment method. Defaults to `bank` for a manually recorded payment; any other value is a gateway the caller supplies. |
| `transaction_id` / `notes` | string | Optional. |
| `invoice` | object | The parent invoice (`id`, `formatted_id`). |
| `client` / `project_id` / `creator_id` | object/int | Derived links. |
| `refund` | object | The linked refund (`has_refund` + details), or empty. |
| `tags` | array | Tag titles. |

---

## List / search payments

```
GET /api/payments
```

Query params: `invoice_id`, `client_id`, `project_id`, `gateway`, `amount_min`, `amount_max`,
`date_start`, `date_end`, `search`, `tags[]`, `sort` (`payment_date`,`payment_amount`,`payment_created`),
`order`, `limit`, `page`.

```bash
curl -G https://your-domain/api/payments -H "Authorization: Bearer YOUR_API_KEY" -d invoice_id=294
```

---

## Get a payment

```
GET /api/payments/{id}
```

---

## Record a payment

```
POST /api/payments
```

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `payment_invoiceid` | integer | yes | Existing invoice. |
| `payment_amount` | numeric | yes | Must be greater than 0. |
| `payment_date` | date | yes | |
| `payment_gateway` | string | no | Payment method. Defaults to `bank` when omitted. |
| `payment_transaction_id` | string | no | |
| `payment_notes` | string | no | |
| `send_payment_email` | string | no | `yes` to queue the "payment received" email to the client (as the UI does). Default no. |

```bash
curl -X POST https://your-domain/api/payments -H "Authorization: Bearer YOUR_API_KEY" \
  -d payment_invoiceid=294 -d payment_amount=150 -d payment_date=2026-06-20 -d payment_gateway=Stripe
```

```php
<?php
$ch = curl_init('https://your-domain/api/payments');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Authorization: Bearer YOUR_API_KEY'],
    CURLOPT_POSTFIELDS     => http_build_query([
        'payment_invoiceid' => 294,
        'payment_amount'    => 150,
        'payment_date'      => '2026-06-20',
        // omit payment_gateway to record a manual 'bank' payment
    ]),
]);
$response = json_decode(curl_exec($ch), true);
curl_close($ch);
```

Returns the new payment (`201`, status `paid`).

---

## Update a payment

```
PATCH /api/payments/{id}
```

Updates the editable fields (a payment cannot be moved between invoices). When
`payment_gateway` is omitted the existing method is preserved.

| Parameter | Type | Required |
|---|---|---|
| `payment_amount` | numeric | yes |
| `payment_date` | date | yes |
| `payment_gateway` | string | no |
| `payment_transaction_id` | string | no |
| `payment_notes` | string | no |

```bash
curl -X PATCH https://your-domain/api/payments/146 -H "Authorization: Bearer YOUR_API_KEY" \
  -d payment_amount=150 -d payment_date=2026-06-20 -d payment_gateway=bank
```

---

## Delete a payment

```
DELETE /api/payments/{id}
```

Deletes the payment (and cascades its linked refund, if any) and refreshes the parent invoice.

---

## Set tags

```
PUT /api/payments/{id}/tags
```

Full-set replace (empty clears).

---

## Refund a payment

```
POST /api/payments/{id}/refund
```

Creates the refund for the payment. A refund returns the **full** payment amount (no partials),
marks the payment `refunded`, and is limited to one per payment (`409` if already refunded).

| Parameter | Type | Required |
|---|---|---|
| `refund_date` | date | yes |
| `refund_notes` | string | no |

```bash
curl -X POST https://your-domain/api/payments/146/refund -H "Authorization: Bearer YOUR_API_KEY" \
  -d refund_date=2026-06-20 -d refund_notes="Customer cancelled"
```

Returns the payment with the nested `refund` (`201`).

---

## Remove a refund

```
DELETE /api/payments/{id}/refund
```

Deletes the linked refund and marks the payment `paid` again. Returns the payment (`200`).

---

## Errors

See [Getting started](/getting-started#errors). Payment-specific:

| Status | Meaning |
|---|---|
| `404 Not Found` | The payment id does not exist. |
| `409 Conflict` | The payment already has a refund. |
| `422 Unprocessable Entity` | Validation failed (e.g. missing/invalid invoice, amount not greater than 0, missing date). |

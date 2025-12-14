# 📄 Payment Orchestrator - API Contract (v1)

**Service:** Payment Orchestrator
**Purpose:** Reliability-first backend core
**Version:** v1
**Owner:** _Manjunatha H_
**Last Updated:** 2025-12-13

---

This document defines the **external API behavior** of the Payment Orchestrator service.
It focuses on **correctness, idempotency, and observability**, not UI or provider-specific details.

---

## Design Principle

- APIs are **idempotent by default** where money is involved
- Writes are **asynchronous**; execution happen in wokrers
- Responses are **predictable and stable**
- Backend owns correctness; clients may retry freely
- Errors are explicit and actionable

---

## Authentication

All endpoints require authentication.

- Auth handled via **httpOnly session cookies**
- User identity is resolved server-side
- API does not accept userId from client input

> **Note:** Auth details are out of scope for this document.

---

## Common Headers

### Required

| Header            | Description                            |
| ----------------- | -------------------------------------- |
| `Content-Type`    | `application/json`                     |
| `Idempotency-Key` | Required for money-affecting endpoints |

---

## Common Response Shape

All responses follow a consistent envelope.

```json
{
  "success": true,
  "data": {},
  "error": null
}
```

On error:

```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "PAYMENT_FATLED",
    "message": "Payment could not be processed"
  }
}
```

---

## Endpoints

### 1️⃣ Create Payment

`POST /payments`
Creates a payment intent and enqueues it for processing.

This endpoint is **idempotent**.

---

#### Request Headers

```sql
Idempotency-key: <unique string per logical payment>
```

---

#### Request Body

```json
{
  "amount": "499.00",
  "currency": "INR",
  "description": "Wallet top-up"
}
```

#### Validation Rules

- `amount`
  - required
  - decimal string
  - > 0
- `currency`
  - required
  - supported enum (currently only `INR`)
- `Idempotency-Key`
  - required
  - must be unique per user per logical payment

---

#### Behavior

- Creates a `Payment` record with status `PENDING`
- Enqueues payment for worker processing
- If the same request is retried with the same `Idempotency-Key`, returns the **original response**
- Does **not** call external payment provider synchronously

---

#### Success Respones (200)

```json
{
  "success": true,
  "data": {
    "paymentId": "pay_123",
    "status": "PENDING"
  },
  "error": null
}
```

---

#### Error Responses

| Status | Code                   | Description                          |
| ------ | ---------------------- | ------------------------------------ |
| 400    | `INVALID_INPUT`        | Validation failed                    |
| 409    | `IDEMPOTENCY_CONFLICT` | Same key used with different payload |
| 401    | `UNAUTHORIZED`         | Auth missing or invalid              |
| 500    | `INTERNAL_ERROR`       | Unexpected server failure            |

---

#### Notes

- `PENDING` indicates intent accepted, not execution sucess
- Client should poll or query payment status if needed
- Client is safe to retry on network failure

---

### 2️⃣ Get Payments Status

`GET /payments/{paymentId}`

Returns the current state of a payment.

---

#### Response

```json
{
  "success": true,
  "data": {
    "paymentId": "pay_123",
    "amount": "499.00",
    "currency": "INR",
    "status": "SUCCESS",
    "createdAt": "2025-12-14T07:06:00Z"
  },
  "error": null
}
```

---

#### Possible Status Values

| Status       | Meaning                                 |
| ------------ | --------------------------------------- |
| `PENDING`    | Accepted, awaiting processing           |
| `PROCESSING` | Being executed by worker                |
| `SUCCESS`    | Completed successfully                  |
| `FAILED`     | Failed after bounded retries (terminal) |

---

#### Error Responses

| Status | Code           | Description       |
| ------ | -------------- | ----------------- |
| 404    | `NOT_FOUND`    | Payment not found |
| 401    | `UNAUTHORIZED` | Not authenticated |

---

# 3️⃣ List Failed Payments (Admin)

`GET /admin/payments/failed`

Returns failed payments for operational visibility and recovery.

---

#### Query Params

| Param    | Description                     |
| -------- | ------------------------------- |
| `limit`  | Number of records (default: 20) |
| `cursor` | Pagination cursor               |

---

#### Response

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "paymentId": "pay_456",
        "reason": "PROVIDER_TIMEOUT",
        "failedAt": "2025-12-14T07:31:00Z"
      }
    ],
    "nextCursor": "cursor_abc"
  },
  "error": null
}
```

---

### 4️⃣ Requeue Failed Payment (Admin)

`POST /admin/payments/{paymentId}/requeue`

Manually retries a failed payment.

---

#### Behavior

- Allowed only for `FAILED` payments
- Re-enqueues payment with a new worker attempt
- Does **not** bypass idempotency or ledger rules

---

#### Response

```json
"success": true,
"data": {
  "paymentId": "pay_456",
  "status": "PROCESSING"
},
"error": null
```

---

#### Idempotency Contract

- Idempotency is enforced at the **database layer**
- Key scope: `(userId, idempotencyKey)`
- Redis is **not** the source of truth
- Conflicting payloads with same key return `409`

---

#### Error Codes (Partial)

| Code                   | Meaning                           |
| ---------------------- | --------------------------------- |
| `INVALID_INPUT`        | Request validation failed         |
| `IDEMPOTENCY_CONFLICT` | Key reused with different payload |
| `PAYMENT_FAILED`       | Execution failed                  |
| `UNAUTHORIZED`         | Auth failure                      |
| `INTERNAL_ERROR`       | Unexpected server error           |

---

#### Out of Scope (v1)

- Webhooks
- Refund APIs
- Subscriptions
- Multi-currency FX
- Client-side SDKs

---

#### Versioning
All endpoints are versioned under:
```bash
//api/v1
```
Breaking changes require `/v2`.

---

#### Summary

This API contract prioritizes:
- Idemptency
- Correctness
- Safe retries
- Operational clarity

it is intentionally minimal and conservative to reduce risk in money movement.

---

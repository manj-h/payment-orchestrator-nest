# 📄 `docs/idempotecy-retry.md`

## Idempotency & Retry Strategy

### Purpose

Payment APIs are vulnerable to duplicate requests caused by:

- user duble-submissions
- network retries
- client timeouts
- load balance re-sending requests

This system enforces **idempotent payment creation** and **retry-safe execution**

- payment intent is created **exactly once**
- payment execution happens **at most once**

— regardless of how many times the client retries.

---

## Idempotency Scope

#### Api-level Idempotency

- Each `POST /payments` required an `Idempotency_key` header.
- Idempotency is enforced using a **database uniqueness constraint**, not in-memory states

**Uniqueness rule:**

```
(user_id, idempotency_key)
```

Only one Payment record may exist per `(userId, idempotencyKey) pair.

if a duplicate request is received:

- the existing payment record is returned
- no new processing is triggered

This guarantees **exactly-once intent creation**.

---

### Why Database-Enforced Idempotency

- Survives API restarts
- Works accross multiple instances
- Cannot be bypasses by race conditons
- Becomes the single source of truth

Redis is **not** used as the idempotency authority

---

## Execution Retries (Worker-Level)

### Retry Philosophy

Retires apply **only to execution** never to outcome.

- Retries occur while payment is in `PROGRESSING`
- Retries stops once a payment reaches a terminal state

---

### Retry Conditions

A retry is allowed if:

- the provider request failed due to timeout or network error
- the provider returns a transient failure(eg.,5xx)

A retry is **not allowed** if:

- the provider confirms sucess
- the provider confirms a terminal filure
- wallet updates has already occurred

---

### Retry Limits

- Maximum retries: **3**
- Backoff: **linear backoff** (eg., 5s, 10s, 20s)
- All retires are recorded in metadata for observability

After retry exhaustion:

- payment state transitions to `FAILED`
- no further automatic retries occur

---

## Wallet Safety During Retries

- Wallet updates occur inside a database transaction
- The wallet row is locked using `SELECT ... FOR UPDATE`
- Ledger entries are written **exactly once**
- Retry logic never re-enters wallet mutation after success

This ensures retries cannot currupt balance

---

## Failure Classification

| Failure Type              | Behavior |
| ------------------------- | -------- |
| Network timeout           | Retry    |
| Provider 5xx              | Retry    |
| Provider success          | Finalize |
| Provider terminal failure | Fail     |
| DB constraint voilation   | Abort    |
| Wallet lock failure       | Retry    |

---

## Observability

- Retry count stores on Payment record
- Metrics emitted:
  - retries per payment
  - retry exhaustion rate
  - provider error rate

This allows operators to distinguish:

- flaky providers
- systemic failures
- client misuse

---

## Guarantees

- No duplicate Payment records
- No duplicate wallet debits
- Safe retries under crashes
- Clear failure boundaries

---

## Non-Goals

- Distributed transaction (2PC)
- Exponential retry tuning
- Client-managed idempotency storage

---

## Summary

Idempotency guarantees **intent uniqueness.**
Retries guarantee **execution resilience.**
Together, they ensure correctness without sacrificing reliability.

---

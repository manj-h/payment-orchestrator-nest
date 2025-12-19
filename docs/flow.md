# 📄 docs/flow.md

## Payment Execution Flow (MVP)

### Overview

This document describes the **end-to-end flow** of a payment through the system,from API request to wallet update.

The system is designed so that:

- API handles **intent creation**
- Workers handle **execution**
- Database enforces **correctness**
- Queue isolates **external failures**

Correctness is guaranteed even under retires, concurrency, and provider instability.

---

## High-Level Flow

1. Client submits payment request
2. API creates (or returns) Payment intent
3. Payment is enquired for execlution
4. Worker claims and process payment
5. External provider is called
6. Wallet and ledger are updated atomically
7. Payment reaches a terminal state

---

## Detailed Step-by-Step Flow

### 1️⃣ Client → API: Create Payment

**Request**

```bash
POST /payment
Idempotency-key: <uuid>
```

\*\*API responsibilities

- Validate input
- Enforce idempotency
- Persist payment intent

**Important**

At this stage:

- **No money is moved**
- **No provider is called**

Only intent is recorded.

---

### 2️⃣ API → Database: Persist Payment Intent

- Insert `PAYMENT` with `status = PENDING`
- Enforce uniqueness on `(userId, IdempotencyKey)
- If duplicate request:
  - Return existing `Payment`
  - Do **not** enqueue another job

This quarantees **exactly-once intent creation** even under retries.

---

### 3️⃣ API → Queue: Enqueue Payment Job

- Publish `PaymentId` to queue
- Job payload is minimal(only identifiers)
- API returns response immediately

**Why asynchronous execution?**

- Provider calls are slow and unrealiable
- Api must run fast, deterministic, and retry-safe

---

### 4️⃣ Worker → Database: Claim Payment for Execution

Worker:

- Loads `Payment` by ID
- Performs a guarded state transition:

```nginx
PENDING → PROCESSING
```

- Transaction succeeds only once
  This ensures **only one worker** executes a give payment

---

### 5️⃣ Worker → Provider: Execution Payment

Provider response is classified as:

- **Success**
- **Retryable failure** (timeouts, transient 5xx)
- **Terminal failure** (Validation, insufficent funds)

At this stage:

- **No wallet mutation occurs**
- Only execution outcome is determined

---

### 6️⃣ Worker → Database: Wallet + Ledger transactions

If provided execution succeeds:

A **single database transaction** is executed:

- `SELECT wallet FOR UPDATE`
- Validate sufficient balance
- Debit wallet
- Insert immutable ledger entry
- Update payment status → `SUCCESS`

If **any step fails**, the transaction is rolled back.
This is the **financial safty boundries** of the system.

---

7️⃣ Worker -> Database: Terminal State Handling

If execution fails:

- Increment retry count
- Record failure reason

**If retryable and under retry limit**

- Re-enqueue job with delay

**If retries exhausted or terminal failure**

- Mark payment `FAILED`
- Emit metrics and logs

No wallet or ledger mutation occures on failure paths.

---

Sequence Diagram (Simplified)

```sql
client
  |
  | POST /payment (idempotencyKey)
  v
 API
  | -- insert Payment(PENDING)
  | -- enqueue job
  v
Queue
  |
  v
Worker
  | -- claim Payment (PROCESSING)
  | -- call Provider
  | -- begin DB transaction
  |         lock Wallet
  |         update Wallet
  |         write Ledger
  |         mark SUCCESS
  | -- commit
```

### Failure Handling Summary

| Failure Point        | Result                       |
| -------------------- | ---------------------------- |
| Duplicate request    | Same Payment returned        |
| Worker crash         | Job retried                  |
| Provider timeout     | Retry with backoff           |
| DB error             | Retry or fail safely         |
| Wallet conflict      | Transaction rollback + retry |
| Ledger write failure | Full transaction rollback    |

Retries **never recreate intent** — They only re-create an existing payment.

---

### Design Guarantees
- API is idempotent
- Workers are retry-safe
- Wallet update are automic
- Ledger is immutable
- External failures are isolated

---

### Non-Goals

- Client-side retry orchestration
- Multi-provider routhing
- Parallel execution per wallet

---

### Summary

The system clearly sperates:
- **Internal** — API
- **External** — Worker
- **Correctness — Database

This separation ensures the system remains correct, auditable, and recoverable under real-word failure conditions.



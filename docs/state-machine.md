# Payment State Machine

**Document:** `docs/state-machine.md`
**Scope:** `Defines the lifecycle of a payment from creation to terminal outcome.
**Audience:** Backend engineers, reviewers.

---

# 1. Purpose

This state meachine exists to guarantee **correctness under failure**.

Payment system must tolerate:

- duplicate requests
- network retries
- provider timeouts
- worker restarts
- concurrent wallet access

The state meachine is the **single source of truth** for what payment is allowed to do at any point of time.

---

## 2. States (MVP)

Only **four states** are implemented in v1.

| State        | Meaning                                    |
| ------------ | ------------------------------------------ |
| `PENDING`    | Payment intent recorded, not yet processed |
| `PROCESSING` | Worker has claimed the payment             |
| `SUCESS`     | Funds sucessfully moved and ledger written |
| `FAILED`     | Payment failed after bounded retries       |

> **Note**
> States like `CANCELED` or `EXPERIED` are **future extensions** and not intentionally implemented in MVP.

> **Terminology:**
> The term **state** corresponds to the persisted payment `status` column in the database.

---

## 3. Allowed Transactions

```
PENDING → PROCESSING → SUCCESS
               ↘
                FAILED
```

## 4. Transaction Rules

| From         | To           | Actor  | Condition                                 |
| ------------ | ------------ | ------ | ----------------------------------------- |
| `PENDING`    | `PROCESSING` | Worker | Job dequeued and sucessfully claimed      |
| `PROCESSING` | `SUCESS`     | Worker | Providers sucess + wallet update commited |
| `PROCESSING` | `FAILED`     | Worker | Retry exhausted or non-retry error        |

🚫 **No other transitions are allowed**
Any attempt to transistion aoutside this graph is rejected.

---

## 4. Invariants (Non-Negotable Rules)

This invariants are enforced at **code + database** level.

### 4.1 Exactly-once wallet update

- Wallet balance is updated **only-once**
- Update happens inside a DB transaction
- Wallet row is locked using `SELECT ... FOR UPDATE`

### 4.2 Ledger is immutable

- Ledger entries are **append-only**
- No updates or deletes
- Every successful payment produces exactly one ledger entry

### 4.3 Terminal states are final

- `SUCCESS` and `FAILURE` are terminal
- No retry or state change after terminal state

### 4.4 State drives behavior

- Workers **must check status** before acting
- Duplicate jobs or retries must be no-ops if state is treminated.

---

## 5. Retry Semantics (Bounded & Safe)

Retries are **worker-level** not API-levle

## Retryable Phase

- Retries only occure while the payment in `PROCESSING`
- Example of retryable failures:
  - provider timeout
  - transient network error
  - temperory provider 5xx responses

### Retry Limits

- Max retry limit **N** (configured)
- Backoff: **bounded exponential backoff**
- Retries are idempotent at the DB layer

### Failure Semantics

- After retries are exhausted:

  - payment transitions to `FAILED`
  - error reason is persisted
  - job is moved to dead-letter queue

> `FAILED` means **failed after retries** not "retryable".

---

## 6. Idempotency Interaction

Idempotency is enforced **before** state transitions.

- `(userId + IdempotencyKey) is unique
- Duplicate API calls:
  - return the **existing payment**
  - do not create next state transitions

The state meachine **never compansates** for missing Idempotancy.
The database enforce it.

---

## 7. Concurrency Guarantees

### Worker Claiming

- A worker climes a payment by transitioning:

  ```
  PENDING → PROCESSING
  ```

- This is done Via a condition update:

  ```js
  UPDATE payments
  SET state = 'PROCESSING'
  WHERE id = ? AND state = 'PENDING'
  ```

if zero rows are updated → another worker already climed it.

## Wallet Locking

- Wallet row is locked **inside the same transaction**
- Lock acquisition order is stable:

  1. Payment row
  2. Wallet row

This prevents deadlocks under concurrency.

---

## Failure Philosaphy

This system favors:

- **correctness over throughput**
- **explicit failures over silent retries**
- **manual recovery over hiddin automation**

Failures must be:

- visible
- explainable
- recoverable

---

## 9. Why This Is Delibrately Simple

This MVP intentionally avoids:

- compensation logic
- saga orchestration
- distributed locks

Reason:

> Reliability comes from **few states + strong inveriants**, not from more code.

---

## Summary

- 4 explicity states
- 3 allowed transitions
- terminal states are immutable
- retries are bounded and worker-driven
- correctness enforces at the database layer

This state machine defines the only valid lifecycle for a payment.

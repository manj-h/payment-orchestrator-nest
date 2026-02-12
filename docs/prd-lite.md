# **PRD-LITE — Payment Orchestrator + Ledger (MVP)**

**Service:** Payment Orchestrator
**Purpose:** Reliability-first backend core
**Version:** v1
**Owner:** _Manjunatha H_
**Last Updated:** 2025-12-13

---

## 🧩 Problem Statement

Payment flows are inherently unreliable.
User retries, network failures, and external provider downtime can easily result in **double charges**, **inconsistent wallet balances**,and **Poor auditability**.

We need a backend orchestration layer that guarantees **correctness of money movement** regardless of retries, concurrency, or provider instability.

This service becomes the **source of truth** for payment intent, execution, and wallet state.

---

## 🎯 Goals

1. Prevent double-charging under all retry scenarios
2. Maintain wallet consistency under concurrent requests
3. Guarantee idempotent API behavior
4. Maintain an immutable ledger of all money movements
5. Support safe, bounded retries for transient failures
6. Provide basic operational visibility and recovery hooks

---

## 🚫 Non-Goals

- Not a payment gateway (Stripe/ Razorpay remain providers)
- No UI or customer dashboards
- No FX or multi-currency cnversion logic
- No fraud detection or risk scoring
- No subscription or recurring billing (future)
- No multi-region or distributed consistency guarantees

This system focuses **only on correctness and reliability of the core payment flow**.

---

## 🧠 Design Philosophy

Correctness is prioritized over throughput.

All money movement is serialized per wallet using database-level locking.
External failures are isolated via asynchronous workers, and retries are strictly bounded to avoid cascading failures.

The system is intentionally conservative: **it is better to be slow than wrong**.

---

## 🔁 Payment State Machine

**Implemented in MVP**:

```
PENDING → PROCESSING → SUCCESS
              ↘
              FAILED
```

- `PENDING` — payment intent recorded
- `PROCESSING` — worker is executing the payment
- `SUCCESS` — wallet updated and ledger written
- `FAILED` — execution failed after bounded retries (terminal)

**Future states (explicitly not implemented in v1)**

- `cancelled`
- `Expired`

These are documented for clarity but intentionally excluded from the MVP.

---

## ✅ Acceptance Criteria (Invariants)

1. `POST /payment` is idempotent using `(userId + idempotencyKey)`
2. Duplicate requests never result in duplicate charges
3. Payment transitions strictly follw the defined state meachine
4. Wallet updates occur inside a DB transaction with row-level
5. Ledger entries are immutable and append-only
6. Each successful payment produces exactly one ledger entry
7. Worker retries are safe and bounded
8. Failed payment are visible via admin APIs
9. System remains consistent during provider downtime

---

## 🧱 Core Concepts

**Idempotency**

- Enforced at the database layer
- Unique constraint on `(userId, idempotencyKey)
- Repeated requests return the original payment result

**Wallet Consistency**

- Wallet rows locked using `SELECT ... FOR UPDATE`
- Prevents race conditions and double deductions
- Locks acquired in a stable order to avoid deadlocks

**Ledger**

- Append-only, immutable record of balance changes
- Primary audit trail for financial correctness
- Never updated or deleted

---

## 📐 Design Targets (Prototype / Single Node)

These are **design intents**, not production SLOs.

- `POST /payments` remains low-latency (writes intent only)
- Worker designed to handle ~20 jobs/sec in local testing
- Queue + retries isolate provider outages for several minutes
- Wallet balances stored as `DECIMAL(18,4) (no floating-point math)

---

## ⚠️ Risks & Mitigation

| Risk                      | Mitigation                  |
| ------------------------- | --------------------------- |
| Double charge on retries  | DB-level idempotency        |
| Concurrent wallet updates | Row-level locking           |
| Provider downtime         | Queue-based async execution |
| Partial failures          | Atomic DB transactions      |
| Infinite retries          | Bounded retry policy        |
| Debug complexity          | Ledger + structured logs    |

---

## 🔍 Operational Visibility

- Admin endpoint to list failed payments
- Manual requeue capability for failed jobs
- Basic metrics exposed:
  - request count
  - retry count
  - queue depth
  - failure rate

---

## 🚀 Rollout & Rollback (MVP Scope)

**Rollout**

1. Apply database migrations
2. State API service
3. Start worker
4. Execute a smoke-test payment
5. Verify wallet and ledger correctness

**Rollback**

1. Disable payment initiation
2. Pause worker consumption
3. Inspect inconsistent records manually
4. Roll back migrations if required

---

## 🔗 Dependencies

- PostgreSQL — transactional system of record
- Redis — queue and operational counters
- BullMQ — woker execution and retry engine
- Payment provider sandbox (Stripe / Razorpay)
- Docker Compose — local runtime
- GitHub Actions - CI

---

## 📃 📃 Summary

This MVP delivers the **minimum reliable core** required to move money safely.

It deliberately trades feature breadth for:

- correctness
- predictability
- debuggability

This system is small by design, but hardened against real-world failure modes.

---

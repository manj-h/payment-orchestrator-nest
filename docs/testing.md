# 📄 `docs/testing.md`

## Testing Strategy (MVP)

This document defines the **testing approach** for the Payment Orchestrator MVP.

The goal of testing here is **correctness under failure and concurrency**, not exhaustive coverage or performance benchmarking.

---

## Testing Philosophy
This system handles **money**, so testing prioritizes:
- **Correctness over coverage**
- **Concurrency safety over happy paths**
- **Failure handling over edge UI cases**
- **Deterministic outcomes under retires**

We do **not** aim for 100% test coverage.
We aim to **prove invariants**

---

## Core Invariant (Must Hold)

If these hold, the system is considered safe:

1. Payments are **idempotent**
2. Wallet balance cannot be corrupted under concurrency
3. Ledger entries are **immutable and correct**
4. Failed execution do **not** mutate money
5. Retries do **not** double-charge

If these hold, the system is safe

---

## Test Levels

### 1️⃣ Unit Tests(Light)

**Purpose**
Validate isolated logic without external dependencies.

**Scope**
- Payment state transition rules
- Retry counter logic
- Input validation
- Idempotency key normalization
- Error classification (retryable vs terminal)

**What we intentionally skip
- Database locking behavior
- Provider integration
- Queue behavior

These belong to integration tests.

---

### 2️⃣ Integration Tests (Core)

**Purpose**
Verify system behavior with **real infrastructure components.

**Componenets involved**
- PostgreSQL
- Redis
- Queue  (BullMQ)
- API service
- Worker service

**Key scenarios**
- Create payment → job enqueued
- Worker processes job → wallet update
- Ledger entry written atomically
- Payment reaches SUCCESS

All integration tests run against a **local Docker Compose setup.

---

### 3️⃣ Concurrency Test (Most Important)

This is the **single most valuable test** in the entire project.

#### Scenario: Idempotency Under Concurrent Requests

**Test**
- Fire **N concurrent POST /payment**s requests
- Same `userId`
- Same `idempotencyKey`

**Expected Result**

- Exactly **one** Payment record created
- Exactly **one** job enqueued
- Wallet balance reduced **once**
- Exactly **one** ledger entry

**Important**
> **This test must be executed multiple times to ensure results are deterministic and not timing-dependent.**

**Why this matters**

If this test passes consistently, the system is safe.

This test demonstrates:

- DB-level idempotency
- Correct transaction boundaries
- Concurrency-safe wallet updates

---



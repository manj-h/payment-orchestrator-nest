# Payment Orchestrator — Data Model (MVP)

This document defines the **minimum set of tables** required to guarantee correctness, idempotency, and auditability for payment execution and wallet balance management.

The database is the **single source of truth**.

---

## 🧠 Design Principle

- Money movements must be **auditable and immutable**
- Wallet balance update must be **serialized**
- Idempotency must be **enforced at DB level**
- State transitions must be **explicit and constrained**
- Schema must support **safe retries and recovery**

Correctness > Throughput > convenience

---

## 🧩 Core Tables Overview

```
User ──┐
       ├── Wallet  ──┐
       │             ├── LedgerEntry
       └── Payment ──┘

```

## 🧱 Tables

### 1️⃣ `users`

> Represents an authenticated account.
> Authentication is assumed to be external to this service.

```sql
users (
  id           UUID PRIMARY KEY,
  email        TEXT UNIQUE NOT NULL,
  created_at   TIMESTAMP NOT NULL
)
```

**Notes**

- Minimal on purpose
- User identity always comes from auth context, not API input

---

### Wallets

> Represents the **current balance** for a user.
> This table is **mutable and protected** via row-level locks.

```sql
wallets(
  id             UUID PRIMARY KEY,
  user_id        UUID UNIQUE NOT NULL REFERENCES users(id),
  balance        DECIMAL(18,4) NOT NULL DEFAULT 0,
  currency       TEXT NOT NULL,
  created_at     TIMESTAMP NOT NULL,
  updated_at     TIMESTAMP NOT NULL
)
```

**Key Invariants**

- One wallet per user
- Balance update **only inside DB transactions**
- Accessed with `SELECT ... FOR UPDATE`

**Why this exsts**

- Fast balance check
- Ledger is history, wallet is snapshot

---

### 3️⃣ `payments`

> Represents **intent to move money**
> This is the core orchestration table.

```sql
payments (
  id                UUID PRIMARY KEY,
  user_id           UUID NOT NULL REFERENCES users(id),
  amount            DECIMAL(18,4) NOT NULL,
  currency          TEXT NOT NULL,
  status            TEXT NOT NULL,
  idempotency_key   TEXT NOT NULL,
  provider_ref      TEXT,
  failure_reson     TEXT,
  created_by        TEXT NOT NULL, -- api | admin
  created_at        TIMESTAMP NOT NULL,
  updated_at        TIMESTAMP NOT NULL,

  UNIQUE( userid, idempotency_key)
)
```

**Status Values (MVP)**

- `PENDING` — accepted, not executed
- `PROCESSING` — worker picked up
- `SUCCESSED` — money moved, ledger written
- `FAILED` — execution failed after bounded retries
  > `CANCLED / EXPERIED` are **feature state**, not implemented in v1.

**Why this exists**

- Seperate _intents_ from _execution_
- Enables idempotency
- Allow safe retries

---

### 4️⃣ `ledger_entries`

> Immutable record for every wallet-effecting event.

```sql
ledger_entries (
  id               UUID PRIMARY KEY,
  wallet_id        UUID NOT NULL REFERENCES wallets(id),
  payment_id       UUID REFERENCES payments(id),
  ammount          DECIMAL(18,4) NOT NULL,
  balance_after    DECIMAL(18,4) NOT NULL,
  type             TEXT NOT NULL, -- DEBIT | CREDIT
  created_at       TIMESTAMP NOT NULL
)
```

**Rules**

- INSERT only(never updated or deleted)
- Written **one after wallet update succeeds**
- `balance_after` allows audit without replay

**Why this exits**

- Finance audit trial
- Debugging discrepancies
- Compliance-ready by design

---

## 🔒 Constraints & Guarantees

| Guarantee                 | Enforced By                      |
| ------------------------- | -------------------------------- |
| No double charge          | UNIQUE(user_id, idempotency_key) |
| Serialized wallet updates | SELECT FOR UPDATE                |
| Immutable history         | Ledger INSERT-only               |
| Safe retries              | Payment state machine            |
| Consistent currency       | Wallet + Payment validation      |

---

## Migration Strategy

All schema changes follow **expand → backfill → switch → contract.**

### Example: Adding `currency` to wallets

#### Step 1 — Add nullable column

```sql
ALTER TABLE wallets ADD COLUMN currency TEXT;
```

#### Step 2 — Backfill existing rows

```sql
UPDATE wallet SET currency = 'INR' WHERE currency IS NULL;
```

#### Step 3 — Enforce constraint

```sql
ALTER TABLE wallets ALTER COLUMN currency SET NOT NULL;
```

#### Step 4 — Update application logic

- Validate currency on create
- Reject missmatch

---

## 🧪 Concurrency Test (Must-Have)

- Spawn N concurrent `POST /payments` requests
- Same `userId + Idempontency_key`
- Assert:

  - Only one `payments` row
  - Only one `ledger_entries` row
  - Wallet balance updated once

  > This test proves the **entire model works.**

---

# 📄 `docs/risks.md`

## Ris Assessment & Mitigation (MVP)

This document identifies the **Primary technical risk** in the Payment Orchestrator MVP and outlines **practical mitigation** approches for a **single-service, single-region** system.

The goal is **correctness first**, not zero risk.

---

## 1️⃣ Concurrency & Double-Check Risk (Critical)

### Risk

Multiple requests or workers could attempt to process the same payment concurrently, leading to:

- Double wallet debits
- Duplicate ledger entires
- Inconsistent balances

This risk commonly occures under:

- client retires
- Network error
- Worker restarts

### Mitigation

- Enforce **idempotency at the database layer** using a unique constraint on (UserId, IdempotencyKey)
- Guard state transition using conditional update (`PENDING → PROCESSING`)
- Lock wallet rows using `SELECT ... FOR UPDATE` before mutating balances
- Ensure Wallet + ledger updates in **single database transaction**

### Why this is acceptable

The database becomes the **source of truth**, eliminating reliance on in-memory state or queue guarantees.

---

## 2️⃣ Payment Provider Downtime / Timeout (High)

### Risk

External payment providers may:

- Timeout
- Return transient 5xx error
- Become temporarily unavailable

This can delay or interrupt payment execution.

### Mitigation

- Isolate provider calls in **background workers**
- Apply **strict request timeouts**
- Classify failures as:
  - retryable (network issue, timeouts)
    -terminal (Validation errors, provider rejection)
- Retry only retryable failures using **bounded attempts with delay**
- Never retry by recreating the payment intent

### Why this is acceptable

API responsiveness and correctness are preserved even when providers are unstable.

---

## 3️⃣ Database Migration Risk (Medium)

### Risk

Schema changes on financial tables may:

- Lock tables
- Break existing queries
- Corrupt financial state if applied incorrectly

### Mitigation

- Follow **incremental migration stratagy:**
  - add nullable columns
  - backfile data
  - update application logic
  - enforce constraints later
- Avoid destructive migrations in a signle step
- Use strong numeric type (`DECIMAL`) for ll monetary values

### Why this is acceptable

This minimizes blast radius while maintaining backword compatibility during rollout.

---

## 4️⃣ Worker Crash or Partial Execution (High)

### Risk

A worker may crash:

- After calling the payment provider
- Before updating wallet and ledger
- During database transaction execution

This could leave execution incomplete

### Mitigation

- Treat provider execution and wallet mutation as **separate phases**
- Ensure wallet mutation always happens inside a **single transaction**
- The system never assumes provider success until the wallet transaction commits
- Rely on job retries for crashed workers
- Payment state transitions prevent duplicate execution

### Why this is acceptable

Retries are safe because **side effects are guarded by idempotent state and database locks.**

---

## 5️⃣ Secreats & Credential Leakes (Medium)

### Risk

Improper handling of:

- Provider API keys
- Database credentials

could lead to security compromise.

### Mitigation

- store secrets in environment variables
- Never commit secrets to source control
- Maintain `.env.example` for documentation
- Mask sensitive feelds in logs
- Restrict provider keys to minimum required permissions

### Why this is acceptable

This align with standard backend security practices for early-state systems.

## 🚫 Out-of-Scope Risks (Explicitly Differed)

The following risks are acknowledged but **intentionally excluded for the MVP**:

- Multi-region consistency
- Fraud detection
- Advanced reconciliation tooling
- Automated incident response

This are differed to keep the system focused and correct.

---

## 📃 📃 Summary

This MVP prioritizes:

- **Correctness over throughput**
- **Database enforced guarantees**
- **Isolation of external failures**
- **Bounded retries instead of best-effort execution**

Thes mitigations are sufficient for a **single-node, production-grade prototype** and align with real-world fintech backend practices.

---

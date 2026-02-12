# 📄 `docs/rollback.md

## Rollback & Recovery Plan (MVP)

This document defines **rollback and recovery procedure** for the Payment Orchestrator MVP.

The object is to **stop further damage, preserve financial correctness,** and **recovery determistically** — not to guarantee zero downtime.

Rollback decissions are made by the **on-call engineer or tech lead,** not automated systems.

---

## Guiding Principles

- Stop writes first
- Never mutate financial history
- Prefer disabling flows over hot-fixing logic
- Correctness over availability
- Manual recovery is acceptable for MVP

---

## 1️⃣ API-Level Rollback

### Scenario

A bug discovered in `POST /payment` that could:

- Create incorrect payment instents
- Enqueue malformed jobs
- Accept invalid input

### Action

- Disable payment creation:
  - Feature flag (if available), or
  - Return `503 Service Unavailable`
- Keep read-only endpoints active:
  - `GET /payment/:id`
  - Admin inspection endpoints
- Investigate and fix the issue offline before re-enabling writes

### Why this works

- Prevents creation of new financial intents
- Existing data remains intact
- No partial or risky rollback is required

---

## 2️⃣ Worker-Level Rollback

## Scenario

Worker logic is faulty, causing:

- Incorrect state transitions
- Unsafe retry behavior
- Wallet update bugs

## Action

- Stop worker processes:
  ```bash
  pm2 stop payment-worker
  # or
  docker stop payment-woker
  ```
- Leave API running **if intent creation is safe**
- New payments may remain in `PENDING` state until execution resumes
- Inspect:
  - Payment in `PROGRESSING`
  - Failed jobs
  - Wallet balances
- Deploy fixed worker version
- Resume job processing gradually

## Why this works

- Prevents further wallet mutations
- Jobs remain safely queued
- Execution can resume in controlled, observable manner

---

## 3️⃣ Queue Control (Job-Level Recovery)

## Scenario

Jobs are malformed, retrying endlessly, or causing repeated failures.

## Action

- Pause queue consumption
- Move problematic jobs to a dead-letter queue
- Fix job payload or worker logic
- Manual requeue jobs **only after verification**

## Important Rule

Jobs **must be replayable without side effects.**

If unsure:

> **Do not replay — inspect first.**

---

## 4️⃣ Database Migration Rollback

## Scenario

A schema change causes:

- Runtime errors
- Lock contention
- Incorrect data reads

## Safe Migration Pattern

All schema changes follow:

1. Add new column as nullable
2. Backfill existing data
3. Switch application reads/writes
4. Verify correctness
5. Drop old column later

## Rollback Steps

- Revert application to prevent stable version
- Stop workers
- Roll back migration using a verified down script
- Resume system only after validation

## Hard Rule

Never delete or alter:

- Ledger entries
- Historical balances

---

## 5️⃣ Data Integrity Incident (Highes Severity)

### Scenario

A wallet balance inconsistency is detected.

## Action

- Immediately stop workers
- Disable payment execution
- Compare:
  - Wallet balance
  - Some of ledger entires
- Recalculate wallet balance using ledger as the **source of truth**
- Apply corrective update in a controlled transaction
- Log and document the incident and root cause

## Why this works

The ledger is **immutable and authoritative**
The wallet is a derived snapshot.

All corrective actions must be **logged and auditable.**

---

## Waht We Do NOT Attempt in MVP

The following are intentionally out of scope:

- Automated rollback
- Live traffic shifting
- Multi-region failover
- Self-healing workflow
- Zero-downtime guarantees

These are deferred to keep the system **simple, correct, and controllable**

---

## 📃 Summary

Rollback in this system is about **damage containment**, not reversibility.

By:

- stopping writes early,
- isolating execution,
- preserving immutable history, and
- recovering from the ledger,

The system remains correct even under failure.

---

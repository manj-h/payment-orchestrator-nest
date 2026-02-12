# 📄`docs/runbook.md`

## Operational Runbook (MVP)

This runbook describes **how to operate, inspect, and recover** the Payment Orchestrator in production-like environments.

The systme is intentionally **manual-fit**

Correctness and auditability are prioritized over automation and uptime guarantees.

Rollback and recovery decisions are made by the **on-call engineer or tech lead**, not automated systems.

---

## 🎯 Purpose

This runbook exists to help engineers:

- Verify system health
- Safely stop exection when issue occure
- Inspect stuck or failed payments
- Recover execution without risking double charges
- Restore correctness after incidents

> When money is involved, **controlled recovery beats fast recovery**

---

## 🩺 Health Checks

### API Health

**Endpoint**

```bash
GET /health
```

**Ckecks**

- API process is running
- Database connectivity
- Redis connectivity
- Queue client reachable

**Expected Response**

```json
{
  "status": "ok",
  "db": "connected",
  "redis": "connected",
  "uptimeSeconds": 12345
}
```

If `/health` fails → disable payment creation immediately.

---

### Worker Health

Verify worker process:

```bash
pm2 status
# or
docker ps
```

Worker must be:

- Running
- Polling the queue
- Not crashing or retrying endlessly

---

## 📊 Key Signals to Monitor

Only the **minimum** required signals are tracked for MVP.

### API

- Error rate(4xx /5xx)
- Sudden latency spikes
- Validation failures

### Workers

- Queue Depth
- Retry count per payment
- Failed payment count

### Database

- Wallet row lock contention
- Slow queries on wallet / ledger tables

---

## 🚨 Common Operational Scenarios

### 1️⃣ Payment API Errors Increasing

**Symptomes**

- `POST /payments` return errors
- Clients report failures

**Actions**

1. Inspect API logs
2. Check recent deploys or config changes
3. If unsafe:

- Disable payment creation (feature flag or 503)

4. Keep read-only endpoints enabled
5. Fix offline, redeploy, then re-enable writes

**Reasoning**
Prevents creation of unsafe financial intents while preserving existing data.

---

### 2️⃣ Queue Backlog Growing

**Symptomes**

- Queue depth increases
- Worker lag
  **Actions**

1. Inspect worker logs
2. Check provider latancy or outage
3. Pause queue if failures repeat
4. Scale workers **only after confirming logic is correct**

**Rules**

> Never scale broken logic.

---

### 3️⃣ Payment Stuck in PROGRESSING

**Symptoms**

- Payment remains PROGRESSING longer than expected
  **Action**

1. Inspect payment record
2. Check worker crash logs
3. Verify provider response
4. If safe → manually requeue
5. If unsafe → **do nothing and investigate**

**Rule**
Never manually change payment state without understanding wallet impact.

---

### 4️⃣ Failed Payment Accumulating

**Symptomes**

- FAILED payment increasing
  **Actions**

1. Inspect failure reasons
2. Identify pattern (timeouts, validation, balance)
3. Retry only after fixing root cause
4. Leave terminal failures untouched

No wallet or ledger changes are required

---

### 5️⃣ Wallet Balance Mismatch (Critical)

**Symptomes**

- Wallet balance != sum of ledger entries
  **Actions**

1. Immediately stop workers
2. Disable payment execution
3. Recaluclate balance from ledger
4. Apply corrective update in a controlled transaction
5. Log and document the incident
6. Resume execution only after validation

**Ledger is the source of truth. Wallet is derived.**

---

## 🔁 Restart Procedures

### Restart API

```bash
pm2 restart api
#or
docker restart api
```

Safe because API is stateless.

---

## Restart Worker

```bash
pm2 restart payment-worker
#or
docker restart payment-woker

```

Safe because jobs are idempotent and retry-safe.

---

## 🔄️ Requeuing Jobs(Manual)

Requeue **only if:**

- Root cause is understood
- Payment is not SUCCESS
- Wallet mutation has not occurred

if unsure → **do not requeue**

---

## 📁 Logs to Inspect

### API Logs

- Validation faliures
- Idempotency conflicts
- Unexpected 5xx errors

### Worker Logs

- Provider request / response
- Retry attempts
- State transition failures

### Database Logs

- Lock waits on wallet table
- Transaction rollbacks

---

## 🚫 Explict Non-Goals (MVP)

The following are intentionally **out of scope**.

- Auto-healing
- Automatic rollbacks
- Infinite retries
- Distributed tracing
- PagerDuty / alert escalation systems
- Zero-downtime quarantees

Manual control is perferred for correctness.

---

## 🧠 Operating Principle

> When uncertain, stop exeution first — investigate secound — recover deliberately.

---

## ✅ Summary

This runbook enables operators to:

- Detect issues early
- Stop damage quickly
- Recovery deterministically
- Preserve financial correctness

It is **simple, explicit, and human-driven** — appropriate for an MVP handling real money.

---

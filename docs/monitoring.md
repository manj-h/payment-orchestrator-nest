# 📄 `docs/monitoring.md`

## Monitoring & Observability (MVP)

This document defines the **minimum monitorng signals** required to operated the Payment Orchestrator safely in production.

The goal is **early detection of correctness and reliability issuesd**, not deep analytics or SRE-grade observability.

---

## Monitoring Philosophy

This system prioritizes **financial correctness over availability**.

Monitoring focuses on answering three questions:

1. **Is money safe?**
2. **Is the system making forward progress?**
3. **Can operators detect and recover from failures quickly?**

We intentionally avoid complex monitoring stacks in the MVP.

---

## Core Signals to Monitor (Non-Negotiable)

### 1️⃣ Payment Health Metrics (Primary Signals)

These metrics indicate whether the system is behaving correctly.

**Payment Created**

- Count of `POST /payments`
- Detects traffic spikes, abuse, or client retry storms

**Payments Succeded**

- Count payments reaching `SUCCESS`

**Payments Failed**

- Count of payments reaching terminal `FAILED`

**Red Flags**

- Sudden increase in failures
- Payment remaining in `PROCESSING` beyond expected duration

---

### 2️⃣ Queue & Worker Metrics (Execution Health)

These metrics indicate whether execution is progressing.

**Queue Depth**

- Number of pending jobs in the payment queue

**Alert Condition**

- Queue depth growing stedily for > N minutes

**Retry Count**

- Average retries per payment

**Alert Condition**

- Retry count exceeding expected bounds

**Worker Throughput**

- Approximate jobs processed per secound

**Interpretation**

A sustained drop may indicate:

- Provider slowness
- Database contention
- Worker crash or stall

---

### 3️⃣ Latency Metrics (Intent Creation Only)

Latency matters primarily at **intent creation**, not execution.

**API Latency**

- `POST /payments` p95 latency
  **Expected**
- Low and stable (API only records intent)

**Red Flag**

- Latency spikes → DB contention, lock pressure, or degraded dependencies

--

### 4️⃣ Error Metrics (Failure Detection)

**API Error Rate**

- Track 4xx vs 5xx separately
- Operational focus is on **5xx** (system faults)

**Worker Errors**

- Provider timeouts
- DB transaction failures
- Lock wait timeouts

**Rule**
Every error **must log**:

- `paymentId`
- `requestId`

This is mandatory for incident debugging

---

## Loging Requirements (Minimum)

All logs must be **structured (JSON)** and include:

- `timestamp`
- `level`
- `message`
- `requestId`
- `paymentId` (if applicable)
- `workerId` (for wrokers)

**Example**

```json
{
  "level": "error",
  "message": "Wallet update failed",
  "paymentId": "pay_123",
  "requestId": "req_456",
  "error": " Insufficient balance"
}
```

Structured logs enable fast root-cause analysis **without dashboards**.

---

## Health Checks

### API Health

`GET /health` must report:

- API availability
- Database connectivity
- Redis connectivity

### Worker Health

- Periodic heartbeat log
- Timestamp of last processed job

**Operator Expectation**
If hartbeat stops, the worker is considered unhealthy.

---

## Alerts (Simple & Manual-Friendly)

Allerts are defined only when correctness is at risk

### Critical Alerts

- Payments stuck in `PROGRESSING` beyond threshold
- Ledger write failures
- Wallet vs ledger inconsistencies

### Warning Alerts

- Queue depth steadily increasing
- Retry count trending upward
- Provider timeout rate increasing

Alerts may initially be **log-based or manual checks.**
Automation can be added later.

---

## Explictly Out of Scope (MVP)

The following are intentionally excluded:

- Distributed tracing
- Per-endpoint dashboards
- SLA / SLO burn rates
- Auto-scaling signals
- Mult-region replication lag

These are deferred to avoid premature complexity.

---

## Operator Checklist (When Something Feels Wrong)

1. Check queue depth
2. Inspect payments in `PROCESSING`
3. Review worker error logs
4. Verify wallet vs ledger consistency
5. Pause workers if correctness is at risk
6. Investigate before resuming execution

---

## Summary

Monitoring in this system exists to:

- Detect financial risk early
- Provide clear recovery signals
- Avoid noisy or misleading metrics
- Support **manual, deterministic recovery**

If monitoring indicates risk, operators should **trust it**.
**Correctness always wins over speed**

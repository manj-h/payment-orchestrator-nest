# Payment Orchestrator Core (NestJS Prototype)

> **Project Status**
>
> This repository contains the original implementation of the Payment Orchestrator built with NestJS, TypeScript, Prisma, PostgreSQL, and BullMQ.
>
> Development on this implementation has been intentionally stopped after shifting the project to **Java and Spring Boot**. The architecture, design documents, and engineering decisions in this repository remain the foundation for the active implementation.
>
> **➡️ Active implementation:** `payment-orchestrator` (Java + Spring Boot)

# **Payment Orchestrator Core**

A reliability-first backend service that orchestrates payment execution, ensures wallet consistency under concurrency, and maintains an immutable ledger of money movements.

---

## 🚀 Purpose

Payment flows fail in the real world — users click twice, gateways timeout, networks drop.
A production system must remain _correct_, even when everything else breaks.

This service provides:

- **Idempotent payment initiation**
- **A strict, auditable payment state machine**
- **Exactly-once wallet updates under concurrency**
- **Immutable ledger entries**
- **Safe retry behavior via queue + worker**
- **Operational visibility (logs, metrics, failed jobs)**

> **Providers may fail — this system must not.**

---

## 📦 Current Scope (MVP)

- `POST /payments` with idempotency (userId + idempotencyKey)
- Payment lifecycle: `PENDING → PROCESSING → SUCCESS | FAILED`
- Worker-driven execution (BullMQ + retries)
- Row-level wallet locking (`SELECT ... FOR UPDATE`)
- Immutable ledger creation
- Admin endpoint for failed payments
- Core metrics: queue depth, retry count, p95 latency
- Structured logging + correlation IDs
- Basic auth + admin-only routes

Full PRD → [`docs/prd-lite.md`](docs/prd-lite.md)

---

## 🧩 Key Guarantees

1. **No double-charging** (idempotent + locked writes)
2. **Wallet consistency** under concurrent requests
3. **Exactly-once ledger entries**
4. **Safe retries** without duplication
5. **Well-defined state transitions**
6. **Observability**: logs, metrics, health checks

---

## 🛠 Tech Stack

### Backend

- **Node.js** + **TypeScript**
- **NestJS / Express** (modular API + workers)
- **PostgreSQL** (source of truth)
- **Prisma ORM** (type-safe DB access)

### Infra

- **Redis + BullMQ** (queue + retry engine)
- **Docker Compose**
- **GitHub Actions CI**

### Observability

- Structured logging (Pino/Winston)
- requestId correlation
- `/health` endpoint
- `/metrics` endpoint (queue depth, retry count, p95 latency)

---

## 📁 Repository Structure

```
docs/               → Product & technical specifications
src/                → API, workers, queues, domain logic
prisma/             → Database schema & migrations
docker/             → Dockerfiles + compose setup
tests/              → Integration + concurrency tests
.env.example        → Environment variables (safe template)
```

---

## 🔐 Authentication

- Lightweight JWT auth for user-facing endpoints
- Admin guard for operational endpoints
- No frontend token storage — backend-managed cookies for security

---

## 🧪 Tests

- Concurrency test: multiple POSTs with same idempotencyKey
- Worker processing test
- Failure mode tests:

  - provider timeout
  - DB locked row
  - retries

---

## License

This project is licensed under the **MIT License**.
See the [LICENSE](./LICENSE) file for details.

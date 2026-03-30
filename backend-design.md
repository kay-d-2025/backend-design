# Backend System Design: User Wallet with Deposit & Withdrawal via Revio

**Author:** [Your Name]
**Date:** March 2026
**Version:** 1.0

---

## Table of Contents

1. [Tech Stack](#1-tech-stack)
2. [System Architecture](#2-system-architecture)
3. [Database Design](#3-database-design)
4. [Deposit & Withdrawal Flow](#4-deposit--withdrawal-flow)
5. [Webhook Handling](#5-webhook-handling)
6. [Security Considerations](#6-security-considerations)
7. [Deployment & Hosting](#7-deployment--hosting)
8. [Potential Failure Points & Mitigations](#8-potential-failure-points--mitigations)
9. [Bonus: Dashboard & Scaling](#9-bonus-dashboard--scaling)

---

## 1. Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Language | Node.js (TypeScript) | Strong async model, large ecosystem, type safety reduces runtime errors in financial logic |
| Framework | Express.js | Lightweight, well-understood, easy to reason about middleware chains |
| Database | PostgreSQL | ACID compliance is non-negotiable for financial data; mature support for row-level locking and serialisable transactions |
| ORM | Prisma | Type-safe queries, clean migration tooling, good DX for schema evolution |
| Auth | JWT (access) + refresh tokens | Stateless, scalable, standard |
| Secret Management | AWS Secrets Manager | Centralised, auditable, avoids secrets in env files or version control |
| Hosting | AWS (ECS Fargate + RDS) | Managed containers, managed database, good fit for this scale |
| CI/CD | GitHub Actions | Straightforward pipeline, integrates well with AWS deployments |
| Monitoring | Datadog | Logs, APM, and alerting in one place |

**On architecture style:** I chose a modular monolith over microservices. The scope is well-defined and a microservices split would add overhead (service discovery, distributed tracing, inter-service auth) without meaningful benefit at this stage. If usage grows significantly, the wallet logic is a natural candidate to extract.

---

## 2. System Architecture

### 2.1 Overview

The system is a modular monolith deployed as a containerised REST API. It exposes endpoints consumed by the frontend, makes outbound calls to Revio, and receives inbound webhooks from Revio on a dedicated route.

```
┌─────────────────────────────────────┐
│         Client (Browser/App)        │
└────────────────┬────────────────────┘
                 │ HTTPS
                 ▼
        ┌─────────────────┐
        │   API Gateway   │  (rate limiting, TLS termination)
        └────────┬────────┘
                 │
        ┌────────▼────────┐
        │   Express API   │
        │  (ECS Fargate)  │
        └──┬──────────┬───┘
           │          │
  ┌────────▼───┐  ┌───▼──────────────┐
  │ PostgreSQL │  │      Revio       │
  │   (RDS)    │  │ Payment Provider │
  └────────────┘  └──────────────────┘
```

### 2.2 Authentication & Authorisation

- Passwords hashed with `bcrypt` (cost factor 12)
- On login, issue a short-lived JWT access token (15 min) and a refresh token (7 days, httpOnly cookie)
- Refresh tokens are stored hashed in the database and can be revoked individually
- Every protected route validates the JWT via middleware; the payload carries `userId` and `role`
- All wallet operations are scoped to the `userId` extracted from the token — the request body `userId` is never trusted
- Admins can view any user's transactions but cannot initiate operations on their behalf

---

## 3. Database Design

### 3.1 Schema Overview

The schema has five tables: `users`, `wallets`, `transactions`, `webhook_events`, and `refresh_tokens`.

**users** stores credentials and role. Each user has an email, a hashed password, and a role of either `user` or `admin`.

**wallets** has a one-to-one relationship with users. It holds the current balance and currency. The balance is stored as a fixed-precision decimal — never a float — and a database-level constraint prevents it from going negative. This is the authoritative source for a user's balance.

**transactions** is an append-only ledger. Every deposit and withdrawal is recorded here with a type (`deposit` or `withdrawal`), a status (`pending`, `completed`, or `failed`), an amount, and a reference ID from Revio once one exists. Each record has an idempotency key — a unique value that prevents the same operation from being written twice, regardless of retries. A metadata field stores a snapshot of the raw Revio payload for auditability without tightly coupling the schema to Revio's response structure.

**webhook_events** logs every inbound webhook from Revio. Each record captures the event type, the raw payload, processing status, attempt count, and any error encountered. This exists purely for auditability and debugging — it is not the source of truth for transaction state.

**refresh_tokens** stores hashed refresh tokens against a user, with an expiry timestamp and a revocation flag.

### 3.2 Indexes & Constraints

Indexes are placed on the foreign keys used in frequent lookups: wallet by user, transactions by wallet, transactions by Revio reference, and refresh tokens by user. The idempotency key on transactions and the token hash on refresh tokens are both unique indexes by nature of their constraints.

The balance non-negative constraint on `wallets` is a last line of defence — application logic enforces this first, but the database will reject any update that would take a balance below zero regardless.

---

## 4. Deposit & Withdrawal Flow

### 4.1 Deposit

**User perspective:** Enter an amount → redirected to Revio-hosted payment page → complete payment → redirected back, balance updated.

**Backend sequence:**

When the user submits a deposit request, the API creates a transaction record in `pending` status and calls Revio to initiate a payment. Revio returns a hosted payment URL which the API passes back to the frontend. The user is redirected to complete payment on Revio's side.

Once the user pays, Revio sends a webhook to the API. The API verifies the webhook (see §5), then updates the transaction to `completed` and credits the wallet balance. Both the transaction update and the balance credit happen inside a single database transaction with a row-level lock on the wallet — this ensures no other operation can read or modify that wallet's balance concurrently, preventing any race condition between simultaneous deposits.

### 4.2 Withdrawal

**User perspective:** Enter amount and payout details → submit → see pending status → notified on completion.

**Backend sequence:**

```
User                  API                        Revio
 │                     │                            │
 │── POST /withdraw ───▶│                            │
 │   { amount }        │                            │
 │                     │── Lock wallet row          │
 │                     │── Deduct balance           │
 │                     │── Create transaction ──────│
 │                     │   status: 'pending'        │
 │                     │── POST /payouts ───────────▶│
 │                     │◀── { payout_ref } ─────────│
 │◀── { status: pending}│                           │
 │                     │                            │
 │                     │◀── Webhook: payout.completed / payout.failed
 │                     │── Update transaction status│
 │                     │── (Refund balance if failed)
```

The balance is deducted immediately on initiation, not on webhook confirmation. This prevents a user from submitting multiple withdrawals before the first one settles. If Revio reports a failed payout via webhook, the amount is refunded within the same DB transaction that updates the status.

---

## 5. Webhook Handling

### 5.1 Verification

Every inbound webhook is verified before any processing:

1. **Signature check:** Revio signs each webhook payload using a shared secret. The API recomputes the expected signature from the raw request body and compares it against the signature header Revio includes on every request, using a timing-safe comparison to prevent timing attacks. Any mismatch returns a `401` immediately.

2. **Timestamp check:** The payload timestamp is validated — webhooks older than 5 minutes are rejected to close the replay window.

3. **Idempotency check:** Before processing, the `webhook_events` table is checked for an existing record matching the same `provider_ref` and `event_type`. If a processed match exists, return `200` without reprocessing.

### 5.2 Processing Flow

```
Revio ── POST /webhooks/revio ──▶ API
                                  │
                      Verify HMAC + timestamp
                                  │
                      Insert into webhook_events (status: 'received')
                      Return 200 OK to Revio immediately
                                  │
                      Process transaction update
                      Update webhook_events (status: 'processed' | 'failed')
```

The `200` is returned to Revio before processing completes. This prevents Revio from treating a slow response as a failure and retrying unnecessarily.

### 5.3 Audit Trail

All webhook events are written to `webhook_events` regardless of outcome, capturing the raw payload, status, attempt count, and any errors. This supports debugging and manual replay if needed.

---

## 6. Security Considerations

### 6.1 Authentication & Authorisation

- JWT access tokens expire after 15 minutes to limit exposure if leaked
- Refresh tokens are stored as hashes; plaintext is never persisted
- `userId` is always extracted from the validated JWT, never from the request body
- Role-based middleware guards admin routes

### 6.2 Replay Attacks & Spoofed Webhooks

- HMAC-SHA256 signature verification on every webhook (see §5.1)
- 5-minute timestamp window rejects replayed payloads
- `idempotency_key` on transactions prevents double-application even if a webhook is delivered more than once

### 6.3 Sensitive Data

- Revio API keys and webhook secrets live in AWS Secrets Manager, injected at runtime — never in `.env` files or source control
- Database credentials managed and rotated via Secrets Manager
- Passwords hashed with `bcrypt` (cost factor 12)
- Bank account details encrypted at rest (AES-256) if stored; where possible, tokenised via Revio

### 6.4 Common Vulnerabilities

| Risk | Mitigation |
|---|---|
| SQL Injection | Parameterised queries via Prisma; raw SQL avoided and always parameterised where used |
| Privilege Escalation | Role from server-signed JWT only, never request payload |
| Mass Assignment | Explicit field allow-lists on all request handlers |
| Rate Limiting | Per-IP and per-user limits enforced at API Gateway on auth and payment endpoints |
| CORS | Restricted to known frontend origins |
| Sensitive Logs | Auth headers and payment fields stripped from logs before shipping to Datadog |

---

## 7. Deployment & Hosting

### 7.1 Infrastructure

- **API:** AWS ECS Fargate — containerised, no server management, horizontal scaling
- **Database:** AWS RDS PostgreSQL, Multi-AZ in production
- **Secrets:** AWS Secrets Manager
- **Load Balancer:** AWS ALB

### 7.2 Environments

Two environments: `staging` and `production`. Staging runs against its own RDS instance and Revio sandbox credentials. Config is injected via Secrets Manager at container start. Staging deploys automatically on merge to `main`; production requires a manual approval step.

### 7.3 CI/CD

```
Feature branch push
       │
       ▼
GitHub Actions: lint → type-check → tests
       │
       ▼
Merge to main → build Docker image → push to ECR
       │
       ▼
Auto-deploy to staging
       │
       ▼
Manual approval → deploy to production
```

### 7.4 Logging & Monitoring

- Structured JSON logs shipped to Datadog via ECS log driver
- Metrics tracked: API latency, error rates, webhook processing time, failed transaction rate
- Alerts on: error rate spikes, failed transactions above threshold, RDS connection saturation
- Sensitive fields stripped from logs at middleware level before reaching Datadog

---

## 8. Potential Failure Points & Mitigations

| Scenario | Risk | Mitigation |
|---|---|---|
| Revio webhook not delivered | Transaction stuck in `pending` | Scheduled job polls Revio for transactions pending longer than 10 min and resolves their status |
| Duplicate webhook delivery | Balance credited twice | `idempotency_key` on transactions + `webhook_events` deduplication check |
| Revio API down during withdrawal | Balance deducted, payout never created | Payout call wrapped in try/catch; failure rolls back the balance deduction within the same DB transaction |
| Concurrent withdrawals | Overdraft | `FOR UPDATE` lock on wallet row + `balance >= 0` DB constraint as final backstop |
| Webhook secret rotation | Signature verification failures | Secrets Manager versioning allows the handler to check both current and previous secret during rotation window |

### 8.1 Reconciliation

A nightly job compares the sum of completed transactions against each wallet balance and flags discrepancies. This is a safety net for edge cases that slip past application-level controls.

---

## 9. Bonus: Dashboard & Scaling

### 9.1 Transaction History

```
GET /wallet/transactions?page=1&limit=20&type=deposit&status=completed
```

A paginated query against the `transactions` table scoped to the authenticated user, with filters for type, status, and date range. The frontend handles rendering; the API returns JSON. If reporting needs grow, a read replica or materialised view would handle analytic queries without impacting the primary database.

### 9.2 Scaling to 100x

The architecture is designed to scale incrementally:

- **Database:** Promote a read replica for balance checks and history queries; partition `transactions` by `created_at` as the ledger grows
- **Application:** ECS Fargate scales horizontally on CPU/memory; scaling policies tightened as load patterns become clear
- **Caching:** Cache wallet balances in Redis with a short TTL for dashboard reads, invalidated on any balance update
- **Architecture:** At sufficient scale, the webhook handler is the clearest candidate to extract as an independent service given its distinct traffic and failure profile

None of these changes require a rewrite — they're additive on top of the existing design.

---

*End of document*
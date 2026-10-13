# Case study 14 · Payment system (checkout payments, like Stripe / Razorpay / Amazon Pay)

**Difficulty:** ⭐⭐⭐⭐ hard. Tests: correctness above everything — idempotency, exactly-once effects, ledgers, reconciliation, third-party integration.

## 1. Requirements

**Functional**
- Accept a payment for an order (card, UPI, wallet, net banking) via external **payment service providers (PSPs)** / gateways.
- Refunds; payment status for the order service; payouts to sellers (optionally).
- A **ledger** of all money movements for accounting.

**Non-functional**
- **Correctness first**: never charge twice, never lose a payment, never lose track of money.
- Highly available, but **consistency beats availability** for money.
- Auditable (every change recorded, nothing silently edited).
- Security and compliance: **PCI DSS** — never store raw card numbers; tokenise.
- Scale is modest compared to feeds: e.g. 10M payments/day ≈ 120/s (peak ~1k/s).

## 2. Key concepts

| Concept | Plain meaning |
|---|---|
| **PSP / gateway** | the external company that actually moves money with banks/card networks (Stripe, Razorpay, Adyen) |
| **Authorise → capture** | first reserve the money on the card, then actually take it (e.g. when the item ships) |
| **Idempotency key** | a unique ID per payment attempt so retries never double-charge |
| **Double-entry ledger** | every movement is recorded as a debit in one account and an equal credit in another; the totals always balance |
| **Reconciliation** | comparing our records with the PSP's and bank's reports to catch mismatches |

## 3. API

```
POST /v1/payments
Idempotency-Key: order-8812-attempt-1
{ "order_id": "8812", "amount": 149900, "currency": "INR", "method_token": "tok_..." }
→ 201 { payment_id, status: "PENDING" | "SUCCEEDED" | "FAILED" | "REQUIRES_ACTION" }

GET  /v1/payments/{id}
POST /v1/payments/{id}/refunds   (also with an Idempotency-Key)
POST /webhooks/psp               (PSP → us: payment status updates)
```

Amounts as **integers in the smallest unit** (paise/cents) — never floating point.

## 4. High-level design

```
Order service ─► Payment service ─► Payments DB (SQL: payment + attempts + idempotency keys)
                     │                     │ (same transaction)
                     │                     └─► outbox ─► Kafka ─► Ledger service ─► Ledger DB (append-only, double-entry)
                     │                                       └─► Order service / notifications / analytics
                     ▼
              PSP adapter(s) ─► Stripe / Razorpay / ... (card data handled by the PSP's hosted form/SDK → token)
                     ▲
        PSP webhooks ─┘ (status updates; verified signatures; idempotent handling)

Reconciliation job (daily): our ledger ↔ PSP settlement files ↔ bank statements → mismatch report / auto-fix
```

## 5. Deep dives

### A. Never charge twice — idempotency end to end
1. The client sends an **idempotency key** per logical payment.
2. The payment service stores `(key → payment_id, request hash, response)` under a **unique
   constraint**. A repeat with the same key returns the stored result; a concurrent duplicate
   hits the constraint and waits / gets 409.
3. Our call to the PSP **also** passes an idempotency key (most PSPs support this), so our
   own retries to the PSP are safe too.

### B. The payment state machine
```
CREATED → PENDING (sent to PSP) → SUCCEEDED
                               ↘ FAILED
                               ↘ REQUIRES_ACTION (3-D Secure / OTP / UPI approval) → SUCCEEDED / FAILED
SUCCEEDED → (PARTIALLY_)REFUNDED
```
Transitions are conditional updates (`WHERE status = 'PENDING'`), so an out-of-order or
duplicate webhook can't move a payment backwards.

### C. Unknown outcomes (the hard case)
Our call to the PSP **times out** — did it charge or not? Don't guess and don't blindly retry
with a new key. Mark the payment `UNKNOWN/PENDING`, then:
- retry with the **same** idempotency key (safe), or
- query the PSP for the status, and/or wait for the webhook,
- and reconciliation catches anything left over.

### D. Ledger (double-entry, append-only)
```
ledger_entries(id, transaction_id, account, direction, amount, currency, created_at)
payment of ₹1,499:
  debit  customer_receivable   1,499
  credit merchant_payable      1,499 - fee
  credit platform_fee_revenue  fee
```
- Entries are **never updated or deleted** — corrections are new reversing entries.
- Sum of debits = sum of credits for every transaction (a built-in check).
- Balances are derived (or cached and verified) from entries.

### E. Consistency across services
Payment status change + ledger + order update can't be one global transaction →
**outbox pattern** (write the payment change and an outbox event in one DB transaction),
idempotent consumers, and **reconciliation** as the final safety net. Saga-style
compensations for refunds/cancellations.

### F. Reliability and security
- Payments DB: SQL with synchronous replication (RPO ≈ 0), multi-AZ.
- Multiple PSPs with routing/failover (and cost/success-rate based routing).
- Card data never touches our servers (PSP hosted fields → token) → smaller PCI scope.
- Verify webhook signatures; encrypt sensitive data; strict access control; full audit log.
- Fraud checks (risk scoring) before authorising.

## 6. Wrap-up

- Idempotency keys (ours and the PSP's), a strict state machine, an append-only double-entry
  ledger, outbox events, and daily reconciliation.
- Money as integers; consistency over availability.
- Monitor: success rate per PSP/method, stuck PENDING payments, reconciliation mismatches, webhook lag.

## Follow-ups

- "Exactly-once payments?" (At-least-once processing + idempotency + reconciliation → no double effects.)
- "The webhook arrives before our own API call returns?" (State machine + conditional updates handle either order.)
- "How do you handle currency conversion / refunds after payout?" (Separate ledger accounts; negative balances recovered from future payouts.)

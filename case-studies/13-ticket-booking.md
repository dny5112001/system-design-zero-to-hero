# Case study 13 · Ticket booking (BookMyShow / Ticketmaster / airline seats)

**Difficulty:** ⭐⭐⭐ medium–hard. Tests: concurrency control, preventing double booking, huge traffic spikes, payments.

## 1. Requirements

**Functional**
- Browse events/shows and see a **seat map** with availability.
- **Select seats → hold them for a few minutes → pay → booking confirmed.**
- Holds expire if payment isn't completed; cancellations and refunds.

**Non-functional**
- **No seat is ever sold twice** (strong consistency) — the core requirement.
- Survive **massive spikes** (a big concert: millions of users at 10:00:00 for 50,000 seats).
- Browsing can be slightly stale; booking must be exact.
- Fair to users; resistant to bots.

## 2. Estimates

```
Normal: 10M bookings/month ≈ 4 bookings/s — tiny
Hot event: 2M users arrive in the first minute for 50k seats → ~30k+ requests/s on ONE event
```

The design challenge isn't average load — it's **contention on a few hot rows** during spikes.

## 3. Data model

```
events(id, venue_id, starts_at, ...)
seats(event_id, seat_id, section, row, price, status, hold_id, hold_expires_at, booking_id, version)
      status: AVAILABLE | HELD | BOOKED
holds(id, user_id, event_id, seat_ids, expires_at)
bookings(id, user_id, event_id, seat_ids, status, payment_id, idempotency_key)
```

Seats live in a **relational DB with transactions** (Postgres), sharded by `event_id` if needed.

## 4. High-level design

```
users ─► CDN (event pages, static seat map) ─► (virtual waiting room for hot events) ─► API gateway (rate limits, bot checks)
         ─► Event service (browse; cached availability)
         ─► Booking service ─► Seats DB (holds and bookings: transactions / conditional updates)
                          ─► Payment service (PSP) ─► webhook ─► confirm booking
         Expiry worker: releases expired holds
```

## 5. Deep dive: preventing double booking

### Option A — Pessimistic lock
```sql
BEGIN;
SELECT * FROM seats WHERE event_id=? AND seat_id IN (...) FOR UPDATE;   -- lock rows
-- check all AVAILABLE, then:
UPDATE seats SET status='HELD', hold_id=?, hold_expires_at=now()+interval '8 min' WHERE ...;
COMMIT;
```
✅ Simple and correct. ❌ Locks queue up under heavy contention. Keep transactions tiny; **never** hold a lock while calling the payment provider.

### Option B — Optimistic / conditional update ⭐
```sql
UPDATE seats SET status='HELD', hold_id=?, hold_expires_at=?
WHERE event_id=? AND seat_id=? AND status='AVAILABLE';
-- rows affected = 1 → you got it; 0 → someone else did
```
(For several seats: do them in one transaction and roll back if any returns 0.)
✅ No long locks, scales well, clearly correct.

### Option C — Redis holds
`SET hold:{event}:{seat} {user} NX EX 480` for a fast hold, then write the booking
to the DB. Fast at huge scale, but now two stores must agree → the DB must still
be the final authority (unique constraint on `(event_id, seat_id)` in bookings).

**Safety net in every option:** a **unique constraint** so two confirmed bookings for
the same seat are impossible at the database level.

### Hold expiry
Holds expire after N minutes: either a worker resets expired `HELD` seats, or reads
treat `HELD AND hold_expires_at < now()` as available (lazy expiry) — usually both.

## 6. Payments flow

```
hold seats ─► create booking PENDING (idempotency key) ─► redirect to payment (PSP)
   ─► PSP webhook "paid" ─► in ONE transaction: verify the hold is still ours → seats BOOKED, booking CONFIRMED
   ─► if the hold expired before payment completed → refund automatically (or re-check availability)
```

Webhooks are retried by the PSP → the confirm handler must be **idempotent**.
Run a reconciliation job against PSP reports (case study 14).

## 7. Surviving the spike (hot events)

- **Virtual waiting room / queue**: users enter a queue at the edge and are let in at a
  rate the booking system can handle (random or FIFO order — fair, and it protects the
  backend). Show their queue position.
- **Static, cached** event pages and seat-map layout on the CDN; availability updates
  pushed or polled with short TTLs — never query the DB for every browser refresh.
- **Rate limits + bot protection** (CAPTCHA, device fingerprinting, per-account ticket limits).
- Optional: **inventory partitioning** — split seat sections across shards/workers so
  contention spreads out.
- Autoscale stateless services ahead of time; pre-warm caches.

## 8. Wrap-up

- Conditional updates (or `FOR UPDATE`) + unique constraint = no double booking.
- Holds with expiry; payment confirmation is idempotent; reconcile with the PSP.
- Waiting room + CDN + rate limits for spikes.
- Monitor: hold → booking conversion, expired holds, payment failures, queue wait times.

## Follow-ups

- "General admission (no seat numbers)?" (A counter: `UPDATE inventory SET remaining = remaining - ? WHERE event_id = ? AND remaining >= ?`.)
- "User's payment succeeded but the confirmation failed?" (The webhook retries + reconciliation fix it; the hold or a refund covers the gap.)
- "How do you show live seat availability to 1M viewers?" (Cached snapshots refreshed every few seconds + push updates; accept slight staleness, recheck at hold time.)

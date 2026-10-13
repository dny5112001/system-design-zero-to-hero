# Case study 12 · Ride-hailing (Uber / Ola / Lyft)

**Difficulty:** ⭐⭐⭐ hard. Tests: geospatial indexing, high-frequency location writes, matching, state machines, consistency where it matters.

## 1. Requirements

**Functional**
- Riders request a ride (pickup, drop); see nearby drivers and an ETA and fare estimate.
- The system **matches** a nearby available driver; the driver accepts or declines.
- Live tracking of the driver during the trip; trip ends → payment → ratings.

**Non-functional**
- Matching within seconds; location updates feel live.
- **A driver must never be assigned two trips at once**, and a trip never gets two drivers (strong consistency here).
- Highly available — if matching is down, the business is down.
- Location data can be slightly stale (a few seconds is fine).

## 2. Estimates

```
1M drivers online at peak, each sends GPS every 4 s  → 250,000 location writes/s
Riders: 20M rides/day ≈ 230 ride requests/s (peak ~1–2k/s)
Location points: 250k/s × 100 B ≈ 25 MB/s; keep only the latest per driver hot,
                 stream history to cheap storage (trip replay, analytics, fraud)
```

The **location write rate** dominates → an in-memory geospatial index, not a disk database.

## 3. High-level design

```
Driver app ══(WebSocket/UDP-ish, every 4s)══► Location service ─► in-memory geo index (sharded by city/region)
                                                       └─► Kafka (location stream) ─► history store, ETA/ML, surge

Rider app ─► API gateway ─► Ride service ─► Matching service ─► geo index: "available drivers near pickup"
                                │                    │
                                │                    └─► offer to best driver (push) ─► accept / timeout → next
                                ▼
                           Trip DB (SQL; trip state machine)   Pricing (fare, surge)   Payment service
Live tracking: driver locations ─► pushed to the rider over WebSocket
```

## 4. Deep dives

### A. Finding nearby drivers (geospatial index)
- Divide the map into cells: **geohash**, **quadtree**, or **H3 hexagons** (Uber's choice — every neighbour is the same distance away).
- Index: `cell_id → set of available driver IDs` (plus each driver's latest location), **in memory** (Redis GEO or a custom service).
- Query: the pickup's cell + its neighbouring rings → candidate drivers → filter (available, car type) → rank by **ETA** (road distance, not straight-line).
- Each update: if the driver moved to another cell, remove them from the old cell and add to the new one.
- **Shard by city/region** — cities are natural partitions, and nearly all queries are local. Big cities split into several shards.

### B. Handling 250k writes/s
In-memory, sharded, only the **latest** location kept hot. Updates are fire-and-forget
(losing one GPS ping doesn't matter — the next arrives in 4 s). History goes to Kafka
→ cheap storage.

### C. Matching without double-booking
- Matching offers the trip to one driver at a time (or a small batch), with a timeout (~10–15 s).
- Driver state changes atomically: `AVAILABLE → OFFERED → ON_TRIP`, using a conditional
  update (`... SET status='ON_TRIP', trip_id=? WHERE driver_id=? AND status='OFFERED' AND offer_id=?`)
  or a lock with expiry in Redis. Exactly one trip wins.
- Trip state machine in a **strongly consistent** store:
  `REQUESTED → MATCHED → DRIVER_ARRIVING → IN_PROGRESS → COMPLETED → PAID` (and `CANCELLED`).
- Better global matches: batch requests for a second or two and solve an assignment problem across many riders/drivers.

### D. ETA and pricing
ETA from a routing engine over the road graph plus live traffic (ML corrections).
**Surge pricing**: per cell, compare demand (requests) vs supply (available drivers)
over the last few minutes (streaming aggregation) → multiplier.

### E. Live tracking
The driver's updates are forwarded to the rider's open WebSocket (via pub/sub on
`trip_id`). Smooth movement on the client by interpolating between points.

### F. Payments
Charge at trip end via the payment service (case study 14): idempotency key = trip ID,
pre-authorise at request time. Driver payouts batched later.

### G. Reliability
- Location / matching services are stateless over the sharded in-memory index; the
  index can be rebuilt from the next round of pings (~4 s) if a shard is lost.
- Multi-AZ; region failover per city.
- Phone loses network mid-trip: the app buffers GPS points and uploads them later (needed for fare calculation).

## 5. Wrap-up

- In-memory geo index (H3/geohash) sharded by city; 250k writes/s of GPS.
- Matching with timeouts + atomic driver state transitions → no double booking.
- Strong consistency for trips and payments; eventual for locations.
- Monitor: match time, acceptance rate, location lag, ETA accuracy.

## Follow-ups

- "Food delivery instead?" (Same core plus restaurant prep time and batching several orders per driver.)
- "Show riders the cars on the map?" (Read the geo index for nearby cells, throttled, slightly stale.)
- "Driver app goes offline?" (Heartbeat timeout → mark unavailable, remove from index.)

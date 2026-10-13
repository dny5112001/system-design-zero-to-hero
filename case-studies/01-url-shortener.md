# Case study 01 · URL shortener (TinyURL / bit.ly)

**Difficulty:** ⭐ easy — the classic warm-up. Tests: estimation, ID generation, caching, read-heavy design.

## 1. Requirements

**Functional**
- Given a long URL, return a short one: `https://sho.rt/aZ3kQ9x`.
- Visiting the short URL **redirects** to the long one.
- Optional: custom alias, expiry date, click analytics.

**Non-functional**
- Very **read-heavy** (people click far more than they create).
- Redirect must be **fast** (< 50 ms server time) and **highly available**.
- Short codes must be **unique** and **not guessable in order** (privacy).
- Durability: a created link must never break.

## 2. Estimates

```
New URLs: 100M / day          → ~1,200 writes/s (peak ~3,000)
Reads 10:1                     → ~12,000 redirects/s (peak ~30,000+)
5 years: 100M × 365 × 5        ≈ 180 B URLs
Record ≈ 500 bytes             → 180B × 500 B ≈ 90 TB (× replication)
Reads/day ≈ 12,000 × 10⁵ ≈ 1.2 B; cache the hot 20% → 0.24 B × 500 B ≈ 120 GB → a small Redis cluster
```

How long must the code be? Base62 (`a-z A-Z 0-9`):
62⁶ ≈ 57 billion (too few for 180 B), **62⁷ ≈ 3.5 trillion** ✔ → **7 characters**.

## 3. API

```
POST /v1/urls   { long_url, custom_alias?, expires_at? }  → 201 { short_url }
GET  /{code}    → 301/302 redirect to long_url  (404 if unknown, 410 if expired)
```

**301 or 302?** 301 (permanent) lets browsers cache the redirect → less load,
but you **lose click analytics** (later clicks don't reach you). 302 (temporary)
→ every click hits you → accurate analytics. Say which you choose and why.

## 4. Data model

```
urls(code PK, long_url, user_id, created_at, expires_at)
```

Access is pure key lookup by `code` → a **key-value / wide-column store**
(DynamoDB, Cassandra) scales easily; Postgres sharded by `code` also works.
Analytics go elsewhere (events → Kafka → warehouse).

## 5. High-level design

```
           ┌─────────── write path ───────────┐
client ─► LB ─► URL service ─► ID/code generator
                    │
                    ├──► DB (code → long_url)
                    │
read path:  client ─► CDN/edge? ─► LB ─► redirect service ─► Redis cache ─(miss)─► DB
                                                   └──► click event ─► Kafka ─► analytics
```

## 6. Deep dives

### A. How to generate the short code — the main question

| Option | How | ✅ | ❌ |
|---|---|---|---|
| **Hash the URL** (MD5/SHA → take 7 chars) | same URL → same code | dedupes identical URLs | collisions → check DB & retry with salt; extra read on write |
| **Random 7 chars** | generate, check if taken | simple, unguessable | collision checks grow as space fills (still tiny at 180B/3.5T ≈ 5%) |
| **Counter + base62** | unique number → base62 | no collisions, no checks | a single counter is a bottleneck/SPOF; codes are sequential (guessable) |
| **Pre-generated key service (KGS)** | generate unused codes offline into a table; servers grab **batches** into memory | fast, no collisions at write time | extra service; keys in a crashed server's batch are wasted (fine) |

**Good answer:** a distributed counter done right — each app server gets a
**range** of IDs (e.g. 1,000,000 at a time) from a coordinator (ZooKeeper/etcd or
a DB row), hands them out from memory, then base62-encodes. To avoid
guessable sequences, scramble the number first (e.g. a reversible bijective
shuffle / Feistel cipher) before base62. Alternatively Snowflake IDs (case study 02).

### B. Read path at 30k+ QPS

- **Redis cache** (cache-aside, LRU): hot links stay in memory; hit ratio > 90%.
- URLs are **immutable** once created → caching is easy (no invalidation except on delete/expiry).
- Optional: cache redirects at the **CDN/edge** for very hot links.
- DB: read replicas or a KV store partitioned by `code`.

### C. Custom aliases
Check-and-insert atomically: `INSERT ... IF NOT EXISTS` (conditional put) to
avoid two users claiming the same alias.

### D. Expiry and cleanup
Store `expires_at`; check on read; a background job (or DB TTL feature) deletes
expired rows lazily.

### E. Abuse
Rate-limit creation per user/IP; scan long URLs against malware/phishing lists;
allow takedowns.

## 7. Wrap-up

- **Bottlenecks**: ID generation (solved with ranges), DB reads (cache).
- **Failure**: replicated KV store across AZs; if the cache dies, DB absorbs load (size it for that or use replicas).
- **Analytics**: async click events → Kafka → aggregate per code per hour.
- **Extensions**: per-user dashboards, QR codes, geo/device stats.

## Follow-up questions to expect

- "Two users shorten the same long URL — same code?" (Your choice: dedupe via hash index, or not — explain.)
- "How do you prevent enumeration of all links?" (Non-sequential codes, rate limiting.)
- "What if the counter service goes down?" (Servers hold pre-fetched ranges; coordinator replicated.)

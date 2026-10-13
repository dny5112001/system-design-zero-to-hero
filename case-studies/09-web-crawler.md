# Case study 09 · Web crawler

**Difficulty:** ⭐⭐ medium. Tests: queues at scale, politeness, deduplication, distributed work.

## 1. Requirements

**Functional**
- Start from seed URLs, download pages, extract links, repeat.
- Store page content for a search indexer (or other consumers).
- Revisit pages periodically to catch changes.

**Non-functional**
- Scale: **1 billion pages/month**.
- **Polite**: don't overload any website; obey `robots.txt`.
- Robust against bad pages, crawler traps, huge files, slow servers.
- Extensible (new content types later).

## 2. Estimates

```
1B pages / month ≈ 1B / (30 × 10⁵ s) ≈ 400 pages/s (peak ~800)
Avg page 500 KB → 500 TB/month raw (compressed ~100 TB); keep 5 years → multiple PB → object storage
```

## 3. High-level design

```
seed URLs ─► URL Frontier (prioritised, per-host queues) ─► Fetcher workers ─► DNS resolver (cached)
                 ▲                                                    │
                 │                                             HTML downloaded
                 │                                                    ▼
          URL filter ◄── "seen URL?" (Bloom filter / URL DB) ◄── Link extractor ◄── Content parser
                                                                      │                │
                                                     "seen content?" (hash/SimHash)    │
                                                                      ▼                ▼
                                                              Content store (S3)   Indexer / consumers (via Kafka)
```

## 4. Deep dives

### URL Frontier — the heart of the crawler
Two jobs:
1. **Priority**: important pages first (PageRank, site popularity, change rate). Several priority queues; a selector picks higher priority more often.
2. **Politeness**: never hit one host too fast. Keep **one queue per host**; each
   host queue is handled by one worker at a time, with a delay between requests
   (respect `Crawl-delay`). A heap/timer tracks "host X may be fetched again at time T".

The frontier is too big for memory (billions of URLs) → mostly on disk / Kafka /
a DB, with in-memory buffers at the front.

### Distribution
Partition by **host** (hash(hostname) → crawler node) so politeness for a host is
enforced in one place, and DNS/robots caches stay local.

### robots.txt
Fetch and **cache** per host (refresh daily). Check every URL before fetching.

### Deduplication
- **URLs**: normalise (lower-case host, remove fragments, sort query params, drop
  tracking params), then check a **Bloom filter** (fast, memory-light; a rare false
  positive just skips a URL) backed by a URL store.
- **Content**: many URLs serve the same page. Hash content for exact duplicates;
  **SimHash/MinHash** for near-duplicates.

### DNS
Lookups are slow and can become the bottleneck → a local caching resolver per node.

### Robustness
- Timeouts, max page size, allowed content types only.
- **Crawler traps** (infinite calendars, session IDs in URLs): max URL length, max
  depth per site, max pages per host per crawl, pattern blacklists.
- Persist fetch state so a crashed worker's URLs are retried (at-least-once; dedupe makes it safe).

### Freshness / recrawl
Track each page's change frequency and recrawl fast-changing pages more often.
Use `If-Modified-Since` / ETags to skip unchanged pages.

### JavaScript-heavy sites
A separate, slower pool of headless browsers for pages that need rendering.

## 5. Wrap-up

- Frontier with priority + per-host politeness, partitioned by host.
- Bloom-filter URL dedupe + content hashing; robots.txt cache; DNS cache.
- Monitor: pages/s, error rate per host, frontier size, duplicate ratio.

## Follow-ups

- "How do you avoid hammering a small site?" (Per-host queue + delay + robots.txt.)
- "How do you prioritise?" (Importance, freshness, domain authority.)
- "Bloom filter false positives?" (A few new URLs get skipped — acceptable; size the filter for a low rate.)

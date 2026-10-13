# Case study 08 · Search autocomplete (typeahead)

**Difficulty:** ⭐⭐ medium. Tests: tries, precomputation, caching, very low latency.

## 1. Requirements

**Functional**
- As the user types, show the **top 5–10 suggestions** for the prefix, ranked by popularity.
- Optional: personalisation, trending, typo tolerance, multiple languages.

**Non-functional**
- **Very fast**: < 100 ms end to end (users type quickly).
- Huge read volume; suggestions can be **minutes/hours stale** (eventual consistency is fine).
- Filter offensive suggestions.

## 2. Estimates

```
10M DAU × 10 searches × ~6 keystrokes = 600M requests/day ≈ 7,000/s (peak ~20k)
```

Read-only on the hot path; data updates come from offline processing.

## 3. Core data structure: trie with cached top-K

A **trie** (prefix tree): each node is a character; paths spell words.

```
         (root)
        /      \
       c        t
       |        |
       a        r
      / \       |
     r   t      e ...
 [car(900),  [cat(500), catalog(120)]
  cart(300)]
```

Naive: for prefix "ca", walk to the node, then search all descendants for the
most popular → too slow.

**Optimisation:** store the **top-K completions at every node**. Lookup = walk
the prefix (length L) → return the stored list. O(L), tiny.
Cost: more memory and slower updates — fine, because updates are offline.

Also limit prefix length (e.g. suggestions only computed for the first ~20 chars).

## 4. High-level design

```
            ┌──────────── offline (data pipeline) ────────────┐
search logs ─► Kafka ─► aggregate counts (Spark/Flink, hourly/daily, time-decayed)
                         ─► build trie + top-K per node ─► snapshot (S3) ─► load into suggestion servers

online:  client (debounce ~100–200ms) ─► CDN/edge cache ─► LB ─► suggestion service (trie in memory)
```

## 5. Deep dives

### Serving
- Trie lives **in memory** on suggestion servers (or Redis: key = prefix → sorted list).
- Shard by **prefix** if too big for one machine (careful: "s" is much bigger than "x" → split hot ranges unevenly, using a shard map based on real data).
- Replicate for read throughput.

### Updating
Rebuild the trie periodically from aggregated logs; blue-green swap of the new
snapshot. Weight recent searches higher (time decay) so trends rise.
For **trending in near-real-time**: a streaming job (Count-Min Sketch / top-K per
prefix over the last hour) merged into results.

### Caching on the client and edge
- Browser caches responses for a prefix (`Cache-Control: max-age=...`).
- CDN caches popular prefixes ("a", "ip", "wea") — huge hit rates.
- **Debounce** keystrokes on the client and cancel stale requests.

### Ranking & personalisation
Global popularity + user's recent searches + location/language. Personal results
are merged on the server (or client) from a small per-user store.

### Filtering
Blocklist offensive/illegal terms when building the trie, plus an online filter for emergencies.

### Typos
Fuzzy matching (edit distance) is expensive; use a spelling-correction step on
the full query or precomputed common misspellings.

## 6. Wrap-up

- Precompute top-K per prefix offline; serve from memory; cache at CDN and browser.
- Monitor: p99 latency, cache hit ratio, freshness of the snapshot.

## Follow-ups

- "Why not query Elasticsearch on each keystroke?" (Possible with prefix/edge-n-gram indexes, but the precomputed trie is faster and cheaper at huge QPS.)
- "Support multiple languages?" (Separate tries per locale; Unicode normalisation.)

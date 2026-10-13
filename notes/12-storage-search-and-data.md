# 12 · Storage types, search, and data pipelines

## Three kinds of storage

| Type | What | Like | Examples | Use for |
|---|---|---|---|---|
| **Block** | raw disk blocks attached to one machine | a hard drive | AWS EBS, local SSD | databases, VM disks |
| **File** | folders and files, shared over the network | a shared drive | NFS, AWS EFS | shared files between servers, legacy apps |
| **Object** | flat buckets of objects (file + metadata) addressed by key, via HTTP | a valet coat-check: give a ticket, get your coat | **S3**, GCS, Azure Blob, MinIO | images, videos, backups, logs, data lakes |

**Object storage is the default for user files.** It's cheap, practically
unlimited, and extremely durable (S3 is designed for 11 nines of durability by
storing copies across several data centres). Objects are written whole (no
editing in the middle).

### Upload pattern: pre-signed URLs

Don't stream large uploads through your app servers:

```
1. client → API: "I want to upload photo.jpg"
2. API → client: a pre-signed S3 URL (valid 10 minutes, for this exact key)
3. client → S3: uploads directly
4. S3 → event (or client → API "done") → processing pipeline (thumbnails, scanning)
```

Big files: **multipart / resumable** uploads (chunks in parallel, retry just the failed chunk).

Store **metadata** (owner, size, key, status) in a database; the **bytes** in
object storage; serve them through a **CDN**.

Storage tiers: hot (standard), infrequent access, archive (Glacier) — move old
data down automatically with lifecycle rules to save cost.

## Search — the inverted index

`LIKE '%phone%'` scans every row — too slow and dumb (no ranking, no typos).

A search engine builds an **inverted index**: for each word, the list of
documents containing it — like the index at the back of a book.

```
"red"    → [doc 1, doc 7, doc 42]
"phone"  → [doc 7, doc 9]
query "red phone" → intersect → doc 7
```

Indexing steps: **tokenize** ("Red Phones!" → "red", "phones"), **normalise**
(lower-case, remove punctuation), **stem** ("phones" → "phone"), drop stop words.

Ranking: **TF-IDF / BM25** (words that are frequent in this doc but rare overall
score high), plus business signals (popularity, recency, personalisation).
Modern search adds **vector/semantic search** (embeddings) — "hybrid search".

**Elasticsearch / OpenSearch** (also Solr, Typesense, Meilisearch): shards +
replicas, near-real-time indexing, fuzzy matching, facets, aggregations.

**Pattern:** the database stays the **source of truth**; search is a **secondary
index** kept in sync via CDC or events (eventually consistent — a new product
appears in search within seconds).

## Data pipelines and analytics

```
apps/DBs ──(events, CDC)──► Kafka ──► stream processing (Flink) ──► real-time dashboards, alerts
                                └──► data lake (S3, Parquet) ──► warehouse (Snowflake/BigQuery) ──► BI, ML
```

- **Batch** (hourly/daily jobs, Spark): big, cheap, delayed.
- **Stream** (continuous, Flink/Kafka Streams): fresh, more complex.
- **ETL vs ELT**: transform before loading, or load raw then transform in the warehouse (modern default).
- **Lambda architecture** (batch + stream layers) vs **Kappa** (stream only, replay for reprocessing).

### Counting at huge scale

- Exact counts of everything are expensive. Often **approximate** is fine:
  **HyperLogLog** (unique visitors in ~12 KB), **Count-Min Sketch** (top-K / heavy hitters) — chapter 15.
- Pre-aggregate into time buckets (per minute/hour) instead of counting raw events at query time.

## Remember

- User files → object storage + metadata in a DB + CDN; upload via pre-signed URLs.
- Search = inverted index in a dedicated engine, synced from the source of truth.
- Analytics belong in a warehouse fed by events/CDC, not on the production DB.

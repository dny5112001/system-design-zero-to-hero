# 07 · Databases: SQL vs NoSQL, indexes, and choosing well

"Which database?" comes up in **every** interview. The right answer is a choice
**plus the reason**, based on the data shape and access pattern.

## Relational (SQL) databases

PostgreSQL, MySQL, SQL Server, Oracle; cloud: Aurora, Cloud SQL, Spanner, CockroachDB.

- Data in **tables** with a fixed **schema**; **relations** via foreign keys; powerful **SQL** with joins.
- **ACID transactions**:

| Letter | Means | Plain words |
|---|---|---|
| **A**tomicity | all or nothing | transfer: debit and credit both happen, or neither |
| **C**onsistency | rules always hold | constraints (unique email, balance ≥ 0) are never broken |
| **I**solation | concurrent transactions don't trip over each other | (levels below) |
| **D**urability | committed = saved | survives a crash (write-ahead log) |

**Choose SQL when**: data is relational, you need transactions (money, orders,
inventory, bookings), flexible queries/reporting, strong consistency.
**This is the safe default** — say "Postgres" unless there's a reason not to.

Scaling SQL: vertical first, read replicas, caching, then sharding (harder —
cross-shard joins/transactions are painful) or a **distributed SQL** database
(Spanner, CockroachDB, YugabyteDB, Aurora DSQL) that shards for you.

### Isolation levels (the interview-grade summary)

| Level | Prevents | Still allows |
|---|---|---|
| Read uncommitted | — | dirty reads (seeing uncommitted data) |
| Read committed (Postgres default) | dirty reads | non-repeatable reads, phantoms, lost updates |
| Repeatable read / snapshot | + non-repeatable reads | write skew |
| Serializable | everything — as if one at a time | (slower, may need retries) |

For "two people book the last seat": use `SELECT ... FOR UPDATE` (row lock),
a conditional update (`UPDATE seats SET held_by=? WHERE id=? AND held_by IS NULL`),
optimistic locking with a version column, or serializable isolation.

## NoSQL databases — four families

| Type | Shape | Examples | Great for | Weak at |
|---|---|---|---|---|
| **Key-value** | `key → value` | Redis, DynamoDB, Riak | caching, sessions, simple lookups by ID at huge scale | queries by anything but the key |
| **Document** | JSON-like documents | MongoDB, Couchbase, Firestore | flexible/nested data, catalogs, profiles | joins, multi-document transactions (limited) |
| **Wide-column** | rows with many dynamic columns, partitioned by key | **Cassandra**, ScyllaDB, HBase, Bigtable | **massive write volume**, time-series, messages, feeds | ad-hoc queries, joins; you design tables per query |
| **Graph** | nodes + edges | Neo4j, Neptune | relationships many hops deep: social graphs, fraud, recommendations | bulk analytics, simple CRUD |

Plus specialised stores:
- **Time-series**: InfluxDB, TimescaleDB, Prometheus (metrics).
- **Search**: Elasticsearch/OpenSearch (chapter 12).
- **Analytics / OLAP (columnar)**: ClickHouse, BigQuery, Snowflake, Redshift, Druid.
- **Vector databases**: pgvector, Pinecone, Milvus (AI similarity search).

### Choosing — a decision guide

```
Need transactions / relations / complex queries?          → SQL (Postgres)
Huge write throughput, access by known key, OK eventually consistent?  → Cassandra/DynamoDB
Flexible, nested, per-entity documents?                    → MongoDB / DynamoDB
Simple lookups, ultra-fast, data fits in RAM?             → Redis
Deep relationship queries?                                → graph DB
Full-text search?                                         → Elasticsearch (as a secondary index)
Analytics over billions of rows?                           → columnar warehouse
Files, images, video?                                     → object storage (S3), never the DB
```

It's normal (and expected) to use **several** — "polyglot persistence" — e.g.
Postgres for orders, Redis for sessions, S3 for images, Elasticsearch for search.

### NoSQL data modelling = start from the queries

In Cassandra/DynamoDB you design tables **around the queries**, and duplicate
data across tables if needed (denormalisation). Choose the **partition key**
carefully: it decides which node holds the data and must spread load evenly.

```
Query: "latest messages in chat X"
Table messages_by_chat: PRIMARY KEY ((chat_id), message_id DESC)
   partition key = chat_id, clustering key = message_id (sorted, time-ordered)
```

## Indexes — the most important performance tool

An **index** is like the index at the back of a book: jump straight to the page
instead of reading everything.

- Without an index: **full table scan** — fine for 1,000 rows, terrible for 1 billion.
- **B-tree** (default in SQL): sorted, great for `=`, ranges, `ORDER BY`.
- **Hash**: `=` only. **LSM-tree** (Cassandra, RocksDB): very fast writes (sequential), compacted in the background.
- **Composite index** `(user_id, created_at)`: order matters — works for `user_id = ?` and `user_id = ? ORDER BY created_at`, not for `created_at` alone.
- Others: full-text, geospatial (R-tree, geohash), inverted (search).
- **Cost**: every index slows writes and uses space. Index what you query.

## Normalisation vs denormalisation

- **Normalised**: each fact stored once; join to combine. Clean, no update anomalies, joins cost.
- **Denormalised**: copy data where it's read (e.g. store author name on each post). Fast reads, but updates must touch every copy.
- Read-heavy systems at scale often denormalise and precompute (feeds, counters).

## OLTP vs OLAP

| | OLTP | OLAP |
|---|---|---|
| Workload | many small reads/writes (the app) | big scans/aggregations (reports) |
| Storage | row-oriented | column-oriented |
| Examples | Postgres, MySQL, DynamoDB | BigQuery, Snowflake, ClickHouse |

Don't run heavy analytics on the production database; stream data (CDC / ETL) into a warehouse.

## Remember

- Default to SQL; choose NoSQL for a stated reason (scale of writes, access pattern, flexibility).
- In NoSQL, design from queries and pick the partition key carefully.
- Indexes make reads fast and writes slower. Use several stores for several jobs.

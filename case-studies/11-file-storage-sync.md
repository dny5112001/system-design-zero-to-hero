# Case study 11 · File storage and sync (Dropbox / Google Drive)

**Difficulty:** ⭐⭐⭐ hard. Tests: chunking, deduplication, metadata vs blob storage, sync and conflicts.

## 1. Requirements

**Functional**
- Upload, download, delete files and folders.
- **Automatic sync** across a user's devices.
- Share files/folders with others; file version history.
- Works offline; syncs when back online.

**Non-functional**
- 50M users, 10M DAU; files up to ~10 GB.
- **Never lose data** (extreme durability).
- Sync should be fast and bandwidth-efficient (mobile, slow networks).
- Strong consistency for **metadata** (the folder tree must not be corrupted).

## 2. Estimates

```
50M users × 10 GB average used ≈ 500 PB stored (before dedupe/replication) → object storage
10M DAU × 2 file changes/day = 20M changes/day ≈ 230/s (peak ~1k/s)
Most changes are small edits to existing files → sending only changed chunks saves huge bandwidth
```

## 3. Key idea: split files into chunks

Split each file into **chunks** (~4 MB), and identify each chunk by its
**content hash** (SHA-256).

- **Upload only changed chunks** — edit one paragraph of a 1 GB file → upload ~4 MB.
- **Deduplication** — the same chunk (same hash) is stored **once**, even across users.
- **Resumable** — a failed upload resumes from the last chunk.
- **Parallel** — upload/download several chunks at once.

A **file** = an ordered list of chunk hashes (its "block list") + metadata.

> Content-defined chunking (rolling hash, as in rsync) makes chunk boundaries follow
> the content, so inserting a byte near the start doesn't shift every later chunk.

## 4. High-level design

```
Desktop/mobile client
  ├─ watcher: notices local file changes
  ├─ chunker: split + hash
  ├─ local index DB: what it last synced
  │
  ├──► Metadata service ─► Metadata DB (SQL, strongly consistent)
  │       files, versions, block lists, folder tree, sharing, per-user journal
  │
  ├──► Block service ─► object storage (S3) keyed by chunk hash, + CDN for downloads
  │
  └◄── Notification service (long poll / WebSocket): "your namespace changed, cursor = 1042"
```

## 5. Upload flow

```
1. Client chunks the file, computes hashes [h1, h2, h3].
2. Client → metadata service: "file X new version = [h1, h2, h3]".
3. Server answers: "I already have h1 and h3; send h2" (dedupe check).
4. Client uploads h2 to the block service (pre-signed URL).
5. Client → commit: metadata DB writes the new version + block list in ONE transaction,
   appends an entry to the user's change journal (cursor++).
6. Notification service tells the user's other devices: "changes after cursor 1041".
```

## 6. Download / sync flow

```
device B (listening via long poll) ─► "changes since cursor 1041?"
  ─► metadata: [file X → version 7 = [h1, h2', h3]]
  ─► B compares with its local index, downloads only h2' (from CDN/block service)
  ─► rebuilds the file locally, updates its cursor
```

## 7. Deep dives

### Metadata DB
Relational, **strongly consistent** (folder moves, renames, sharing must be atomic).
Shard by user / namespace (a user's or a shared folder's tree). This is the critical
system to protect; blobs are simpler (immutable, content-addressed).

### Conflicts
Two devices edit the same file offline. On commit, the server checks the base
version: if the file changed since the client last synced → **conflict**. Typical
answer (Dropbox): keep both — save one as "file (Asha's conflicted copy).docx".
Real-time collaborative editors (Google Docs) instead use OT/CRDTs on the document
operations.

### Versioning
Each commit creates a new version pointing at its block list. Old versions share
unchanged chunks, so history is cheap. Retention policy: N days/versions.
Garbage collection: chunks no longer referenced by any version are deleted later
(reference counting / mark-and-sweep).

### Durability
Object storage with replication / erasure coding across zones (S3: 11 nines).
Never acknowledge a commit until all chunks are durably stored.

### Sharing and permissions
ACLs on folders/files in the metadata DB; check on every metadata and block access
(signed, short-lived URLs for blocks).

### Notifications
Clients keep a long poll / WebSocket open to the notification service; changes are
published per namespace. On reconnect, clients sync from their **cursor** — nothing
is missed even if a notification was.

### Security
TLS in transit, encryption at rest. Cross-user dedupe can leak "someone already has
this file" → some services dedupe only within a user.

## 8. Wrap-up

- Chunking + content hashing = delta sync, dedupe, resumable parallel transfers.
- Strongly consistent metadata DB + immutable content-addressed blobs.
- Journal/cursor-based sync with push notifications; conflicted copies.
- Monitor: sync latency, upload failure rate, metadata DB latency, storage growth.

## Follow-ups

- "Why not store files in the database?" (Huge blobs kill DB performance and cost — metadata in the DB, bytes in object storage.)
- "How does a 10 GB upload survive a flaky network?" (Chunked, resumable, parallel.)
- "How do you delete safely with dedupe?" (Reference counts / GC after the retention window.)

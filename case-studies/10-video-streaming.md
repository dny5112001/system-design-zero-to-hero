# Case study 10 · Video streaming platform (YouTube / Netflix)

**Difficulty:** ⭐⭐⭐ hard. Tests: large-file upload, processing pipelines, CDNs, adaptive streaming, cost.

## 1. Requirements

**Functional**
- Upload videos; watch them smoothly on any device and network.
- Video metadata: title, description, thumbnails; search; view counts; likes/comments.

**Non-functional**
- Huge scale: e.g. 1B daily viewers, 500 hours uploaded per minute (YouTube-ish).
- Smooth playback, fast start (< 2 s), minimal buffering.
- Uploads must never be lost; processing may take minutes.
- **Cost** matters enormously (storage + bandwidth).

## 2. Estimates

```
Uploads: 500 hours/min × 1,440 min ≈ 720k hours/day
Storage: source + ~5 renditions ≈ 10 GB per hour of video → ~7 PB/day → tiered object storage
Watching: 1B viewers × 30 min ≈ 500M hours/day; at ~3 Mbps average
          1 hour ≈ 1.35 GB → ~675 PB/day of egress (!)
          → served almost entirely by CDNs / ISP-embedded caches (Netflix Open Connect, Google GGC)
```

The numbers say: **CDN + storage tiering + efficient encoding** decide the design.

## 3. High-level design

```
UPLOAD
client ─(resumable, chunked, pre-signed URLs)─► object storage: raw/
              │ "upload complete" event
              ▼
     Processing pipeline (a DAG of jobs on a worker fleet, via queues)
       ├─ validate / virus & policy scan
       ├─ split into segments ─► transcode in parallel ─► several resolutions & codecs
       │                         (240p … 4K; H.264 / VP9 / AV1)
       ├─ package for streaming (HLS / DASH: small segments + manifests)
       ├─ thumbnails, captions (speech-to-text), content moderation
       └─ write outputs ─► object storage: processed/ ─► mark video READY in metadata DB

WATCH
client ─► API: GET /videos/{id} ─► metadata (cache + DB) + signed manifest URL
client ─► CDN edge ─(miss)─► regional cache / origin shield ─(miss)─► object storage
          fetches segments, picking a quality per segment based on bandwidth (ABR)
```

## 4. Deep dives

### Upload
Large files over flaky mobile networks → **resumable, multipart** uploads directly
to object storage (pre-signed URL per chunk). A metadata row is created first, with
status `UPLOADING → PROCESSING → READY / FAILED`.

### Transcoding pipeline
- Split the video into chunks (by GOP/scene), **transcode chunks in parallel** across
  many workers, then stitch → minutes instead of hours.
- Model it as a **DAG** of tasks run by an orchestrator (Temporal / Airflow / custom),
  with retries per task, idempotent outputs (deterministic object keys), and a queue per stage.
- Prioritise: short videos and popular creators first; GPUs / hardware encoders for speed.
- Re-encode **popular** videos with better codecs (AV1): encode cost is paid once,
  bandwidth savings are forever.

### Adaptive bitrate streaming (ABR)
Video is cut into **2–10 second segments** at several qualities. A **manifest**
(HLS `.m3u8` / DASH `.mpd`) lists them. The player measures bandwidth and buffer,
then picks the quality **per segment** → when the network drops you see lower
quality instead of a spinner. Segments are plain HTTP files → perfect for CDNs.

### CDN strategy
- A small set of popular videos gets most views → cached at the edge.
- Long tail: regional caches, then origin. An **origin shield** layer collapses many edge misses into one origin request.
- Pre-position (push) predictably popular content overnight (Netflix's approach).
- Signed URLs / tokens against hotlinking; DRM (Widevine / FairPlay) for paid content.

### Metadata and counters
- Metadata DB (sharded SQL or Bigtable-style) + Redis cache.
- **View counts**: play events → Kafka → stream aggregation → batched updates.
  Approximate in real time, corrected later; filter bots and fraud.
- Comments: a separate service/store. Likes: counters as above.

### Search and recommendations
Search index (Elasticsearch) fed by metadata events. Recommendations: offline ML
plus online candidate ranking — mention it, go deeper only if asked.

### Storage cost
Keep the source; drop renditions of never-watched videos and regenerate on demand;
move cold videos to cheaper tiers with lifecycle rules.

## 5. Wrap-up

- Upload straight to storage → parallel transcode DAG → HLS/DASH segments → CDN.
- ABR for smooth playback; counters via stream aggregation.
- Monitor: rebuffering ratio, time-to-first-frame, CDN hit ratio, pipeline queue time, failed transcodes.

## Follow-ups

- "Live streaming?" (Ingest via RTMP/SRT/WebRTC → real-time transcoding → low-latency HLS/DASH with ~2–5 s segments → CDN; chat over WebSockets.)
- "A video goes viral right after upload?" (CDN pull-through with request coalescing + origin shield; pre-warm.)
- "Copyright detection?" (Audio/video fingerprinting inside the pipeline, Content ID style.)

# Case study 16 · Distributed job scheduler (cron at scale / task queue)

**Difficulty:** ⭐⭐⭐⭐ hard. Tests: time-based partitioning, leases, at-least-once execution, idempotency, retries, leader election.

## 1. Requirements

**Functional**
- Submit jobs to run **once at a time** ("send reminder at 09:00 tomorrow") or on a **schedule** (cron: every day at 02:00).
- Jobs run on a pool of workers; see status and history; cancel/pause.
- Retries with backoff on failure; timeouts; optional dependencies (job B after job A).

**Non-functional**
- Scale: e.g. 100M scheduled jobs, 10k jobs due per second at peak.
- Jobs run **on time** (within a few seconds) and **at least once** — never silently skipped.
- Avoid running the same job twice at the same time (and make double runs harmless).
- No single point of failure.

## 2. Data model

```
jobs(job_id, owner, type, payload, schedule (cron or one-time), next_run_at, status,
     max_retries, timeout_s, idempotency_key, created_at)
job_runs(run_id, job_id, scheduled_for, status, attempt, worker_id, lease_expires_at,
         started_at, finished_at, error)
```

## 3. High-level design

```
clients ─► Scheduler API ─► Jobs DB (partitioned by time bucket / job_id)
                                  │
             Scheduler/dispatcher nodes (each owns some partitions)
               every second: find jobs with next_run_at ≤ now → create a job_run → push to queue
                                  │                                   → compute next next_run_at (cron)
                                  ▼
                       Queue (Kafka / SQS), per priority/type
                                  │
                       Worker pool ─► run the job (with timeout) ─► report result
                                  │
                       Retry (backoff) / DLQ / alerts;  status → job_runs
```

## 4. Deep dives

### A. Finding due jobs efficiently
Scanning 100M rows every second is impossible. Options:
- Index on `next_run_at`, and each scheduler node polls **its own partition**:
  `SELECT ... WHERE partition = ? AND next_run_at <= now() ORDER BY next_run_at LIMIT 1000`.
- **Time-bucketed** storage: key jobs by minute (`bucket = 2026-10-13T09:00`); at each
  minute, load that bucket. Far-future jobs sit in cold buckets.
- For short delays: Redis sorted set with score = run time (`ZRANGEBYSCORE due 0 now`), or a
  **hierarchical timing wheel** in memory (how Kafka handles delayed operations).

### B. Who dispatches which jobs? (no duplicates, no gaps)
Partition the jobs (e.g. 1,024 partitions by `job_id`). Each scheduler node holds a
**lease** on a set of partitions via etcd/ZooKeeper (or a DB lease table). If a node dies,
its lease expires and another node takes over those partitions → no job is skipped.

### C. Claiming a job safely
Move a job from `SCHEDULED` to `QUEUED` with a conditional update
(`UPDATE ... SET status='QUEUED' WHERE job_id=? AND status='SCHEDULED' AND next_run_at=?`),
or Postgres `SELECT ... FOR UPDATE SKIP LOCKED` so many dispatchers can grab different rows without blocking.

### D. At-least-once execution
- Workers take a run with a **lease/visibility timeout** (e.g. 5 minutes) and heartbeat to extend it.
- Worker crashes → the lease expires → the run is retried by another worker.
- Therefore a job may run **twice** → jobs must be **idempotent** (idempotency key per
  `job_id + scheduled_for`; the job checks "already done?" or uses conditional writes).
- Use **fencing tokens** (attempt numbers) so a zombie worker that wakes up late can't overwrite a newer attempt's result.

### E. Retries and failures
Exponential backoff with jitter, `max_retries`, then **DLQ** + alert. Distinguish retryable
errors (timeouts, 503) from permanent ones (bad payload). Per-job **timeouts** kill stuck runs.

### F. Recurring (cron) jobs
After dispatching a run, compute the next `next_run_at` from the cron expression (in the
job's **time zone** — careful with daylight saving). If the system was down, decide the
"misfire" policy: run once to catch up, run all missed, or skip.

### G. Fairness and overload
Per-tenant quotas and priority queues so one customer scheduling 1M jobs at 09:00 can't
starve others; spread "top of the hour" spikes with small jitter when exact timing isn't critical.

### H. Dependencies (workflows)
For "B after A", store a DAG; a run becomes eligible when its parents succeed. At that
point you're building a workflow engine — mention Airflow (batch DAGs) and Temporal /
Step Functions (durable workflows) as existing options.

## 5. Wrap-up

- Time-indexed/bucketed job store; partitions leased to scheduler nodes; queue to workers.
- At-least-once with leases + heartbeats; idempotent jobs + fencing; retries, timeouts, DLQ.
- Monitor: dispatch lag (now − scheduled_for), queue depth, failure/retry rates, stuck runs.

## Follow-ups

- "Exactly once?" (Not really — at-least-once + idempotent jobs.)
- "A job must run at exactly 09:00:00.000?" (Expect seconds of jitter; pre-load upcoming jobs into in-memory timers on the owning node.)
- "How do you scale to 1M jobs due at midnight?" (More partitions + workers, queue buffering, jitter, per-tenant limits.)

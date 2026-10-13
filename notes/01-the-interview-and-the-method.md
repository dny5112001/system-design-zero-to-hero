# 01 · What a system design interview really is — and the method to win it

## What they're actually testing

A system design interview is **not** a quiz with one right answer. The
interviewer wants to see whether you can:

1. **Turn a vague request into clear requirements** ("design Instagram" — which parts?).
2. **Make reasonable choices and explain the trade-offs** ("I'll use X because..., the cost is...").
3. **Spot where it will break at scale** and fix it.
4. **Communicate** — lead the conversation, think out loud, take hints.

> The best candidates sound like a senior engineer in a design review, not like
> someone reciting a blog post. **"It depends — and here's what it depends on"**
> beats a memorised diagram every time.

How it's judged, roughly:

| Level | What's expected |
|---|---|
| Junior / new grad | Sensible high-level design; knows the basic building blocks |
| Mid | Clear requirements, solid design, can go deep on 1–2 parts |
| Senior | Drives the whole interview, explains trade-offs, handles failure cases and scale |
| Staff+ | Also: operations, cost, evolution over time, org/team impact, very deep dives |

## The method: 7 steps in ~45 minutes

Use the same structure every time. It keeps you calm and shows maturity.

```
1. Requirements          5 min   What exactly are we building?
2. Estimates             3 min   How big? (users, requests/sec, storage)
3. API                   3 min   How do clients talk to it?
4. Data model            3 min   What do we store, and where?
5. High-level design    10 min   Boxes and arrows for the main flows
6. Deep dives           15 min   The 2–3 hardest parts, in detail
7. Wrap-up               5 min   Bottlenecks, failures, trade-offs, what's next
```

### Step 1 — Requirements (never skip this)

Ask questions before drawing anything. Split into two lists:

**Functional** — what the system **does**:
- "Users can post a photo, follow others, and see a feed of people they follow."
- Pick the **3–4 core features** and say what's out of scope: *"I'll leave out
  stories, ads and DMs unless you'd like me to cover them."*

**Non-functional** — how **well** it does it:

| Quality | Typical question |
|---|---|
| Scale | How many users? Daily active users (DAU)? Read-heavy or write-heavy? |
| Latency | How fast must the feed load? (e.g. < 200 ms) |
| Availability | Is it OK to be down 1 minute a month? 1 hour? |
| Consistency | Must everyone see a new post instantly, or is a few seconds OK? |
| Durability | Can we ever lose data? (photos: never; view counts: a bit is OK) |
| Other | Security, privacy, cost, global users, mobile clients |

Write both lists on the board. **Refer back to them** when making choices:
*"Because we said availability matters more than instant consistency for the
feed, I'll use eventual consistency here."*

### Step 2 — Back-of-the-envelope estimates

Quick maths to decide **how big** the design must be (chapter 02). Round
aggressively. Only estimate what changes the design:

- requests per second (reads and writes separately),
- storage per day / over 5 years,
- bandwidth, and maybe memory for a cache.

### Step 3 — API

A few endpoints, written simply:

```
POST /v1/posts            { image_id, caption }          -> { post_id }
GET  /v1/feed?cursor=...  -> { posts: [...], next_cursor }
POST /v1/users/{id}/follow
```

This pins down exactly what the system must do.

### Step 4 — Data model

Main entities, their key fields, and **which database** and why:

```
User(id, name, ...)                 -> SQL
Post(id, user_id, image_url, ...)   -> SQL or wide-column, sharded by user_id
Follow(follower_id, followee_id)    -> SQL / graph
Images                              -> object storage (S3) + CDN
```

### Step 5 — High-level design

Draw the main path end to end: client → load balancer → services → databases,
caches, queues. Walk through **one write** and **one read** out loud.
Keep it simple first; make it work, then make it scale.

### Step 6 — Deep dives

Pick the hardest parts — or let the interviewer pick. Typical ones:
- the hot path at scale (feed generation, matching drivers),
- data partitioning and hot spots,
- consistency of a critical operation (payments, bookings),
- failure handling (what if this service/region dies?).

For each, offer **options with trade-offs**, then **choose**.

### Step 7 — Wrap-up

- Bottlenecks and single points of failure, and how you'd fix them.
- Monitoring and alerting: what would you watch?
- What you'd do with more time / at 10× scale.

## Golden rules

1. **Clarify first, design second.** Jumping straight to "I'll use Kafka" is the #1 mistake.
2. **Think out loud.** Silence gives the interviewer nothing to grade.
3. **Start simple, then evolve.** One server → then scale it. Show the reasoning.
4. **Every choice has a cost.** Name it: "Caching makes reads fast, but data can be stale for up to the TTL."
5. **Use numbers.** "50k reads/sec is too much for one database, so..."
6. **Drive, but listen.** If the interviewer nudges ("what about hot users?"), follow it — that's where the points are.
7. **Don't name-drop without reasons.** "Cassandra, because writes are huge, access is by key, and we can accept eventual consistency" — good. "Cassandra because it scales" — weak.
8. **Manage time.** Don't spend 20 minutes on requirements or 30 on one box.

## Phrases that help

- "Before I design, let me make sure I understand the scope..."
- "I'll assume X; let me know if that's wrong."
- "There are two main options here... I'd pick the second because..., accepting that..."
- "The bottleneck here is..., so I'd..."
- "If this component fails, then..."
- "To keep it simple I'll start with one region, then discuss going global."

## How to study with these notes

1. Read chapters 02–16 to learn the **building blocks** and their trade-offs.
2. Work through the **case studies** — first try each one yourself for 30 minutes, then compare.
3. Practise **out loud** with a friend or a timer. Draw on paper or Excalidraw.
4. Use chapter 17 (cheat sheet) the night before.

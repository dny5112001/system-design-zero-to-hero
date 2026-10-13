# 13 · API design

The API is the **contract** with clients. In interviews, a few clear endpoints
early on make the rest of the design precise.

## REST basics

Resources as **nouns**, actions as **HTTP verbs**:

```
GET    /v1/users/42              read
POST   /v1/users                 create          → 201 Created + Location header
PUT    /v1/users/42              replace
PATCH  /v1/users/42              partial update
DELETE /v1/users/42              delete          → 204 No Content
GET    /v1/users/42/orders       nested collection
POST   /v1/orders/99/cancel      an action that isn't plain CRUD (fine to use a verb here)
```

- Consistent naming (plural nouns, lower-case, hyphens).
- Correct status codes (chapter 04) and a consistent error body:
  `{ "error": { "code": "card_declined", "message": "...", "request_id": "..." } }`.
- **Versioning**: `/v1/...` in the path (most common) or a header. Never break
  existing clients; add fields, don't rename/remove them.

## Pagination — never return "everything"

| Type | How | ✅ | ❌ |
|---|---|---|---|
| **Offset** | `?limit=20&offset=40` | simple; jump to page N | slow on deep pages (DB still skips rows); items shift if data changes |
| **Cursor (keyset)** | `?limit=20&cursor=<last_seen_id_or_timestamp>` | fast at any depth, stable with new inserts | no "jump to page 57" |

Feeds, timelines, chats → **cursor**. Return `next_cursor` (often an encoded
`(created_at, id)` pair) in the response.

## Idempotency — making retries safe

Networks fail **after** the server did the work but **before** the client got
the answer. The client retries → double charge?

- GET/PUT/DELETE are naturally idempotent. **POST is not.**
- Solution: an **idempotency key** — the client generates a unique ID per
  logical operation and sends it:

```
POST /v1/payments
Idempotency-Key: 6f1c...e2
```

Server: store `(key → result)` for e.g. 24 hours. Same key again → return the
**stored result** instead of doing it twice. (Stripe's API is the famous example.)
Handle the race: two identical requests arriving together → unique constraint on
the key, second one waits or gets 409.

## Rate limiting (from the API's point of view)

Protect the service and enforce fair use: return **429 Too Many Requests** with
`Retry-After` and `X-RateLimit-Limit/Remaining/Reset` headers. Designing the
limiter itself is a case study.

## Other API concerns

- **Authentication**: API keys (server-to-server), OAuth 2.0 / OIDC + JWT access
  tokens (users), mTLS (internal services).
- **Authorization**: check per resource ("is this user allowed to read order 99?") — the #1 API security bug (broken object-level authorization).
- **Filtering/sorting/fields**: `?status=paid&sort=-created_at&fields=id,total`.
- **Long operations**: return **202 Accepted** + a job ID; client polls
  `GET /jobs/{id}` or receives a **webhook** callback.
- **Webhooks** (your server calls theirs): sign payloads (HMAC), retry with
  backoff, consumers must be idempotent.
- **Batch endpoints** to reduce chatty calls.
- **Compression** (gzip/brotli), **ETags** for conditional GETs (304 Not Modified).

## REST vs GraphQL vs gRPC

| | REST | GraphQL | gRPC |
|---|---|---|---|
| Shape | many endpoints | one endpoint, client picks fields | RPC methods from a `.proto` contract |
| Format | JSON | JSON | binary Protobuf |
| Strengths | simple, cacheable (HTTP caching, CDN), universal | no over/under-fetching; great for varied UIs | fast, typed, streaming, code generation |
| Weaknesses | over/under-fetching, many round trips | caching harder, complex queries can be expensive (need limits) | not browser-native, less human-readable |
| Typical use | public APIs, mobile/web backends | product frontends with complex data needs | internal service-to-service |

## In the interview

Write 3–6 endpoints with key parameters and responses. Mention pagination,
idempotency for anything that creates/charges, and auth. Don't spend more than
~3 minutes.

## Remember

- Nouns + verbs, versioned, consistent errors.
- Cursor pagination for feeds; idempotency keys for payments/creates.
- REST outside, gRPC inside, GraphQL when clients need flexible shapes.

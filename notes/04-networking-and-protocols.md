# 04 · Networking and protocols you need

## What happens when you type a URL (classic question)

```
1. DNS:   "shop.com" → IP address (browser cache → OS → resolver → root → .com → shop.com's DNS)
2. TCP:   3-way handshake with that IP (SYN, SYN-ACK, ACK)
3. TLS:   agree on encryption keys, check the certificate (HTTPS)
4. HTTP:  send "GET /", maybe via a CDN / load balancer
5. Server builds the response (app → cache → database)
6. Browser renders HTML, then fetches CSS/JS/images (often from a CDN)
```

## DNS — the internet's phone book

Turns names into IP addresses. Answers are **cached** for a time-to-live (TTL).
- **Records**: A (name → IPv4), AAAA (IPv6), CNAME (alias), MX (mail), TXT, NS.
- **Uses in design**: simple load balancing (several IPs), **geo-DNS** (send users
  to the nearest region), failover (point away from a dead region).
- Downside: caching means changes take up to the TTL to spread.

## TCP vs UDP

| | TCP | UDP |
|---|---|---|
| Delivery | guaranteed, in order, retransmits lost packets | best effort, may lose/reorder |
| Connection | yes (handshake) | no |
| Speed | more overhead | lighter, lower latency |
| Used for | web, APIs, databases, file transfer | video calls, gaming, DNS, live streaming, QUIC (HTTP/3) |

## HTTP essentials

- **Methods**: GET (read), POST (create), PUT (replace), PATCH (partial update), DELETE.
- **Idempotent** = doing it twice has the same effect as once: GET, PUT, DELETE are; POST isn't (chapter 13).
- **Status codes**: 2xx ok (200, 201 created, 204 no content) · 3xx redirect (301 permanent, 302 temporary, 304 not modified) · 4xx your fault (400, 401 not logged in, 403 not allowed, 404, 409 conflict, 429 too many requests) · 5xx server fault (500, 502 bad gateway, 503 unavailable, 504 timeout).
- **Headers** worth knowing: `Cache-Control`, `ETag`, `Authorization`, `Content-Type`, `Retry-After`.

### HTTP versions

| Version | Key idea |
|---|---|
| HTTP/1.1 | one request at a time per connection; keep-alive reuses connections |
| HTTP/2 | many requests in parallel over **one** connection (multiplexing), header compression |
| HTTP/3 | runs over **QUIC (UDP)**: faster setup, no head-of-line blocking, survives network switches (Wi-Fi → mobile) |

### HTTPS / TLS

Encrypts traffic and proves the server is who it claims (certificates).
**TLS termination** is often done at the load balancer, which then talks to
servers over the private network (or re-encrypts for zero-trust/mTLS).

## Real-time communication — how does the server push to the client?

| Technique | How | Good for |
|---|---|---|
| **Short polling** | client asks every N seconds | simple, low-frequency updates; wasteful |
| **Long polling** | client asks; server holds the request open until there's news (or timeout) | fallback when WebSockets aren't possible |
| **Server-Sent Events (SSE)** | one long HTTP response the server keeps writing to; **server → client only** | live scores, notifications, AI token streaming |
| **WebSocket** | upgrade HTTP into a **two-way** persistent connection | chat, multiplayer, collaborative editing |
| **WebRTC** | peer-to-peer media/data (UDP) | video/voice calls |

WebSockets are **stateful** connections: each user stays connected to one server.
So you need: a way to find **which server holds user X's connection**
(a registry in Redis), pub/sub between servers, and reconnection logic.
(Case study: chat.)

## API styles

| Style | What | When |
|---|---|---|
| **REST** | resources + HTTP verbs, JSON | public APIs, most web/mobile backends |
| **GraphQL** | client asks exactly for the fields it needs, one endpoint | many clients with different data needs; avoids over/under-fetching |
| **gRPC** | binary (Protobuf), HTTP/2, generated clients, streaming | fast service-to-service calls inside your system |

## Inside vs outside your system

- **Public internet → you**: HTTPS, DNS, CDN, load balancers, API gateway, WAF.
- **Between your services**: private network (VPC), gRPC/HTTP, service discovery, mTLS.

## Remember

- URL → DNS → TCP → TLS → HTTP → server → render. Know it cold.
- TCP = reliable, UDP = fast; HTTP/2 multiplexes, HTTP/3 uses QUIC over UDP.
- Push to clients with SSE (one-way) or WebSockets (two-way); WebSockets need a connection registry.

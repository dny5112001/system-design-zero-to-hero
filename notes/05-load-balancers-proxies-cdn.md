# 05 · Load balancers, proxies, API gateways and CDNs

## Load balancer — the traffic director

Spreads requests across many servers so no single one is overwhelmed, and stops
sending traffic to dead ones.

```
           ┌──► server 1
client ──► LB ──► server 2
           └──► server 3   (health check failed → skipped)
```

### Layer 4 vs Layer 7

| | L4 (transport) | L7 (application) |
|---|---|---|
| Looks at | IP + port only | full HTTP: path, headers, cookies |
| Speed | faster, simpler | a bit more work |
| Can do | spread TCP/UDP connections | route `/api` vs `/images`, A/B tests, auth, TLS termination, rewrite headers |
| Examples | AWS NLB, LVS | AWS ALB, NGINX, Envoy, HAProxy, Traefik |

### Algorithms

| Algorithm | How | Note |
|---|---|---|
| Round robin | take turns | simple; assumes equal servers |
| Weighted round robin | bigger servers get more | mixed hardware |
| Least connections | send to the least busy | good for long requests/WebSockets |
| Least response time | fastest recently | adaptive |
| IP / consistent hash | same client/key → same server | caches, sticky behaviour (chapter 09) |
| Random / "power of two choices" | pick 2 at random, use the less busy | surprisingly good at scale |

### Health checks

The LB pings `/health` regularly; failing servers are removed until they recover.
Make the check meaningful but cheap (chapter 16: liveness vs readiness).

### The load balancer itself must not be a SPOF

Run 2+ (active-passive with a floating IP, or active-active), or use a managed
cloud LB, which is redundant by design. DNS can spread across LBs in several regions.

## Reverse proxy vs forward proxy

- **Forward proxy** sits in front of **clients** (company proxy, VPN): hides who the client is.
- **Reverse proxy** sits in front of **servers**: hides the servers; adds TLS,
  caching, compression, rate limiting. A load balancer is a kind of reverse proxy.

## API gateway — the front desk for your APIs

A reverse proxy specialised for APIs, usually at the edge of a microservice system:
- one entry point that **routes** to many services,
- **authentication**, **rate limiting**, request validation,
- request/response transformation, API keys, usage metering,
- sometimes aggregation (one call → several services).

Examples: AWS API Gateway, Kong, Apigee, Envoy-based gateways.
Watch out: it can become a bottleneck or a "god service" — keep business logic out of it.

**BFF (Backend for Frontend)**: a small gateway per client type (web, iOS,
Android) shaping data the way that client needs.

## CDN — Content Delivery Network

Copies of your content on **edge servers worldwide**, near users. A user in
Mumbai gets the image from a Mumbai edge, not from Virginia.

```
user ──► nearest CDN edge ──(cache miss only)──► your origin (S3 / servers)
```

- **What to put on it**: images, video, JS/CSS, downloads; also cacheable API
  responses; increasingly, code at the edge (Cloudflare Workers, Lambda@Edge).
- **Pull CDN**: edge fetches from origin on first request, then caches. Easy; first request is slow.
- **Push CDN**: you upload content to the CDN ahead of time. Good for big, predictable files.
- **Invalidation**: purge by URL, or better, **versioned file names**
  (`app.3f9a1c.js`) so new versions are new URLs and old caches don't matter.
- **Benefits**: lower latency, much less load and bandwidth on your origin, DDoS absorption.

## Putting it together (typical web stack)

```
Users ─► DNS (geo) ─► CDN ─► WAF ─► Load balancer / API gateway ─► app servers (stateless)
                                                                      │
                                              cache (Redis) ◄─────────┤
                                              database(s)   ◄─────────┤
                                              queue ─► workers ◄──────┘
                                              object storage (S3) ◄── CDN origin
```

## Remember

- L4 balances connections; L7 understands HTTP and can route smartly.
- LBs need health checks and redundancy themselves.
- API gateway = auth + rate limits + routing at the edge. CDN = content near users.

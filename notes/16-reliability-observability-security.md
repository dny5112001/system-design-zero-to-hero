# 16 · Reliability, observability and security

The "wrap-up" topics that separate senior answers from the rest.

## Reliability

### Redundancy at every layer

| Layer | Redundancy |
|---|---|
| Servers | N+1 or more stateless instances behind a load balancer |
| Zones | spread across **availability zones** (separate data centres in a region) |
| Data | replicas + backups (backups are different from replicas — a bad `DELETE` replicates instantly!) |
| Regions | for disaster recovery or global users |
| Dependencies | fallbacks, multiple providers for critical ones (payments, SMS) |

### Disaster recovery: RPO and RTO

- **RPO** (Recovery Point Objective): how much **data** can we lose? (last 5 minutes? zero?)
- **RTO** (Recovery Time Objective): how long can we be **down**? (1 hour? 1 minute?)

| Strategy | RPO / RTO | Cost |
|---|---|---|
| Backups + restore | hours / hours | 💲 |
| Pilot light (minimal standby) | minutes / tens of minutes | 💲💲 |
| Warm standby (scaled-down copy running) | seconds–minutes / minutes | 💲💲💲 |
| Multi-region active-active | ~0 / ~0 | 💲💲💲💲 |

**Test restores and failovers regularly** — untested backups don't count.

### Health checks

- **Liveness**: is the process alive? (restart if not) — never check dependencies.
- **Readiness**: can it serve now? (remove from load balancer if not) — may check dependencies.

### Safe deployments

- **Rolling**: replace instances gradually.
- **Blue-green**: run the new version alongside, switch traffic at once, switch back instantly if bad.
- **Canary**: send 1% → 10% → 100% of traffic to the new version, watching metrics.
- **Feature flags**: ship code turned off; turn it on gradually, off instantly.
- **Backward-compatible DB migrations** (expand → migrate → contract) so old and new code both work during the rollout.

### Chaos engineering
Deliberately break things (kill instances, add latency) in a controlled way to
prove the system survives (Netflix's Chaos Monkey).

## Observability — knowing what's happening

| Pillar | Answers | Tools |
|---|---|---|
| **Metrics** | "is something wrong, and how much?" (numbers over time) | Prometheus, Grafana, Datadog, CloudWatch |
| **Logs** | "what exactly happened?" (structured JSON events) | ELK/OpenSearch, Loki, Splunk |
| **Traces** | "where did this request spend its time across services?" | OpenTelemetry, Jaeger, Tempo, X-Ray |

Pass a **request/trace ID** through every service and log it — then you can follow one request end to end.

### What to measure

- **Golden signals** (Google SRE): **latency, traffic, errors, saturation**.
- **RED** for services: Rate, Errors, Duration.
- **USE** for resources: Utilization, Saturation, Errors.
- Business metrics: orders/min, sign-ups, payment success rate — often the first sign of trouble.

### Alerting
Alert on **user-facing symptoms** (error rate, p99 latency, SLO burn rate), not
on every CPU spike. Every alert should be actionable, with a runbook.

## Security essentials

| Area | Key points |
|---|---|
| **Authentication** (who are you?) | passwords hashed (bcrypt/Argon2), MFA, OAuth2/OIDC, SSO; short-lived access tokens + refresh tokens |
| **Authorization** (what may you do?) | RBAC/ABAC; check on **every** request for **every** object; least privilege |
| **In transit** | TLS everywhere; mTLS between services |
| **At rest** | encrypt disks/databases/backups; keys in a KMS; rotate them |
| **Secrets** | a secrets manager (Vault, AWS Secrets Manager), never in code or images |
| **Input** | validate everything; parameterised queries (SQL injection); output encoding (XSS); CSRF protection |
| **Edge** | WAF, DDoS protection (CDN/Cloudflare/Shield), rate limiting, bot detection |
| **Network** | private subnets for databases; security groups; zero-trust between services |
| **Data** | collect less; mask/tokenise PII and card data (PCI DSS); data residency (GDPR, India's DPDP Act); retention and deletion |
| **Audit** | immutable audit logs for sensitive actions |

## Cost (a staff-level touch)

Mention it: CDN and caching cut bandwidth/DB cost; object storage tiers;
autoscaling; spot instances for batch jobs; pick consistency/replication levels
that the business actually needs; avoid cross-region data transfer where possible.

## Remember

- Redundancy across zones, real backups, defined RPO/RTO, tested recovery.
- Metrics + logs + traces with a shared request ID; alert on user-facing symptoms.
- Authn, authz on every object, TLS everywhere, secrets in a manager, least privilege.

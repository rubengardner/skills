# Specialist I — Anti-Patterns

These are the patterns that kill systems in production. Treat all of them as hard errors in design reviews.

---

## SCALABILITY ANTI-PATTERNS

### 1. Premature Distribution

**The pattern:** Designing microservices or distributed systems before proving a monolith is insufficient.

**The harm:** Distributed systems multiply operational cost, debugging complexity, and failure modes. Teams that distribute prematurely get all the cost with none of the benefit.

**Evidence:** Shopify handles 30 TB/minute on a modular monolith. Stack Overflow serves 2 billion page views/month on a handful of servers.

**Fix:** Start with a modular monolith. Extract services only when you have a measured bottleneck that cannot be solved by scaling the whole monolith.

---

### 2. Stateful Application Servers

**The pattern:** Storing session data, user state, or cached results in the application process's memory.

**The harm:** Sticky sessions → cannot load-balance freely. Instance replacement → users lose sessions. Deploy → state lost.

**Fix:** Every piece of state that must survive a request lives in the data layer: Redis (sessions, ephemeral state), database (persistent state), object storage (files).

---

### 3. Missing Cache Invalidation

**The pattern:** Setting a 5-minute TTL and assuming the cache will eventually be correct.

**The harm:** Users see stale data for up to TTL after a write. After your update, your own data shows the old version. This is the most common source of "the UX is broken" bugs.

**Fix:** Invalidate cache keys on write. Use TTL only as a fallback safety net, not the primary invalidation mechanism.

---

### 4. Offset Pagination on Large Tables

**The pattern:** `SELECT * FROM orders ORDER BY id OFFSET 50000 LIMIT 20`

**The harm:** `OFFSET N` scans and discards N rows before returning results. At high page numbers, it is slower than a full table scan. Results are also unstable if rows are inserted/deleted between pages.

**Fix:** Cursor-based pagination: `WHERE id > :cursor ORDER BY id LIMIT 20`. Stable, fast regardless of position.

---

## RELIABILITY ANTI-PATTERNS

### 5. Missing Timeouts

**The pattern:** `httpx.get(url)` with no timeout parameter.

**The harm:** One slow upstream holds one thread indefinitely. With enough concurrent requests to slow upstreams, all threads are blocked. The service becomes unresponsive.

**Fix:** Every outbound call has an explicit timeout aligned to the SLO budget. Every database call has a statement timeout. Every Redis call has a socket timeout.

---

### 6. Retrying Permanent Errors

**The pattern:**
```python
@celery_task(max_retries=5, autoretry_for=(Exception,))
def process(record):
    validate_schema(record)  # raises ValueError if schema is invalid → retried 5 times
```

**The harm:** Wastes queue resources. Delays routing to DLQ. Creates illusion of progress during guaranteed-to-fail retries.

**Fix:** Classify errors before retrying. `ValueError`, HTTP 400, HTTP 422, constraint violations → permanent, DLQ immediately. `TimeoutError`, HTTP 503, HTTP 429 → transient, retry with backoff.

---

### 7. No Circuit Breaker on Downstream Calls

**The pattern:** Calling an external service with only a timeout, no circuit breaker.

**The harm:** The downstream service goes slow (not down). Timeouts queue up. Thread pool exhausts. Your service becomes unresponsive. This is cascading failure — one downstream problem takes down your entire service.

**Fix:** Circuit breaker on every external call. When the downstream is struggling, fail fast instead of waiting for timeouts.

---

### 8. Shared Database Across Services

**The pattern:** Two services read and write to the same database tables.

**The harm:** Services are tightly coupled through the database schema. Changing a table requires coordinating all services that use it. One service's bad query can starve the other's connection pool. This is the "distributed monolith" trap — you have the operational complexity of microservices with the coupling of a monolith.

**Fix:** Each service owns its data. Cross-service data access happens through API calls or event subscriptions, not shared table access.

---

### 9. "Set and Forget" Alerts

**The pattern:** Alerting on raw metrics: `CPU > 80%`, `memory > 85%`.

**The harm:** Too many false positives (noise). Alerts fire during normal load spikes. On-call ignores alerts because they rarely indicate real user impact. Real incidents are missed in the noise.

**Fix:** Alert on user-visible symptoms (error rate, latency) and SLO burn rate. Raw resource metrics are useful in dashboards; they are poor alert signals.

---

## MAINTAINABILITY ANTI-PATTERNS

### 10. No Architecture Decision Records

**The pattern:** Important architectural decisions made in Slack threads, Jira tickets, or people's heads.

**The harm:** Engineers leave. Context is lost. Three years later, no one knows why the system uses X instead of Y. New engineers either blindly preserve bad decisions or blindly reverse good ones.

**Fix:** ADRs in the repo for every irreversible decision. One page, in plain language, next to the code it governs.

---

### 11. The Distributed Monolith

**The pattern:** Microservices that communicate synchronously, share databases, or must be deployed together.

**The harm:** You have all the operational cost of microservices (distributed traces, network calls, partial failures) with none of the independence benefits (separate deployments, separate scaling, team autonomy).

**Telltale signs:**
- "We have to deploy service A and service B together."
- "Service A calls service B, which calls service C synchronously — a chain of 5 services."
- "Both services write to the users table."

**Fix:** If services cannot be deployed and scaled independently, they should be one service (or one module in a monolith). If they must be separate services, decouple through events and separate data stores.

---

### 12. No Observability Until Production Breaks

**The pattern:** "We'll add monitoring later." Launching with `print()` statements and no metrics.

**The harm:** First production incident: you are blind. You cannot measure the impact. You cannot identify the cause. You cannot confirm the fix worked.

**Fix:** Observability is implemented before the first production deployment, not after. Structured logging, basic metrics (request rate, error rate, latency), and health checks are non-negotiable day-one requirements.

---

### 13. Long-Lived Feature Branches

**The pattern:** Feature branches that live for weeks. Merged with a massive PR (> 500 lines).

**The harm:** Long branches accumulate merge conflicts. Large PRs get superficial reviews. Deployment risk is concentrated in one big release. This is the single largest predictor of high Change Failure Rate (DORA).

**Fix:** Trunk-based development with feature flags. Ship incomplete features behind a flag. Small PRs (< 200 lines) reviewed and merged daily.

---

### 14. Synchronous Chains of Services

**The pattern:** Request A → Service B → Service C → Service D → response to A.

**The harm:** Latency compounds: 100ms + 150ms + 80ms = 330ms minimum for A, plus network overhead. Reliability compounds: if each service has 99.9% availability, the chain has 99.7% (0.999³). Adding services makes it worse.

**Fix:** Identify which steps must be synchronous (the user needs the result immediately) and which can be async (fire and forget, eventual consistency is acceptable). Move the async steps to a queue.

---

## QUICK ANTI-PATTERN CHECKLIST

Raise a flag in code or design review for any of these:

- [ ] Application server stores session/state in local memory
- [ ] `httpx.get(url)` or `requests.get(url)` with no timeout
- [ ] `autoretry_for=(Exception,)` in a Celery task
- [ ] No circuit breaker on any external service call
- [ ] Two services accessing the same database table
- [ ] `SELECT ... OFFSET N` on a large table
- [ ] No SLO defined for a production service
- [ ] No rollback plan for a deployment
- [ ] Microservices that must be deployed together
- [ ] No ADR for a decision that is expensive to reverse
- [ ] Service in production with no structured logs or metrics
- [ ] Feature branch older than 5 days
- [ ] PR larger than 500 lines with no feature flag

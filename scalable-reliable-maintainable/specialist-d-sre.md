# Specialist D — SRE & Reliability Engineering

## SOURCE

Most of this comes directly from the Google SRE Book (free at sre.google/sre-book) and SRE Workbook (sre.google/workbook). If you are serious about reliability engineering, read both.

---

## 1. SLI / SLO / SLA / ERROR BUDGET

These four concepts form the entire measurement framework for reliability. Define them before you build.

### Definitions

**SLI (Service Level Indicator)**
A quantitative measurement of service behavior. Must be something you can actually measure.

Common SLIs:
- **Availability**: `successful_requests / total_requests` (a 503 is not successful; a 200 is)
- **Latency**: % of requests completing in under X milliseconds (p99, p95, p50)
- **Throughput**: requests per second
- **Error rate**: `failed_requests / total_requests`
- **Freshness**: how recently the data was updated (for data pipelines)

**SLO (Service Level Objective)**
An internal target threshold for an SLI, measured over a rolling window.

```
SLO = "99.9% of API requests complete in < 200ms over a rolling 28-day window"
SLO = "99.5% availability measured over a rolling 28-day window"
```

- This is what your team commits to internally.
- Set the SLO at the threshold where users become unhappy — not at 100%.
- Rolling 28-day windows are preferred over calendar months (user experience doesn't reset on the 1st).
- Aim for multiple SLOs covering different dimensions: availability + latency + freshness.

**SLA (Service Level Agreement)**
The external, contractual commitment to customers. Always set lower than the SLO.

```
SLO = 99.9% availability  →  SLA = 99.5% availability
```

The gap between SLO and SLA is your warning zone: you breach your SLO before you breach your SLA. Fix the problem before contractual consequences.

**Error Budget**
```
Error Budget = 1 - SLO
99.9% SLO → 0.1% error budget = 43.8 minutes/month of allowed downtime
99.0% SLO → 1.0% error budget = 7.3 hours/month of allowed downtime
```

The error budget is the most powerful reliability tool: it is a shared resource owned by both engineering and product. Spending it on risky deployments is legitimate. Running it to zero through incidents is not.

---

## 2. ERROR BUDGET POLICY

The error budget turns reliability from a constraint into a negotiating tool.

```
Budget > 50% remaining:
  → Deploy freely. Run experiments. Ship features.

Budget 25–50% remaining:
  → Increase monitoring frequency. Slow risky releases.
  → Require extra testing for high-risk changes.

Budget < 25% remaining:
  → Pause all non-critical feature work.
  → Only security patches and P0 bug fixes.
  → Engineering focus shifts to reliability work.

Budget exhausted (0%):
  → Feature freeze until budget recovers.
  → Mandatory postmortem on all incidents in the window.
  → No new experiments.
```

This policy makes reliability a product decision, not just an engineering constraint. When product wants to ship a risky feature, the cost is measured in error budget, not just "risk."

---

## 3. PRODUCTION READINESS REVIEW

Before any new service goes to production, review it against this checklist:

**Service fundamentals**
- [ ] SLOs defined (availability + latency at minimum)
- [ ] Error budget policy documented
- [ ] Runbook written: what to do when the service is paged

**Reliability**
- [ ] All outbound calls have timeouts configured
- [ ] Retry logic with backoff for transient failures
- [ ] Circuit breakers on critical dependencies
- [ ] Graceful shutdown implemented (handles SIGTERM, drains in-flight requests)
- [ ] Load tested at 2× expected peak traffic

**Observability**
- [ ] Structured logs emitted on every request (with correlation ID)
- [ ] Metrics: request rate, error rate, latency (p50, p95, p99)
- [ ] Traces instrumented (OpenTelemetry)
- [ ] Alerting configured on SLO burn rate (not just raw metrics)
- [ ] Dashboard deployed

**Deployment and rollback**
- [ ] Deployment can be rolled back in < 5 minutes
- [ ] Feature flags available for new functionality
- [ ] Health check endpoint (`/healthz`) implemented and tested
- [ ] Database migrations are backward-compatible (old code can run against new schema)

**Operations**
- [ ] On-call rotation defined
- [ ] Incident response playbook exists
- [ ] Dependency SLOs documented (what can this service tolerate losing?)

---

## 4. CHAOS ENGINEERING

> "If it hurts, do it more often." — Netflix

Chaos engineering is the practice of deliberately injecting failure into production (or near-production) systems to discover weaknesses before they become incidents.

**The 5 Principles (from principlesofchaos.org):**

1. **Define steady state** — what does "normal" look like in measurable terms (request rate, error rate, latency)?
2. **Hypothesize** — "steady state will hold in both the control group and the experiment group."
3. **Vary real-world events** — crash a node, inject latency, fill the disk, partition the network.
4. **Run in production** (or as close as possible) — staging misses production failure modes.
5. **Automate experiments continuously** — one-off experiments give one-time confidence. Continuous experiments catch regressions.

**Starting with chaos (progressive approach):**

```
Level 1: Kill a single process/container. Does the service recover automatically?
Level 2: Kill an instance. Does the load balancer reroute? Does a new instance start?
Level 3: Inject network latency (200ms) on calls to dependency X. Does the circuit breaker trip?
Level 4: Kill an entire availability zone. Does traffic route to other zones?
Level 5: Exhaust disk space on the database node. What happens to writes?
```

**Tools:** Chaos Monkey (Netflix OSS), Gremlin, LitmusChaos (Kubernetes-native), AWS Fault Injection Simulator.

**Run chaos experiments with a kill switch.** You must be able to stop the experiment immediately if unexpected harm occurs. Never run chaos experiments without a monitoring dashboard open and an incident response plan ready.

---

## 5. BLAMELESS POSTMORTEM

After every significant incident (anything that consumes > 10% of monthly error budget, or requires on-call wakeup), write a blameless postmortem.

**The principle:** People are intelligent, well-intentioned, and working within systems. If they made a mistake, the system made it too easy to make. Fix the system, not the person.

**The format:**

```markdown
## Incident Title
Short noun phrase: "Order service latency spike caused by Redis connection pool exhaustion"

## Impact
- Duration: 14:23 UTC – 15:07 UTC (44 minutes)
- Error rate peaked at 34%
- ~12,000 orders affected; ~800 failed permanently

## Timeline
14:23 — First alert fires: p99 latency > 2s
14:31 — On-call acknowledges, starts investigation
14:45 — Root cause identified: Redis maxconn reached
14:52 — Connection pool limit increased, deployed
15:07 — Error rate returns to baseline

## Root Cause
Redis connection pool limit (default: 10) was never changed from the default.
Under Black Friday load (8× normal), all 10 connections were held by slow background jobs,
causing all synchronous API requests to queue waiting for a connection.

## Contributing Factors
- No alerting on Redis connection pool utilization
- Production readiness review did not check connection pool configuration
- Load test was run at 2× normal load, not 8×

## Action Items
| Action | Owner | Due |
|--------|-------|-----|
| Increase Redis maxconn to 100, add autoscaling | @alice | 2024-03-20 |
| Add alert: Redis connection pool utilization > 80% | @bob | 2024-03-18 |
| Add connection pool config to production readiness checklist | @carol | 2024-03-22 |
| Run load test at 10× normal for all future services | @dave | 2024-03-25 |
```

**What a postmortem is NOT:** a performance review, an opportunity to assign blame, a summary of what the on-call person did wrong.

---

## 6. SLO ALERTING

Alert on **SLO burn rate**, not raw metrics. Raw metric alerts fire too late (you've already burned the budget) or too early (a spike that recovers quickly). Burn rate tells you how fast you're consuming your error budget.

```
Burn rate = current_error_rate / error_budget_rate

If SLO = 99.9% (0.1% budget), and your current error rate is 1%:
Burn rate = 1% / 0.1% = 10×

At burn rate 10×, you'll exhaust the monthly budget in 3 days.
```

**Multi-window burn rate alerting (Google's recommendation):**

| Window | Burn rate threshold | Alert severity |
|--------|--------------------|--------------------|
| 1h + 5m | > 14.4× | Page immediately (critical) |
| 6h + 30m | > 6× | Page (high) |
| 3d + 6h | > 1× | Ticket (low) |

Short windows catch fast burns. Long windows catch slow, persistent degradation. Use both.

# Specialist H — Maintainability

## THE CORE INSIGHT

A system is only as maintainable as the team's ability to understand it, change it safely, and deploy changes with confidence. Maintainability is not a property of the code alone — it is a property of the code + the team + the process + the tooling.

---

## 1. DORA METRICS

DORA (DevOps Research and Assessment, now part of Google) identified the four metrics most strongly predictive of software delivery performance. Published annually in the State of DevOps Report.

**The metrics:**

| Metric | Measures | Elite benchmark |
|--------|----------|----------------|
| **Deployment Frequency** | How often code reaches production | Multiple times per day |
| **Change Lead Time** | First commit → running in production | < 1 hour |
| **Change Failure Rate** | % of deployments that cause an incident | 0–5% |
| **Failed Deployment Recovery Time** | Detection → full recovery | < 1 hour |

**The key finding:** Speed and stability are not trade-offs. Elite performers deploy frequently AND have low failure rates. Teams that slow down to improve stability get neither.

**How to use DORA:**
- Use as a diagnostic compass for systemic problems, not as individual performance metrics.
- High lead time → look at branch strategy, PR review lag, test suite speed.
- High failure rate → look at test coverage, missing canary deployments, lack of feature flags.
- Slow recovery → look at rollback capability, missing runbooks, alert noise drowning out real signals.

```python
# Calculate change lead time from git log
import subprocess
from datetime import datetime

def calculate_lead_time(days: int = 30) -> float:
    """Average time (hours) from first commit to deploy for last N days."""
    log = subprocess.check_output([
        "git", "log", "--format=%H %ct",
        f"--since={days} days ago", "origin/main"
    ]).decode()

    # Compare commit timestamps to deploy timestamps in your CI/CD system
    # This is the outline — implement with your actual deploy record source
    ...
```

---

## 2. ARCHITECTURE DECISION RECORDS (ADRs)

An ADR is a short document capturing one architectural decision: the context, the decision, and the consequences. Immutable once accepted. Stored in the repository next to the code it governs.

**Why ADRs exist:** The #1 cause of unmaintainable systems is lost context. Engineers leave. Tickets get closed. Confluence pages go stale. ADRs stay in the repo, versioned alongside the code. When someone asks "why do we use X?" in three years, the answer is one `git log` away.

**The canonical format (Michael Nygard, 2011):**

```markdown
# ADR-0012: Use PostgreSQL as the primary datastore

## Status
Accepted

## Context
We need a primary relational datastore for the order management service.
Key requirements: ACID transactions (inventory reservation must be atomic),
complex queries with JOINs, existing team expertise, managed hosting on AWS.

Three options evaluated: PostgreSQL, MySQL, DynamoDB.

## Decision
We will use PostgreSQL 16 hosted on Amazon RDS.

Reasons:
- Serializable isolation eliminates inventory double-booking without application-level locking
- PostGIS extension available for future location-based features
- Team has 5+ years of PostgreSQL experience; MySQL expertise is limited to 2 engineers
- RDS provides automated backups, read replica setup, and point-in-time recovery

## Consequences
Positive:
- Strong consistency for financial operations without distributed transaction complexity
- Rich query language enables complex reporting without a separate analytics DB

Negative:
- Horizontal write scaling requires sharding (deferred until write throughput proves necessary)
- RDS costs more than self-managed PostgreSQL; accepted as operational simplicity trade-off

Neutral:
- Locks us into PostgreSQL-specific features (jsonb, full-text search). Switching costs would be high.
```

**ADR rules:**
- Store at `docs/adr/` in the repo. Number sequentially: `0001-use-postgres.md`.
- Never delete an ADR. Supersede it with a new one.
- Gate significant architectural changes through a lightweight ADR PR review.
- Backfill the 5–10 most consequential existing decisions when introducing ADRs to a project.

---

## 3. TECHNICAL DEBT

### The Quadrant (Fowler/Cunningham)

| | Deliberate | Inadvertent |
|--|-----------|-------------|
| **Reckless** | "We don't have time for design" | "What's layering?" |
| **Prudent** | "We must ship now; we'll fix it in Q2" | "Now we know how we should have done it" |

Only **Prudent-Deliberate** debt is acceptable: a conscious trade-off with a documented repayment plan. All other quadrants are problems to fix, not manage.

### Hotspot Analysis — the Highest-Signal Technique

Combine version control churn (files changed frequently) with complexity (cyclomatic complexity). Files in the top 10% of both are your highest-priority debt.

```bash
# Find high-churn files (changed most often in last 6 months)
git log --since="6 months ago" --name-only --format="" | \
  grep "\.py$" | sort | uniq -c | sort -rn | head -20
```

```python
# Calculate cyclomatic complexity with radon
# pip install radon
# radon cc src/ -s -n C  (show functions with complexity grade C or worse)
```

High churn + high complexity = the most dangerous technical debt. Every future change to these files is more expensive and more likely to introduce bugs.

### Debt Management Strategy

1. **Make it visible**: automated static analysis (SonarQube, Ruff, pylint) in CI, not periodic audits.
2. **Quantify in business terms**: translate complexity scores to estimated maintenance hours per quarter.
3. **Prioritize by risk**: debt in high-churn, critical-path code is 10× more dangerous than debt in stable, rarely-touched code.
4. **Allocate budget explicitly**: dedicate ~20% of sprint capacity to debt repayment. Without explicit allocation, it never happens — features always win.
5. **Track rhythmically**: monthly metric dashboard, quarterly architectural review.

---

## 4. DEPLOYMENT STRATEGIES

### Blue-Green
Two identical environments. Blue is live. Green has the new version. Switch the load balancer.

```
[Load Balancer]
      ↓ (switch)
Blue [old version] → Green [new version]
```

**Use when:** Complete version swap, instant rollback required, well-tested change, can afford 2× infra briefly. Best for: financial systems, compliance-critical apps.

**Trade-off:** All users affected at once if post-switch issues emerge. Database migrations must be backward-compatible (blue and green share the DB during transition).

### Canary
New version deployed to a small % of traffic. Gradually increase as metrics confirm stability.

```
[Load Balancer]
  95% → Old Version
   5% → New Version (canary)
          ↓ metrics OK
  50% → New Version
          ↓ metrics OK
 100% → New Version
```

**Use when:** Large user base, uncertain about new behavior, want real-world validation before full rollout.

**Trade-off:** Higher routing complexity. Requires per-version metrics to be meaningful. Slower to reach 100%.

### Feature Flags
Code is deployed to all users but new behavior is hidden behind a runtime toggle.

```python
# pip install unleash-client or use LaunchDarkly
from unleash import UnleashClient

client = UnleashClient(url="https://unleash.myapp.com", app_name="order-service")
client.initialize_client()

def create_order(request, payload):
    if client.is_enabled("new-payment-flow", {"userId": str(request.user.id)}):
        return new_payment_flow(payload)
    return legacy_payment_flow(payload)
```

**Use when:** Decouple deploy from release, target specific user cohorts, A/B testing, kill switch for risky features, trunk-based development.

**Trade-off:** Flag debt accumulates — must be cleaned up after rollout or they become hidden complexity. Each flag path must be tested.

---

## 5. FITNESS FUNCTIONS

An architectural fitness function provides an automated integrity check for an architectural characteristic. It is the mechanism for protecting what matters as a system evolves.

```
"Our modules must not have circular dependencies"
→ Enforced by import-linter in CI

"Response time must stay under 200ms at p99"
→ Enforced by k6 load test in CI/CD pipeline

"No direct ORM calls from the API layer"
→ Enforced by architecture tests (custom pytest)

"Code coverage must not decrease"
→ Enforced by coverage gate in CI
```

A build that breaks a fitness function is treated like a failing test — the change cannot be promoted. This is automated architectural governance.

```python
# Fitness function: no module may import from more than 3 other modules
# (enforced in CI via pytest)
import ast
import os

def test_module_coupling():
    max_imports = 3
    for filepath in glob("src/modules/**/*.py"):
        tree = ast.parse(open(filepath).read())
        imports = [n for n in ast.walk(tree) if isinstance(n, (ast.Import, ast.ImportFrom))]
        module_imports = {i.names[0].name.split(".")[2]
                          for i in imports
                          if hasattr(i, "module") and "modules" in (i.module or "")}
        assert len(module_imports) <= max_imports, \
            f"{filepath} imports from {len(module_imports)} modules (max {max_imports})"
```

---

## MAINTAINABILITY REVIEW CHECKLIST

**Observability**
- [ ] All services emit structured JSON logs with correlation IDs
- [ ] Metrics: request rate, error rate, p99 latency exported to Prometheus
- [ ] Distributed tracing instrumented (OpenTelemetry)
- [ ] Dashboards exist for all production services

**Deployment safety**
- [ ] Every deployment can be rolled back in < 5 minutes
- [ ] New features behind feature flags where appropriate
- [ ] Database migrations backward-compatible (old code runs against new schema)
- [ ] Canary or blue-green for high-risk deployments

**Architecture governance**
- [ ] ADRs written for all irreversible decisions
- [ ] Fitness functions enforced in CI (import boundaries, performance, coverage)
- [ ] DORA metrics tracked (deployment frequency, lead time, failure rate, recovery time)

**Technical debt**
- [ ] Static analysis runs in CI (SonarQube or equivalent)
- [ ] Hotspot analysis run quarterly (churn × complexity)
- [ ] ~20% of sprint capacity explicitly allocated to debt repayment
- [ ] No TODO/FIXME comments older than 6 months without a ticket

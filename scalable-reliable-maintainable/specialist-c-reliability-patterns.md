# Specialist C — Reliability Patterns

## THE FUNDAMENTAL TRUTH

Every dependency will fail. Every network call will time out. Every disk will fill. The question is not "will it fail?" but "how does your system behave when it does?"

Reliable systems fail gracefully: they degrade, not collapse. A single failing dependency must not take down the entire system.

---

## THE RELIABILITY PATTERN STACK

Apply these in combination. Each one addresses a different failure mode.

```
Timeout          → prevents indefinite blocking on slow calls
Retry            → recovers from transient failures automatically
Circuit Breaker  → stops hammering a failing dependency
Bulkhead         → limits blast radius when one dependency is slow
Fallback         → serves degraded response when primary path fails
```

---

## 1. TIMEOUT

**Every outbound call must have a finite timeout.** Without a timeout, one slow upstream holds a thread forever. With N simultaneous requests, you run out of threads and the entire service becomes unavailable.

```python
import httpx

# WRONG: no timeout — one slow upstream = thread starvation
response = httpx.get("https://api.external.com/data")

# RIGHT: timeout aligned to your SLO budget
response = httpx.get(
    "https://api.external.com/data",
    timeout=httpx.Timeout(connect=2.0, read=5.0, write=2.0, pool=1.0),
)
```

**Set timeouts based on the SLO budget:** If your p99 latency SLO is 500ms, your timeout for a non-critical downstream call should be well under that. For critical downstream calls, either meet the budget or rethink the synchronous dependency.

**Cascading timeouts:** In a chain A → B → C, B's timeout must be shorter than A's. If A gives up in 2s, B must give up in 1s, leaving room for error handling.

---

## 2. RETRY WITH EXPONENTIAL BACKOFF AND JITTER

Retry transient failures. Never retry permanent failures.

```python
import random
import time

class TransientError(Exception): pass
class PermanentError(Exception): pass

def retry(
    fn,
    max_attempts: int = 5,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
):
    """
    Retries fn on TransientError with exponential backoff and full jitter.
    Raises immediately on PermanentError.
    """
    for attempt in range(max_attempts):
        try:
            return fn()
        except PermanentError:
            raise  # never retry
        except TransientError as e:
            if attempt == max_attempts - 1:
                raise
            cap = min(base_delay * (2 ** attempt), max_delay)
            sleep = random.uniform(0, cap)           # full jitter
            time.sleep(sleep)
```

**Why jitter?** Without jitter, all retrying clients send requests simultaneously after the same backoff delay — a "retry storm" that hammers the recovering service. Jitter spreads retries over the backoff window.

**Classify before retrying:**

| Error | Type | Action |
|-------|------|--------|
| Network timeout | Transient | Retry with backoff |
| HTTP 429 (rate limited) | Transient | Retry after Retry-After header |
| HTTP 503 (service unavailable) | Transient | Retry with backoff |
| HTTP 400 (bad request) | Permanent | Do not retry |
| HTTP 404 (not found) | Permanent | Do not retry |
| HTTP 422 (validation) | Permanent | Do not retry |
| DB unique constraint violation | Permanent | Do not retry |

---

## 3. CIRCUIT BREAKER

Detects a failing dependency and stops sending requests to it until it recovers. Prevents: thread starvation from queued timeouts, cascading failure, resource exhaustion.

```
Closed (normal)
   ↓ failure_count > threshold
Open (failing fast)
   ↓ after reset_timeout
Half-Open (probing)
   ↓ probe succeeds → Closed
   ↓ probe fails → Open again
```

```python
# pip install pybreaker
import pybreaker

db_breaker = pybreaker.CircuitBreaker(
    fail_max=5,         # open after 5 consecutive failures
    reset_timeout=30,   # probe after 30s
)

@db_breaker
def query_analytics_db(query: str) -> list:
    return analytics_db.execute(query)

def get_analytics(query: str) -> list | None:
    try:
        return query_analytics_db(query)
    except pybreaker.CircuitBreakerError:
        # Circuit is open — return fallback, don't wait
        return None  # caller uses default/cached value
```

**Circuit breaker for HTTP clients:**

```python
from httpx import Client

client = Client(base_url="https://api.partner.com")
breaker = pybreaker.CircuitBreaker(fail_max=10, reset_timeout=60)

@breaker
def call_partner_api(endpoint: str) -> dict:
    response = client.get(endpoint, timeout=5.0)
    if response.status_code >= 500:
        raise TransientError(f"HTTP {response.status_code}")
    return response.json()
```

---

## 4. BULKHEAD

Isolates resources (thread pools, connection pools) so a failing dependency cannot exhaust resources needed by other dependencies. Inspired by ship hull compartments.

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

# Separate thread pools per dependency
analytics_executor = ThreadPoolExecutor(max_workers=5, thread_name_prefix="analytics")
payments_executor = ThreadPoolExecutor(max_workers=10, thread_name_prefix="payments")
notifications_executor = ThreadPoolExecutor(max_workers=3, thread_name_prefix="notify")

# Slow analytics service can only exhaust 5 threads — payments still has 10
async def handle_request(order_id: str) -> dict:
    loop = asyncio.get_event_loop()

    # These run in separate, isolated pools
    payment_task = loop.run_in_executor(payments_executor, process_payment, order_id)
    analytics_task = loop.run_in_executor(analytics_executor, log_analytics, order_id)

    payment_result = await payment_task
    asyncio.ensure_future(analytics_task)  # non-critical, don't await
    return payment_result
```

**Bulkhead + circuit breaker:** Bulkhead limits the blast radius. Circuit breaker limits the duration. Use both together.

---

## 5. FALLBACK

When the primary path fails (after circuit break, timeout, or retry exhaustion), return a degraded but valid response.

```python
def get_product_recommendations(user_id: str) -> list[dict]:
    try:
        # Primary: personalized recommendations from ML service
        return recommendation_service.get(user_id, timeout=2.0)
    except (TransientError, pybreaker.CircuitBreakerError):
        # Fallback: popular items (always available, no ML needed)
        return get_popular_items(limit=10)
    except PermanentError:
        # Fallback: empty list with a flag
        return []

def get_user_preferences(user_id: str) -> dict:
    try:
        return preferences_service.get(user_id)
    except Exception:
        # Fallback: last known good value from cache
        cached = redis.get(f"prefs:{user_id}")
        return json.loads(cached) if cached else DEFAULT_PREFERENCES
```

**When NOT to use fallback:** When partial/stale data causes more harm than an honest error. Payment calculations, medical dosing, inventory reservation — these must fail loudly, not silently degrade.

---

## 6. HIGH AVAILABILITY TOPOLOGIES

### Active-Passive
One node handles all traffic. One or more hot standbys monitor and wait. On primary failure, a standby is promoted (seconds of downtime).

```
Client → Load Balancer → Primary [Active]
                       → Standby [Passive — monitoring only]
```

**Use for:** Databases where single-primary guarantees consistency (PostgreSQL, MySQL). The standby accepts no traffic and cannot diverge.

### Active-Active
All nodes handle traffic. On node failure, remaining nodes absorb the load with zero downtime (they were already serving).

```
Client → Load Balancer → Node 1 [Active]
                       → Node 2 [Active]
                       → Node 3 [Active]
```

**Use for:** Stateless compute (web servers, API servers — trivially active-active), databases that natively support multi-primary with conflict resolution (Cassandra, DynamoDB global tables).

**Never do active-active for a primary relational database without a database that explicitly supports it.** Two PostgreSQL primaries writing independently = data corruption.

### Quorum Writes (for distributed state)

In a cluster of N nodes, a write is confirmed only after `N/2 + 1` nodes acknowledge it (a majority quorum). This prevents split-brain: two separate network partitions cannot both confirm a write to the same key.

```
5-node cluster: quorum = 3
Write to node 1 → propagates to nodes 2, 3 (quorum met) → confirmed
Even if nodes 4, 5 are unreachable, only one partition has quorum
```

---

## 7. GRACEFUL DEGRADATION

Convert hard dependencies (required) into soft dependencies (preferred) wherever the user experience allows it.

```python
class OrderService:
    def create_order(self, cart: Cart) -> Order:
        order = self._create_core_order(cart)         # hard dependency — must succeed

        # Soft dependencies — degrade if unavailable
        try:
            self._apply_loyalty_points(order)         # nice to have
        except Exception:
            logger.warning("loyalty_service_unavailable", order_id=str(order.id))
            # Order still completes without loyalty points

        try:
            self._send_confirmation_email(order)      # async anyway
        except Exception:
            logger.warning("email_service_unavailable", order_id=str(order.id))
            # Queue for retry; don't fail the order

        return order
```

**The matrix:**

| Dependency | Impact if down | Dependency type | Action on failure |
|------------|---------------|-----------------|-------------------|
| Payment gateway | Order cannot complete | Hard | Fail loudly |
| Inventory service | Oversell risk | Hard | Fail loudly |
| Recommendation engine | Show popular items instead | Soft | Fallback |
| Email service | Email delayed, not lost | Soft | Queue for retry |
| Analytics tracking | Metrics gap | Soft | Log and skip |

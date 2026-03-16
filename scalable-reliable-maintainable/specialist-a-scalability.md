# Specialist A — Scalability Patterns

## THE ONE RULE

**Profile first. Scale the bottleneck. Never distribute by instinct.**

The cost of premature distribution: operational complexity, debugging surface area, network failure modes, and infrastructure cost — all multiplied — for zero benefit if the real bottleneck is a slow query or a missing cache.

---

## DECISION TREE

```
Need more capacity?
│
├─ Is it a single slow operation (query, transform)?
│   → Fix the operation first. Index the query. Optimize the code.
│
├─ Is the bottleneck I/O-bound (waiting on network, disk)?
│   → Async I/O (asyncio, non-blocking). No new machines needed.
│
├─ Is it read-heavy (same data read many times)?
│   → Cache layer (Redis). Add read replicas for DB reads.
│
├─ Is it CPU-bound and parallelizable?
│   → Horizontal scaling (add stateless instances behind a load balancer).
│
├─ Is traffic spiky (bursts, background jobs)?
│   → Queue-based load leveling. Decouple producer from consumer.
│
├─ Is it a single machine hitting resource limits?
│   → Vertical scaling (more RAM/CPU) as short-term. Plan horizontal.
│
└─ Is data volume exceeding single-node DB limits?
    → See specialist-b-database-scaling.md
```

---

## 1. HORIZONTAL VS VERTICAL SCALING

### Vertical Scaling (scale up)
Add more CPU, RAM, or storage to an existing machine. Simple — no code changes. Limited by the largest available machine. Cannot survive hardware failure.

**Use when:** Legacy monolith that cannot be horizontally distributed, short-term pressure relief, database primary node (databases are stateful — horizontal scaling requires deliberate design).

### Horizontal Scaling (scale out)
Add more machines. Distribute traffic across them. Requires stateless application servers. Scales theoretically without limit.

**Use when:** Web/API servers, stateless microservices, any embarrassingly parallel workload.

### The Prerequisite: Statelessness

Horizontal scaling only works if instances are interchangeable. An instance is interchangeable only if it holds no local state.

```python
# WRONG: state in process memory — two requests to different instances give inconsistent results
_in_memory_session = {}

def set_session(user_id, data):
    _in_memory_session[user_id] = data  # dies on redeploy, broken with 2+ instances

# RIGHT: state in the data layer
import redis
r = redis.Redis()

def set_session(user_id: str, data: dict, ttl: int = 3600) -> None:
    r.setex(f"session:{user_id}", ttl, json.dumps(data))

def get_session(user_id: str) -> dict | None:
    raw = r.get(f"session:{user_id}")
    return json.loads(raw) if raw else None
```

**Everything stateful lives in the data layer:** sessions (Redis), distributed locks (Redis + Redlock), file uploads (S3/object storage), job queues (Redis/RabbitMQ/Kafka).

---

## 2. CACHING LAYERS

### The Multi-Layer Model

```
Browser/Client Cache            ← static assets, API responses with Cache-Control
        ↓
CDN Edge Cache                  ← static assets, public GET responses
        ↓
Reverse Proxy / API Gateway     ← repeated API responses, rate limiting
        ↓
Application Cache (Redis)       ← hot query results, sessions, rate counters
        ↓
Database                        ← source of truth
```

### What Goes Where

| Layer | Contents | TTL | Tool |
|-------|----------|-----|------|
| CDN | Static files (JS, CSS, images), public pages | Hours–Days | Cloudflare, CloudFront |
| Redis (application) | Session tokens, computed aggregates, hot records, rate limit counters | Seconds–Hours | Redis 7.x |
| DB query result | Expensive join results, rarely-changing reference data | Minutes | Redis or ORM-level |

Redis reads are 10–100× faster than a disk-backed database for the same data.

### Cache Patterns

```python
# Cache-Aside (Lazy Loading) — general purpose starting point
def get_product(product_id: str) -> dict:
    cached = redis.get(f"product:{product_id}")
    if cached:
        return json.loads(cached)                        # cache hit

    product = db.query("SELECT * FROM products WHERE id = %s", product_id)
    redis.setex(f"product:{product_id}", 300, json.dumps(product))  # populate
    return product

# Write-Through — keeps cache consistent on every write
def update_product(product_id: str, data: dict) -> None:
    db.execute("UPDATE products SET ... WHERE id = %s", product_id)
    redis.setex(f"product:{product_id}", 300, json.dumps(data))     # update cache
    redis.delete("products:list")                                    # invalidate list

# Write-Behind — high-write throughput, tolerate brief inconsistency
def record_view(product_id: str) -> None:
    redis.incr(f"views:{product_id}")    # instant in-memory counter
    # background job flushes to DB every 60s
```

### Cache Invalidation Rules

- Invalidate the specific key on write, don't wait for TTL expiry.
- Set TTL as a fallback safety net, not the primary invalidation mechanism.
- When invalidating list caches, use a cache key that includes a version/tag that you can bump on writes.
- Never cache mutable financial data, inventory counts, or session-critical state with long TTLs.

---

## 3. LOAD BALANCING

| Algorithm | How it works | Use when |
|-----------|-------------|----------|
| **Round Robin** | Requests distributed in sequence | Homogeneous servers, stateless HTTP |
| **Weighted Round Robin** | Round robin with server capacity weights | Mixed instance sizes |
| **Least Connections** | Route to server with fewest active connections | Variable request duration, WebSockets, DB connections |
| **Consistent Hashing** | Hash key maps to a point on a ring; nearest server serves it | Distributed caches (same key → same server), session affinity |
| **IP Hash** | Route by client IP | Legacy stateful apps (last resort only) |

### Consistent Hashing — Why It Matters

With simple modulo hashing (`hash(key) % N`), adding or removing one server remaps all keys — every cache misses simultaneously (thundering herd). Consistent hashing remaps only `K/N` keys when a server is added/removed (`K` = total keys, `N` = server count).

```python
import hashlib
from bisect import bisect_right, insort

class ConsistentHashRing:
    def __init__(self, virtual_nodes: int = 150):
        self.ring: list[int] = []
        self.nodes: dict[int, str] = {}
        self.virtual_nodes = virtual_nodes

    def add_server(self, server: str) -> None:
        for i in range(self.virtual_nodes):
            key = self._hash(f"{server}:{i}")
            insort(self.ring, key)
            self.nodes[key] = server

    def get_server(self, key: str) -> str:
        h = self._hash(key)
        idx = bisect_right(self.ring, h) % len(self.ring)
        return self.nodes[self.ring[idx]]

    def _hash(self, value: str) -> int:
        return int(hashlib.md5(value.encode()).hexdigest(), 16)
```

---

## 4. QUEUE-BASED LOAD LEVELING

Absorb traffic spikes by decoupling producers from consumers through a queue. Producers enqueue work and return immediately. Consumers drain the queue at their own pace. Add consumer instances to increase throughput linearly.

```
Web Request → [Queue] → Worker Pool
                ↑
          Buffers spikes.
          Workers scale independently.
          Worker crash = task stays in queue.
```

```python
# Celery + Redis: classic app-level load leveling
from celery import Celery

app = Celery(broker="redis://localhost:6379/0")

@app.task(
    bind=True,
    max_retries=5,
    autoretry_for=(TransientError,),
    retry_backoff=True,
    retry_jitter=True,
)
def process_order(self, order_id: str) -> None:
    order = fetch_order(order_id)
    run_fulfillment_logic(order)

# Scale consumers without touching the producer:
# celery -A myapp worker -c 20  (20 concurrent workers)
```

**Tool selection:**

| Tool | Use case |
|------|----------|
| Celery + Redis | App-level task offloading (Django, FastAPI) |
| Kafka | High-throughput event streaming, replay, audit |
| RabbitMQ | Complex routing, reliable delivery |
| AWS SQS | Managed, serverless-friendly |

---

## 5. AUTO-SCALING

Reactive auto-scaling (scale when CPU > 70%) is the baseline. Predictive auto-scaling (scale before known traffic peaks based on historical patterns) reduces cold-start latency and capacity incidents.

**Rules:**
- Scale out aggressively on rising load. Scale in conservatively on falling load (avoid flapping).
- Health checks must be fast and reliable — failing health checks trigger premature replacement.
- Warm-up time matters: a new instance that isn't ready yet must not receive traffic. Use readiness probes (Kubernetes) or ELB target group health checks.

---

## SCALING DECISION TABLE

| Data size | Latency | Team size | → Architecture |
|-----------|---------|-----------|---------------|
| < 10 GB | Minutes OK | 1–5 | Single well-optimized machine |
| 10–100 GB | Minutes OK | 5–15 | Horizontal + Redis cache + read replicas |
| > 100 GB structured | Minutes OK | 15+ | Sharded DB or PySpark/Dask |
| Any size | < 1s always | Any | Queue + async workers + cache |
| Real-time stream | < 100ms | Any | Kafka + stream processor |

# Specialist B — Database Scaling

## THE ONE RULE

**Do not shard until you have exhausted every other option.** Sharding introduces cross-shard query complexity, distributed transaction nightmares, and uneven data distribution bugs. Most systems never legitimately need it.

Exhaust in this order: optimize queries → add indexes → add read replicas → vertical scale → partition within a single node → shard.

---

## SCALING LADDER

```
Step 1: Optimize the query and schema
        └─ Indexes, query plans, N+1 elimination, denormalization

Step 2: Connection pooling (PgBouncer, ProxySQL)
        └─ Reduces connection overhead. Often 2–5× throughput improvement for free.

Step 3: Read replicas
        └─ Route read-heavy queries to replicas. Frees primary for writes.

Step 4: Vertical scaling
        └─ More RAM, faster SSDs. High per-unit cost but buys time.

Step 5: Table partitioning (within one instance)
        └─ Splits large tables by range/hash/list. Reduces index size. Faster scans.

Step 6: Caching layer (Redis)
        └─ Hot reads never reach the DB.

Step 7: Sharding (multiple instances)
        └─ Last resort. Only when a single node's write throughput is the provable limit.
```

---

## 1. READ REPLICAS

The primary node handles all writes. Replicas stream changes from the primary and serve read queries.

```
Primary ──writes──> WAL/binlog ──replication──> Replica 1
                                              └──> Replica 2
                                              └──> Replica 3 (analytics)
```

**Route reads to replicas:**

```python
# Django: multiple DB routing
# settings.py
DATABASES = {
    "default": {"HOST": "primary.db", ...},
    "replica": {"HOST": "replica.db", ...},
}

# router.py
class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):
        return "replica"  # all reads go to replica

    def db_for_write(self, model, **hints):
        return "default"  # all writes go to primary

    def allow_relation(self, obj1, obj2, **hints):
        return True

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == "default"
```

**Replication lag warning:** Replicas are slightly behind the primary. Never read from a replica immediately after a write if you need read-your-own-writes consistency. Options:
- Use the primary for reads within the same request that performed a write.
- Use synchronous replication for critical data (financial records).
- Use sticky sessions to route a user to the same replica for a session window.

---

## 2. TABLE PARTITIONING

Split one large table into smaller partitions, all within the same database instance. Improves query performance by reducing the scan range and allows older partitions to be dropped quickly.

```sql
-- PostgreSQL range partitioning by date
CREATE TABLE orders (
    id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    amount NUMERIC NOT NULL
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Queries with WHERE created_at = '2024-01-15' only scan one partition
-- DROP TABLE orders_2024_01; -- instant archival of old data
```

**Partition strategies:**
- **Range** (dates, sequential IDs): best for time-series, log data, archival.
- **Hash** (user_id, order_id): best for even distribution across partitions.
- **List** (region, category): best for known, finite value sets.

---

## 3. SHARDING

Sharding distributes data across multiple separate database instances (shards). Each shard is a full database server holding a subset of the rows.

**Only shard when:** Single-node write throughput is the demonstrated bottleneck and vertical scaling is cost-prohibitive.

### Shard Key Selection (most critical decision)

```python
# The shard key determines data distribution and query routing.
# A bad shard key causes hot shards (some shards overloaded, others idle).

# GOOD shard keys: high cardinality, evenly distributed, matches most query patterns
shard_key = user_id    # if most queries are per-user
shard_key = tenant_id  # for multi-tenant SaaS

# BAD shard key: low cardinality or time-correlated (causes hot shards)
shard_key = country        # most traffic from US → US shard overloaded
shard_key = created_at     # all new writes go to the newest shard

# Shard routing
def get_shard(key: str, num_shards: int) -> int:
    return int(hashlib.md5(key.encode()).hexdigest(), 16) % num_shards

SHARD_CONNECTIONS = {
    0: connect("shard-0.db"),
    1: connect("shard-1.db"),
    2: connect("shard-2.db"),
}

def get_db(user_id: str):
    return SHARD_CONNECTIONS[get_shard(user_id, len(SHARD_CONNECTIONS))]
```

### Problems Sharding Creates

| Problem | Impact | Mitigation |
|---------|--------|-----------|
| Cross-shard queries | Scatter-gather: query all shards, merge results | Denormalize, or accept higher query cost |
| Cross-shard transactions | No atomic transactions across shards | Saga pattern or avoid cross-shard writes |
| Hot shards | Uneven load | Better shard key; consistent hashing with virtual nodes |
| Rebalancing | Adding a shard requires data migration | Consistent hashing reduces remapping |
| Schema migrations | Must run on every shard | Orchestrate carefully |

---

## 4. CONSISTENCY MODEL SELECTION

Choose the consistency model before choosing the database. The model determines which databases are valid options.

| Need | Consistency model | Database options |
|------|-----------------|-----------------|
| Financial transactions, inventory, locks | **Serializable / Linearizable** | PostgreSQL (serializable isolation), CockroachDB, Spanner |
| User profiles, product catalog, feeds | **Eventual consistency** | DynamoDB, Cassandra, MongoDB |
| Messaging with reply ordering | **Causal consistency** | CockroachDB (snapshot isolation), Cassandra (lightweight transactions) |
| Read-heavy, staleness OK | **Read-your-writes** | PostgreSQL + read replicas (route by session) |

**The practical question:** "What is the cost of a stale read vs. the cost of a write failure?"
- Low stale read cost (social feed, product view count) → eventual consistency.
- High stale read cost (bank balance, inventory, booking) → strong consistency.

---

## 5. DATABASE-SPECIFIC SCALING PATTERNS

### PostgreSQL
- PgBouncer for connection pooling (reduces connection overhead dramatically).
- `pg_partman` for automated partition management.
- Read replicas via streaming replication.
- `EXPLAIN ANALYZE` to catch bad query plans before they hit production.
- Logical replication for zero-downtime migrations.

### Redis
- Redis Cluster for horizontal partitioning of key space.
- Sentinel for high availability (automatic failover).
- Use Redis for: sessions, rate limiting, pub/sub, leaderboards, distributed locks.
- Never use Redis as your primary database for data you cannot afford to lose.

### Cassandra / DynamoDB
- Both are AP systems (eventual consistency, partition-tolerant).
- Design tables around access patterns, not normalization.
- Partition key = the field you always query by. Clustering key = sort order within a partition.
- No JOINs. Denormalize aggressively. One table per query pattern.

---

## DECISION SUMMARY

```
Is the slow query fixable with an index or rewrite?
  → Fix the query. Don't scale yet.

Is it read-heavy (> 5:1 read/write ratio)?
  → Add read replicas. Route reads there.

Is the table scan slow on a large time-series table?
  → Add range partitioning by date.

Is the DB primary write throughput the bottleneck (provable)?
  → Shard. Choose shard key carefully. Accept cross-shard query complexity.

Is the data naturally non-relational (document, key-value)?
  → Use the appropriate purpose-built database. Don't fight the relational model.
```

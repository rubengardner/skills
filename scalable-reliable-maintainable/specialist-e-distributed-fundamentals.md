# Specialist E — Distributed Systems Fundamentals

## THE HONEST FRAMING

Most engineers invoke CAP theorem incorrectly. Most "eventual consistency" decisions are made without thinking through what "eventual" means in their specific failure scenario. This specialist cuts through the theory to the practical decision rules.

Primary sources: Martin Kleppmann's *Designing Data-Intensive Applications*, Kyle Kingsbury's Jepsen analyses (jepsen.io), the original papers.

---

## 1. CAP THEOREM — WHAT IT ACTUALLY MEANS

**The theorem:** A distributed system can guarantee at most two of:
- **C**onsistency — all nodes see the same data at the same time (linearizability)
- **A**vailability — every non-failing node responds to every request
- **P**artition Tolerance — the system continues operating when the network splits

**The key insight:** Partition tolerance is not optional. Networks fail. If you have two nodes, they can become unreachable to each other. CAP reduces to a single question: **when a partition occurs, do you stay consistent or stay available?**

- **CP systems** (HBase, etcd, ZooKeeper): during a partition, refuse writes to stay consistent. "I'd rather return an error than risk diverging data."
- **AP systems** (Cassandra, DynamoDB, CouchDB): during a partition, continue serving reads/writes. "I'd rather serve stale data than return an error."

**The practical question is not "CP or AP?"** It is: "What is the cost of serving stale data vs. the cost of refusing a request?"

| Scenario | Stale data cost | → Choice |
|----------|----------------|---------|
| Bank account balance | Very high (overdraft) | CP |
| Inventory reservation | High (oversell) | CP |
| Social media like count | Low (off by a few) | AP |
| User profile photo | Low | AP |
| Distributed lock | Very high (double action) | CP |
| Product description | Low | AP |

---

## 2. PACELC — THE MORE USEFUL MODEL

PACELC (Daniel Abadi, 2012) extends CAP to the normal case (no partition):

```
If Partition:       choose Availability or Consistency
Else (normal):      choose Latency or Consistency
```

This is more honest because most of the time there is no partition, yet you are still making a latency/consistency trade-off on every single request.

| System | Partition choice | Normal choice | Practical implication |
|--------|-----------------|--------------|----------------------|
| Cassandra | Availability | Latency | Eventual consistency always; very low latency |
| DynamoDB | Availability | Latency | Eventual by default; strong consistency costs extra |
| HBase | Consistency | Consistency | Strong consistency; higher write latency |
| etcd / ZooKeeper | Consistency | Consistency | Linearizable; used for distributed locks, leader election |
| PostgreSQL | Consistency | Consistency | ACID; the right default for relational data |
| Google Spanner | Consistency | Consistency | Linearizable globally via TrueTime |

---

## 3. CONSISTENCY MODELS (from strongest to weakest)

```
Linearizability (Strict Serializability)
  └─ Every operation appears instantaneous. Real-time ordering globally.
  └─ Most expensive. Used for: distributed locks, leader election, unique IDs.
  └─ Tools: etcd, ZooKeeper, Google Spanner

Serializability
  └─ Transactions execute as if serial (but not necessarily in real-time order).
  └─ The I in ACID. Used for: financial transactions, inventory.
  └─ Tools: PostgreSQL (serializable isolation), CockroachDB

Sequential Consistency
  └─ All nodes see operations in the same order, but not necessarily real-time.

Causal Consistency
  └─ Causally related operations (A happened before B) seen in the same order by all nodes.
  └─ Unrelated operations can be seen in any order.
  └─ Used for: comments and replies, collaborative document editing.
  └─ Tools: MongoDB (sessions), CockroachDB (snapshot isolation)

Eventual Consistency
  └─ All replicas will converge if writes stop.
  └─ No guarantee on when. No guarantee on intermediate states.
  └─ Used for: shopping carts, like counts, social feeds, DNS.
  └─ Tools: Cassandra, DynamoDB (default), CouchDB
```

**A critical warning from Jepsen (jepsen.io):** Many databases claim consistency levels they do not actually provide. MongoDB claimed to be linearizable and was not (for years). Cassandra's "quorum" reads can still return stale data under specific failure conditions. Test your database with Jepsen-style fault injection before relying on its consistency guarantees in a critical system.

---

## 4. CONSENSUS: RAFT

Consensus is the problem of getting a cluster of nodes to agree on a single value, even when some nodes fail. It is required for: leader election, distributed locks, replicated state machines.

**Paxos** (Lamport, 1989) was the first practical algorithm. It is notoriously hard to understand and implement correctly.

**Raft** (Ongaro & Ousterhout, 2014) was designed specifically to be understandable. It is now the dominant consensus algorithm in production systems (etcd, CockroachDB, TiKV, Consul).

### How Raft Works (simplified)

```
Roles:
  Leader — handles all writes, replicates to followers
  Follower — accepts replicated entries from leader, votes in elections
  Candidate — a follower that has timed out and is requesting votes

Normal operation:
  1. Client sends write to Leader
  2. Leader appends entry to its log
  3. Leader replicates entry to Followers (AppendEntries RPC)
  4. Once a MAJORITY (quorum) acknowledges, Leader commits
  5. Leader responds to client

Leader election:
  1. Follower times out (no heartbeat from Leader)
  2. Follower becomes Candidate, increments term, votes for itself
  3. Sends RequestVote RPC to all peers
  4. If majority votes granted → becomes Leader
  5. Immediately sends heartbeats to prevent new elections
```

**The quorum rule:** A cluster of N nodes can tolerate `(N-1)/2` node failures. A 3-node cluster tolerates 1 failure. A 5-node cluster tolerates 2 failures.

```
Cluster size | Quorum | Tolerated failures
3            | 2      | 1
5            | 3      | 2
7            | 4      | 3
```

Odd-numbered cluster sizes are standard (no tie-breaking problem in elections).

**Interactive visualization:** raft.github.io — the best way to build intuition for leader election and log replication.

---

## 5. THE OUTBOX PATTERN

The most common source of data loss in event-driven systems: a service writes to the database AND publishes to Kafka in two separate operations. One can succeed and the other can fail.

```python
# WRONG: dual-write — one can fail without the other
def create_order(order_data: dict) -> Order:
    order = db.insert(order_data)           # succeeds
    kafka.publish("order.created", order)   # crashes here → event lost
    return order
```

**The Outbox Pattern:** Write the event to an `outbox` table in the same database transaction as the business change. A separate process (the "outbox processor") reads the outbox and publishes to Kafka. No dual-write.

```python
# RIGHT: single atomic transaction
def create_order(order_data: dict) -> Order:
    with transaction.atomic():
        order = Order.objects.create(**order_data)
        OutboxEvent.objects.create(      # same transaction
            topic="order.created",
            key=str(order.id),
            payload=serialize(order),
            status="pending",
        )
    # Outbox processor (separate process) polls and publishes
    return order

# Outbox processor (runs independently)
def process_outbox() -> None:
    events = OutboxEvent.objects.filter(status="pending").order_by("created_at")[:100]
    for event in events:
        kafka.publish(event.topic, event.key, event.payload)
        event.status = "published"
        event.save()
```

This guarantees: if the transaction commits, the event will eventually be published. If the transaction rolls back, no event is published. No lost events.

---

## 6. MUST-READ PAPERS (prioritized)

For each paper: why it matters > what it says.

| Paper | Why It Matters |
|-------|---------------|
| **Dynamo** (Amazon, 2007) | Defined eventual consistency, consistent hashing, vector clocks, sloppy quorums. The foundation for DynamoDB and Cassandra. |
| **Raft** (Ongaro, 2014) | The consensus algorithm you will actually implement or use. Read this before touching etcd or CockroachDB internals. |
| **Kafka** (Kreps, 2011) | The unified log concept — why treating the message log as the source of truth changes everything. |
| **Spanner** (Google, 2012) | Shows what's possible when you synchronize physical clocks globally. TrueTime enables external consistency at planetary scale. |
| **GFS** (Google, 2003) | First systems paper that treats hardware failure as the norm, not the exception. Changes how you design for failure. |
| **Lamport Clocks** (1978) | Foundation of every logical clock, vector clock, and causal consistency system. Short paper, essential conceptually. |

**Reading lists:**
- dancres.github.io/Pages/ — curated distributed systems papers
- pdos.csail.mit.edu/6.824 — MIT 6.5840 course reading list
- jepsen.io — real consistency failure analyses of production databases

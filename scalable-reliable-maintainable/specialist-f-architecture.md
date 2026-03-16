# Specialist F — Architecture Patterns

---

## 1. MONOLITH vs MODULAR MONOLITH vs MICROSERVICES

### The Honest Comparison

| | Monolith | Modular Monolith | Microservices |
|--|----------|-----------------|---------------|
| Deployment | Single unit | Single unit | Per-service |
| Scaling | Whole app | Whole app | Per-service |
| Team autonomy | Low (shared codebase) | Medium (module ownership) | High (service ownership) |
| Operational cost | Very low | Low | High |
| Debugging | Easy (single process) | Easy | Hard (distributed traces required) |
| Latency (internal calls) | In-process (µs) | In-process (µs) | Network (ms) |
| Failure modes | Simple | Simple | Complex (partial failure, timeouts) |
| When to choose | MVP, small team | Growing team, monolith getting messy | Large org, proven bottleneck, team autonomy required |

### Real-World Evidence

- **Shopify**: 2.8 million lines of Ruby, modular monolith, handles Black Friday at 30 TB/minute. They componentized their monolith rather than microservicing it.
- **Stack Overflow**: Serves 6,000 requests/second, ~2 billion page views/month on a small number of servers with a modular monolith. 12ms median page render.
- **Amazon, Netflix, Uber**: Moved to microservices only after hitting genuine team coordination and scaling limits in large monoliths. They had hundreds of engineers and clear domain boundaries before they split.

### Decision Rule

> Start with a modular monolith. Draw clean domain boundaries inside it. Extract to microservices only when: (a) a specific scaling bottleneck cannot be solved by scaling the whole monolith, (b) team autonomy requirements mean release coupling is causing real pain, or (c) a fault isolation requirement means one domain failing must not take down others.

**The distributed monolith trap:** Microservices cut along technical lines (user service, order service) rather than domain lines (domain-driven design). The result: services that are tightly coupled through synchronous calls, shared databases, and coordinated deploys. All the complexity of distributed systems, none of the independence benefits.

### What a Modular Monolith Looks Like

```
myapp/
├── modules/
│   ├── orders/
│   │   ├── domain/          # domain models, interfaces
│   │   ├── application/     # use cases
│   │   ├── infrastructure/  # ORM, external calls
│   │   └── api/             # HTTP handlers
│   ├── payments/
│   │   └── ...
│   └── inventory/
│       └── ...
├── shared/
│   └── events/              # cross-module event contracts
└── config/
```

**Module boundaries are enforced in code:**
```python
# pip install import-linter
# .importlinter
[importlinter]
root_package = myapp

[importlinter:contract:orders-independence]
name = Orders cannot import from Payments
type = forbidden
source_modules = myapp.modules.orders
forbidden_modules = myapp.modules.payments
# Cross-module communication via domain events only, not direct imports
```

---

## 2. CQRS (Command Query Responsibility Segregation)

Separates the write model (commands that change state) from the read model (queries optimized for display). Read and write sides can evolve, scale, and be optimized independently.

```
Command side:  POST /orders → OrderCommandHandler → domain model → write DB (normalized)
Query side:    GET /orders  → OrderQueryHandler   → read model → read DB (denormalized)
```

### When CQRS Helps

- Read and write shapes are very different (rich domain model vs flat view DTO)
- Read traffic >> write traffic (scale read model independently)
- Multiple read projections of the same data are needed (dashboard view, mobile view, search index)
- Domain logic is complex (separating it from query concerns reduces cognitive load)

### When CQRS Hurts

- Simple CRUD applications — the extra plumbing costs more than it saves
- Small teams — indirection slows development
- Systems needing immediate read-after-write consistency — CQRS read models are eventually consistent

```python
# CQRS with Dagster or a simple projection updater
# Write side: rich domain command
class PlaceOrderCommand:
    customer_id: str
    items: list[OrderItem]
    coupon_code: str | None

class PlaceOrderHandler:
    def handle(self, cmd: PlaceOrderCommand) -> Order:
        order = Order.create(cmd.customer_id, cmd.items)
        if cmd.coupon_code:
            order.apply_coupon(self._coupon_repo.get(cmd.coupon_code))
        self._order_repo.save(order)
        self._event_bus.publish(OrderPlaced(order_id=order.id))
        return order

# Read side: flat projection optimized for the order list screen
class OrderListProjection:
    def on_order_placed(self, event: OrderPlaced) -> None:
        # Denormalized: everything the list screen needs in one row
        OrderReadModel.objects.create(
            order_id=event.order_id,
            customer_name=...,   # denormalized from customer
            item_count=...,
            total=...,
            status="pending",
        )
```

---

## 3. EVENT SOURCING

Store the full sequence of state-change events rather than current state. The current state is derived by replaying events.

```
Traditional:  orders table has one row per order (current state only)
Event Sourced: order_events table has every event (OrderPlaced, ItemAdded, OrderShipped...)
               current state = replay all events for order_id
```

### When Event Sourcing Helps

- Full audit trail required by compliance (finance, healthcare, insurance)
- Temporal queries: "what was the account balance on March 1st?"
- Complex business rules driven by event history
- Need to replay history or rebuild projections

### When Event Sourcing Hurts

- Simple CRUD domains — snapshots and projections add operational complexity for zero benefit
- Teams unfamiliar with the pattern — learning curve is steep
- Event schema evolution is permanently hard: old events must remain deserializable forever

### The Key Gotcha: Schema Evolution

```python
# Event schema from year 1
class OrderPlaced_v1(BaseModel):
    order_id: str
    customer_id: str
    amount: float

# Year 2: split amount into subtotal + tax
# You CANNOT change v1 events — they are historical facts
# You must version events and write a migration/upscast:

class OrderPlaced_v2(BaseModel):
    order_id: str
    customer_id: str
    subtotal: float
    tax: float
    amount: float  # kept for backward compat

def upcast_v1_to_v2(event: dict) -> dict:
    if event["version"] == 1:
        event["subtotal"] = event["amount"] * 0.9
        event["tax"] = event["amount"] * 0.1
        event["version"] = 2
    return event
```

**Microsoft Azure Architecture Center's honest summary:** "If your application does not require hyper-scale or performance, has a simple or small domain, or naturally works well with CRUD — event sourcing is not useful."

---

## 4. EVENT-DRIVEN ARCHITECTURE

Services communicate by publishing and consuming events rather than synchronous RPC.

```
Traditional (synchronous):
Order Service → HTTP → Inventory Service → HTTP → Notification Service
(tight coupling, failure cascades)

Event-driven (async):
Order Service → publishes OrderPlaced → Kafka
  Inventory Service (consumer) → reserves stock
  Notification Service (consumer) → sends confirmation
(loose coupling, services fail independently)
```

### Critical Pitfalls

**1. No idempotency:** Kafka delivers at-least-once. Consumers will receive the same event multiple times (during retries, rebalancing, re-processing). Every consumer must be idempotent.

```python
def handle_order_placed(event: OrderPlaced) -> None:
    if ProcessedEvent.objects.filter(event_id=event.id).exists():
        return  # idempotent skip — already processed

    with transaction.atomic():
        reserve_inventory(event.order_id, event.items)
        ProcessedEvent.objects.create(event_id=event.id)
```

**2. Dual-write:** Never write to the DB and publish to Kafka in separate operations. Use the outbox pattern (see specialist-e-distributed-fundamentals.md).

**3. Event schema as public API:** Once other services consume your events, the schema is a contract. Version it. Use a schema registry (Confluent Schema Registry with Avro or Protobuf). Backward-compatible additions only.

**4. No distributed tracing:** In an event-driven system, a request spans multiple services and events. Without a correlation ID propagated through every event, debugging is nearly impossible.

```python
# Propagate correlation ID through events
@dataclass
class OrderPlaced:
    order_id: str
    correlation_id: str  # carries through all downstream events
    ...
```

**5. "Event hell":** Too many fine-grained events with no clear ownership. Define events at the business domain level: `OrderPlaced`, `PaymentProcessed`, `OrderShipped`. Not: `OrderStatusFieldUpdated`.

### The Decision

Use event-driven architecture when:
- Services are genuinely independent and can process events asynchronously
- You need audit trails (the event log is the audit trail)
- You need to fan out one write to multiple consumers without tight coupling
- High-throughput write scenarios where synchronous coupling would be a bottleneck

Do not use when:
- You need immediate consistency across the operation
- The "events" are really just database change notifications with no business meaning
- The team doesn't yet have the observability infrastructure to trace across services

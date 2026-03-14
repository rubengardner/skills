# Specialist F — Resilience Patterns

## THE RESILIENCE LAWS

1. **Classify errors before handling.** Transient → retry. Permanent → DLQ. Never retry permanent errors.
2. **Every pipeline must be idempotent.** Re-running produces the same result, no duplicates.
3. **Every stage must be checkpointed.** On failure, resume from the last checkpoint, not from scratch.
4. **Bound your concurrency.** Unbounded concurrency is a resource exhaustion bug.
5. **Fail fast, not silently.** Never swallow exceptions with bare `except: pass`.

---

## 1. ERROR CLASSIFICATION

```python
# Define error hierarchy up front — never mix transient and permanent handling

class PipelineError(Exception):
    pass

class TransientError(PipelineError):
    """Safe to retry: network timeout, rate limit, temporary service unavailability."""
    pass

class PermanentError(PipelineError):
    """Do not retry: invalid schema, missing required field, auth failure, business rule violation."""
    pass

# Map third-party exceptions to your hierarchy at the boundary
def call_external_api(payload: dict) -> dict:
    try:
        response = httpx.post(API_URL, json=payload, timeout=10.0)
        if response.status_code == 429:
            raise TransientError("rate limited")
        if response.status_code == 422:
            raise PermanentError(f"invalid payload: {response.text}")
        response.raise_for_status()
        return response.json()
    except httpx.TimeoutException:
        raise TransientError("request timed out")
    except httpx.HTTPStatusError as e:
        raise TransientError(f"HTTP {e.response.status_code}") if e.response.status_code >= 500 else PermanentError(str(e))
```

---

## 2. RETRIES WITH EXPONENTIAL BACKOFF + JITTER

```python
import random
import time
from typing import TypeVar, Callable

T = TypeVar("T")

def retry_with_backoff(
    fn: Callable[[], T],
    max_retries: int = 5,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
) -> T:
    """
    Retry fn on TransientError with exponential backoff and jitter.
    Raises immediately on PermanentError.
    Raises the last TransientError after max_retries.
    """
    last_error: Exception | None = None

    for attempt in range(max_retries):
        try:
            return fn()
        except PermanentError:
            raise  # Never retry permanent errors
        except TransientError as e:
            last_error = e
            if attempt == max_retries - 1:
                break
            # Exponential backoff with full jitter
            delay = min(base_delay * (2 ** attempt), max_delay)
            jitter = random.uniform(0, delay * 0.1)
            time.sleep(delay + jitter)

    raise last_error  # type: ignore

# Usage
result = retry_with_backoff(lambda: call_external_api(payload))
```

---

## 3. IDEMPOTENCY

Every write operation must produce the same result when executed multiple times. Three patterns:

### Pattern A: Upsert (database writes)

```python
# PostgreSQL upsert — safe to re-run
def upsert_order(order: OrderRecord) -> None:
    db.execute("""
        INSERT INTO orders (order_id, customer_id, amount, created_at)
        VALUES (%(order_id)s, %(customer_id)s, %(amount)s, %(created_at)s)
        ON CONFLICT (order_id)
        DO UPDATE SET
            amount = EXCLUDED.amount,
            updated_at = NOW()
    """, order.model_dump())
```

### Pattern B: Check-then-skip

```python
def process_event(event_id: str, event: dict) -> None:
    # Check if already processed
    if ProcessedEvent.objects.filter(event_id=event_id).exists():
        logger.info("event_already_processed", event_id=event_id)
        return  # idempotent skip

    with transaction.atomic():
        process_business_logic(event)
        ProcessedEvent.objects.create(event_id=event_id, processed_at=now())
```

### Pattern C: Idempotency keys for external calls

```python
import uuid

def charge_customer(customer_id: str, amount: float, order_id: str) -> dict:
    # Use order_id as idempotency key — same order_id = same charge, regardless of retries
    return payment_gateway.charge(
        customer_id=customer_id,
        amount=amount,
        idempotency_key=f"charge-{order_id}",  # deterministic key
    )
```

---

## 4. CHECKPOINTING

Save pipeline progress after each stage. On failure, resume from the last checkpoint.

```python
import json
from pathlib import Path
from datetime import datetime

class PipelineCheckpoint:
    def __init__(self, pipeline_id: str, checkpoint_dir: str = "/tmp/checkpoints"):
        self.path = Path(checkpoint_dir) / f"{pipeline_id}.json"
        self.path.parent.mkdir(parents=True, exist_ok=True)

    def load(self) -> dict:
        if self.path.exists():
            return json.loads(self.path.read_text())
        return {"completed_stages": [], "last_offset": 0}

    def save(self, stage: str, offset: int | None = None) -> None:
        state = self.load()
        if stage not in state["completed_stages"]:
            state["completed_stages"].append(stage)
        if offset is not None:
            state["last_offset"] = offset
        state["updated_at"] = datetime.utcnow().isoformat()
        self.path.write_text(json.dumps(state))

    def is_done(self, stage: str) -> bool:
        return stage in self.load()["completed_stages"]

# Usage in a pipeline
def run_pipeline(pipeline_id: str) -> None:
    checkpoint = PipelineCheckpoint(pipeline_id)

    if not checkpoint.is_done("extract"):
        raw_data = extract()
        save_to_staging(raw_data)
        checkpoint.save("extract")

    if not checkpoint.is_done("transform"):
        raw_data = load_from_staging()
        clean_data = transform(raw_data)
        save_to_staging(clean_data, stage="clean")
        checkpoint.save("transform")

    if not checkpoint.is_done("load"):
        clean_data = load_from_staging(stage="clean")
        load_to_warehouse(clean_data)
        checkpoint.save("load")
```

---

## 5. DEAD LETTER QUEUE (DLQ)

Every pipeline that processes external input must have a DLQ for records that cannot be processed.

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Any

@dataclass
class DLQRecord:
    original: Any
    error_type: str  # "permanent" | "exhausted_retries" | "schema_invalid"
    error_message: str
    pipeline: str
    stage: str
    failed_at: datetime

class DeadLetterQueue:
    def __init__(self, storage_backend):
        self.backend = storage_backend  # DB table, S3 bucket, Kafka topic

    def send(self, record: DLQRecord) -> None:
        self.backend.write(record)
        # Alert on high DLQ rate (connect to your metrics system)

dlq = DeadLetterQueue(storage_backend=PostgresDLQBackend())

def safe_process(record: dict, pipeline: str, stage: str) -> None:
    try:
        process(record)
    except PermanentError as e:
        dlq.send(DLQRecord(
            original=record,
            error_type="permanent",
            error_message=str(e),
            pipeline=pipeline,
            stage=stage,
            failed_at=datetime.utcnow(),
        ))
    except TransientError as e:
        # retry_with_backoff handles this; DLQ after exhaustion
        raise
```

---

## 6. CIRCUIT BREAKER

Stops calling a failing dependency to allow it to recover. Prevents cascading failures.

```python
# pip install pybreaker
import pybreaker

# After 5 failures, open the circuit for 60 seconds
breaker = pybreaker.CircuitBreaker(fail_max=5, reset_timeout=60)

@breaker
def call_enrichment_service(customer_id: str) -> dict:
    return httpx.get(f"{ENRICHMENT_URL}/{customer_id}", timeout=5).json()

# Wrap in your error handler
def safe_enrich(customer_id: str) -> dict | None:
    try:
        return call_enrichment_service(customer_id)
    except pybreaker.CircuitBreakerError:
        # Circuit is open — service is down, skip enrichment, continue pipeline
        logger.warning("circuit_open", service="enrichment", customer_id=customer_id)
        return None
    except TransientError:
        return None
```

**States:**
- **Closed** (normal): calls pass through, failures counted
- **Open** (failing): calls fail immediately without touching the service
- **Half-open** (recovering): one probe call; if it succeeds, closes the circuit

---

## 7. BACKPRESSURE

Prevents a fast producer from overwhelming a slow consumer.

### Pattern A: Bounded async queue

```python
import asyncio

# maxsize enforces backpressure: producer blocks when queue is full
queue = asyncio.Queue(maxsize=500)

async def producer(queue: asyncio.Queue) -> None:
    async for record in stream_source():
        await queue.put(record)  # blocks when queue is full
    await queue.put(None)        # sentinel

async def consumer(queue: asyncio.Queue, semaphore: asyncio.Semaphore) -> None:
    while True:
        record = await queue.get()
        if record is None:
            await queue.put(None)
            break
        async with semaphore:    # limits concurrent processing
            await process(record)
        queue.task_done()
```

### Pattern B: Kafka prefetch (RabbitMQ / Kafka consumer)

```python
# RabbitMQ: process only 1 message at a time before acking
channel.basic_qos(prefetch_count=1)

# Kafka: limit how many messages are buffered per poll
consumer = Consumer({
    "max.poll.records": 500,  # max records per poll call
    "fetch.max.bytes": 52428800,  # 50 MB max per fetch
})
```

### Pattern C: Token bucket rate limiter

```python
import asyncio
import time

class TokenBucket:
    def __init__(self, rate: float, capacity: float):
        self.rate = rate        # tokens per second
        self.capacity = capacity
        self.tokens = capacity
        self.last_refill = time.monotonic()

    async def acquire(self) -> None:
        while True:
            now = time.monotonic()
            elapsed = now - self.last_refill
            self.tokens = min(self.capacity, self.tokens + elapsed * self.rate)
            self.last_refill = now

            if self.tokens >= 1:
                self.tokens -= 1
                return

            wait = (1 - self.tokens) / self.rate
            await asyncio.sleep(wait)

limiter = TokenBucket(rate=100, capacity=100)  # 100 requests/sec

async def rate_limited_fetch(url: str) -> dict:
    await limiter.acquire()
    return await client.get(url)
```

---

## 8. SAGA PATTERN (Distributed Transactions)

When a pipeline writes to multiple systems and partial failures must be rolled back.

```python
# Prefect 3.x compensation pattern
from prefect import flow, task
from prefect.transactions import transaction

@task
def create_order(order_data: dict) -> str:
    order_id = db.orders.insert(order_data)
    return order_id

@create_order.on_rollback
def cancel_order(txn):
    db.orders.delete(txn.get("order_id"))

@task
def reserve_inventory(order_id: str, items: list) -> None:
    inventory.reserve(order_id, items)

@reserve_inventory.on_rollback
def release_inventory(txn):
    inventory.release(txn.get("order_id"))

@flow
def place_order(order_data: dict, items: list) -> str:
    with transaction():
        order_id = create_order(order_data)
        reserve_inventory(order_id, items)  # if this fails, create_order rolls back
    return order_id
```

# Specialist C — Single-Machine Pipelines

## WHEN THIS IS ENOUGH

A single machine is sufficient when:
- Dataset fits in RAM (< 16 GB) or can be streamed in chunks
- Processing completes within acceptable time (minutes to low hours)
- The bottleneck is I/O, not CPU across many cores
- Team is small (1–5 engineers)
- You need easy local debugging and reproducibility

**Always start here. Only scale out after profiling proves you must.**

---

## POLARS — DEFAULT DATAFRAME LIBRARY

Polars (Rust-backed) is the default for single-machine dataframe work. It is 10–50× faster than Pandas for most operations, uses Apache Arrow format, and supports lazy evaluation for query optimization.

### Lazy Evaluation (preferred pattern)

```python
import polars as pl

result = (
    pl.scan_parquet("data/*.parquet")       # lazy scan, no data loaded yet
    .filter(pl.col("status") == "active")
    .filter(pl.col("amount") > 0)
    .group_by("customer_id")
    .agg([
        pl.col("amount").sum().alias("total_amount"),
        pl.col("order_id").count().alias("order_count"),
    ])
    .sort("total_amount", descending=True)
    .collect()                              # execute the full optimized plan
)
```

### Reading in Chunks (when data exceeds RAM)

```python
def process_in_chunks(file_path: str, chunk_size: int = 100_000) -> None:
    reader = pl.read_csv_batched(file_path, batch_size=chunk_size)

    while True:
        batches = reader.next_batches(5)  # read 5 chunks at a time
        if not batches:
            break
        for batch in batches:
            processed = transform(batch)
            write_to_sink(processed)
```

### Schema Validation at Ingestion

```python
from pydantic import BaseModel, field_validator
from datetime import datetime

class OrderRecord(BaseModel):
    order_id: str
    customer_id: str
    amount: float
    created_at: datetime

    @field_validator("amount")
    @classmethod
    def amount_must_be_positive(cls, v: float) -> float:
        if v <= 0:
            raise ValueError(f"amount must be positive, got {v}")
        return v

def validate_batch(records: list[dict]) -> tuple[list[OrderRecord], list[dict]]:
    valid, invalid = [], []
    for r in records:
        try:
            valid.append(OrderRecord(**r))
        except ValueError:
            invalid.append(r)  # route to DLQ
    return valid, invalid
```

---

## ASYNCIO — FOR I/O-BOUND PIPELINES

Use `asyncio` when the bottleneck is waiting on external I/O (APIs, databases), not CPU. Running 50 concurrent HTTP requests serially is the most common avoidable bottleneck.

### Pattern: Bounded Concurrent Fetching

```python
import asyncio
import httpx
from typing import Any

async def fetch_one(client: httpx.AsyncClient, record_id: str) -> dict:
    response = await client.get(f"/api/records/{record_id}", timeout=10.0)
    response.raise_for_status()
    return response.json()

async def fetch_all(record_ids: list[str], max_concurrent: int = 50) -> list[dict]:
    semaphore = asyncio.Semaphore(max_concurrent)  # bound concurrency

    async def bounded_fetch(record_id: str) -> dict:
        async with semaphore:
            return await fetch_one(client, record_id)

    async with httpx.AsyncClient() as client:
        tasks = [bounded_fetch(rid) for rid in record_ids]
        return await asyncio.gather(*tasks, return_exceptions=False)
```

### Pattern: Async Pipeline with Bounded Queue (producer-consumer)

```python
import asyncio

async def producer(queue: asyncio.Queue, sources: list[str]) -> None:
    for source in sources:
        data = await fetch(source)
        await queue.put(data)          # blocks if queue is full (backpressure)
    await queue.put(None)              # sentinel to stop consumers

async def consumer(queue: asyncio.Queue, sink) -> None:
    while True:
        item = await queue.get()
        if item is None:               # sentinel received
            await queue.put(None)      # pass sentinel to other consumers
            break
        processed = transform(item)
        await sink.write(processed)
        queue.task_done()

async def run_pipeline(sources: list[str], num_consumers: int = 5) -> None:
    queue = asyncio.Queue(maxsize=100)  # maxsize enforces backpressure

    producer_task = asyncio.create_task(producer(queue, sources))
    consumer_tasks = [
        asyncio.create_task(consumer(queue, sink))
        for _ in range(num_consumers)
    ]

    await asyncio.gather(producer_task, *consumer_tasks)
```

---

## MULTIPROCESSING — FOR CPU-BOUND TRANSFORMS

Use `multiprocessing` (not `threading`) for CPU-bound work. Python's GIL means threads don't parallelize CPU computation.

```python
from concurrent.futures import ProcessPoolExecutor
from functools import partial

def transform_partition(records: list[dict], config: dict) -> list[dict]:
    # CPU-heavy transform runs in a separate process
    return [heavy_transform(r, config) for r in records]

def parallel_transform(all_records: list[dict], num_workers: int = 4) -> list[dict]:
    # Split into partitions
    chunk_size = len(all_records) // num_workers
    partitions = [
        all_records[i:i + chunk_size]
        for i in range(0, len(all_records), chunk_size)
    ]

    with ProcessPoolExecutor(max_workers=num_workers) as executor:
        results = list(executor.map(
            partial(transform_partition, config=CONFIG),
            partitions
        ))

    return [record for partition in results for record in partition]
```

---

## GENERATOR-BASED PIPELINES (memory-efficient)

Use generators when data must flow through multiple stages without loading it all into memory.

```python
from typing import Generator, Iterator
import csv

def extract(file_path: str) -> Generator[dict, None, None]:
    with open(file_path) as f:
        reader = csv.DictReader(f)
        for row in reader:
            yield row  # one row at a time, never full file in memory

def transform(records: Iterator[dict]) -> Generator[dict, None, None]:
    for record in records:
        if record["status"] == "active":  # filter
            yield {
                "id": record["id"],
                "amount": float(record["amount"]),
            }

def load(records: Iterator[dict], batch_size: int = 1000) -> int:
    batch, total = [], 0
    for record in records:
        batch.append(record)
        if len(batch) >= batch_size:
            db.bulk_insert(batch)
            total += len(batch)
            batch.clear()
    if batch:
        db.bulk_insert(batch)
        total += len(batch)
    return total

# Compose: no intermediate lists in memory
def run():
    raw = extract("large_file.csv")
    clean = transform(raw)
    count = load(clean)
    print(f"Loaded {count} records")
```

---

## WHEN SINGLE-MACHINE IS NOT ENOUGH

Distribute only when you can measure one of these:
- Data volume regularly exceeds 80% of available RAM and chunking is impractical
- Processing time misses SLAs after optimizing transforms (Polars, compiled code)
- Horizontal failover is required for a pipeline with a real-time SLA
- ML training requires multiple GPUs

Do not distribute because it "seems more scalable." Measure first.

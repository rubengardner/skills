# Specialist B — Orchestration

## WHAT ORCHESTRATION DOES

An orchestrator handles: scheduling, dependency ordering, retries, logging, result persistence, and parallelism across pipeline stages. Without an orchestrator, you write all of that yourself and get it wrong.

---

## PREFECT 3.X — CANONICAL PATTERNS

### Basic Flow Structure

```python
from prefect import flow, task
from prefect.tasks import task_input_hash
from datetime import timedelta

@task(
    retries=3,
    retry_delay_seconds=exponential_backoff(backoff_factor=2),
    cache_key_fn=task_input_hash,
    cache_expiration=timedelta(hours=1),
    tags=["extract"],
)
def extract(source_id: str) -> list[dict]:
    return fetch_from_api(source_id)

@task(retries=2, retry_delay_seconds=10)
def transform(records: list[dict]) -> list[dict]:
    return [clean(r) for r in records]

@task
def load(records: list[dict], destination: str) -> int:
    return write_to_warehouse(records, destination)

@flow(name="daily-etl", log_prints=True)
def run_pipeline(source_ids: list[str], destination: str = "warehouse") -> dict:
    results = {}
    for sid in source_ids:
        raw = extract(sid)
        clean = transform(raw)
        count = load(clean, destination)
        results[sid] = count
    return results
```

### Dynamic Task Mapping (parallel fan-out)

```python
@flow
def parallel_pipeline(source_ids: list[str]):
    # All extract tasks run in parallel
    raw_futures = extract.map(source_ids)
    clean_futures = transform.map(raw_futures)
    load.map(clean_futures)
```

### Idempotent Caching (skip already-processed data)

```python
from prefect.tasks import task_input_hash

@task(cache_key_fn=task_input_hash, cache_expiration=timedelta(hours=24))
def extract_partition(date: str, source: str) -> list[dict]:
    # Prefect skips this call if the same (date, source) was run within 24h
    return fetch(date, source)
```

### Deployment and Scheduling

```python
# deploy.py
from prefect import serve
from myapp.pipelines import run_pipeline

if __name__ == "__main__":
    run_pipeline.serve(
        name="daily-etl-deployment",
        cron="0 6 * * *",  # 6 AM daily
        parameters={"source_ids": ["src-1", "src-2"]},
    )
```

---

## DAGSTER — CANONICAL PATTERNS

### Asset-Centric Design

```python
from dagster import asset, AssetIn, MetadataValue
import polars as pl

@asset(
    description="Raw orders from source database",
    metadata={"source": "postgres.orders_raw"},
)
def raw_orders() -> pl.DataFrame:
    return read_postgres("SELECT * FROM orders_raw WHERE updated_at > %(since)s")

@asset(
    ins={"raw_orders": AssetIn()},
    description="Validated and cleaned orders",
)
def cleaned_orders(raw_orders: pl.DataFrame) -> pl.DataFrame:
    return (
        raw_orders
        .filter(pl.col("amount") > 0)
        .drop_nulls(["order_id", "customer_id"])
    )

@asset(
    ins={"cleaned_orders": AssetIn()},
    description="Daily revenue aggregated by customer",
)
def daily_revenue(cleaned_orders: pl.DataFrame) -> pl.DataFrame:
    return (
        cleaned_orders
        .group_by(["customer_id", pl.col("created_at").dt.date().alias("date")])
        .agg(pl.col("amount").sum().alias("revenue"))
    )
```

### Partitioned Assets (process data in date slices)

```python
from dagster import asset, DailyPartitionsDefinition

@asset(partitions_def=DailyPartitionsDefinition(start_date="2024-01-01"))
def daily_orders(context) -> pl.DataFrame:
    date = context.partition_key  # "2024-03-14"
    return read_postgres(f"SELECT * FROM orders WHERE date = '{date}'")
```

### Freshness Policies (SLA enforcement)

```python
from dagster import FreshnessPolicy

@asset(freshness_policy=FreshnessPolicy(maximum_lag_minutes=60))
def dashboard_metrics(daily_revenue: pl.DataFrame) -> pl.DataFrame:
    # Dagster alerts if this asset is > 60 min stale
    ...
```

---

## AIRFLOW 3.X — USE ONLY FOR EXISTING DEPLOYMENTS

### Modern Task Flow API

```python
from airflow.decorators import dag, task
from datetime import datetime, timedelta

@dag(
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    default_args={"retries": 3, "retry_delay": timedelta(minutes=5)},
)
def daily_pipeline():
    @task
    def extract() -> list[dict]:
        return fetch_data()

    @task
    def transform(data: list[dict]) -> list[dict]:
        return clean(data)

    @task
    def load(data: list[dict]) -> None:
        write_to_warehouse(data)

    load(transform(extract()))

dag = daily_pipeline()
```

### Dynamic Task Mapping (Airflow 2.3+)

```python
@task
def process_partition(partition_id: int) -> dict:
    return run_on_partition(partition_id)

# Expands to N parallel tasks at runtime
results = process_partition.expand(partition_id=list(range(50)))
```

---

## RETRY CONFIGURATION (ALL ORCHESTRATORS)

**Rule**: Retries are for transient failures (network timeout, rate limit, temporary unavailability). Never retry permanent failures (invalid schema, missing required field, auth error).

```python
# Prefect: exponential backoff
@task(
    retries=5,
    retry_delay_seconds=[10, 30, 60, 120, 300],  # explicit backoff schedule
)
def flaky_api_call(): ...

# Or use the built-in helper
from prefect.tasks import exponential_backoff
@task(retries=4, retry_delay_seconds=exponential_backoff(backoff_factor=2))
def flaky_api_call(): ...
```

---

## ORCHESTRATOR COMPARISON

| Feature | Prefect 3.x | Dagster | Airflow 3.x |
|---------|-------------|---------|-------------|
| API style | Python functions | Asset decorators | Python + YAML |
| Lineage | Basic | First-class | Limited |
| Dynamic tasks | Yes | Yes | Yes (2.3+) |
| Local dev | Excellent | Good | Painful |
| Managed cloud | Yes (Prefect Cloud) | Yes (Dagster Cloud) | Yes (MWAA, Astro) |
| Learning curve | Low | Medium | High |
| Best for | Dynamic workflows | Data mesh / lineage | Enterprise legacy |

# Specialist D — Distributed Compute

## WHEN TO USE DISTRIBUTED

Distribute only when single-machine is provably insufficient:
- Regularly processing > 100 GB batches
- ML training requiring multiple GPUs
- Fault-tolerant SLAs requiring horizontal failover for batch jobs
- Processing time SLA cannot be met on a single node after optimization

---

## DASK — SCALE-OUT PANDAS/POLARS

Dask is the default scale-out option for Python analytical workloads. It mirrors the Pandas/NumPy API and runs on a local cluster (multiprocessing), a remote cluster, or Kubernetes.

### Basic Dask DataFrame

```python
import dask.dataframe as dd

# Reads all parquet files lazily across workers
df = dd.read_parquet("s3://my-bucket/data/*.parquet")

result = (
    df[df["status"] == "active"]
    .groupby("customer_id")["amount"]
    .sum()
    .reset_index()
    .compute()  # triggers distributed execution
)
```

### Custom Transforms with Dask

```python
import dask.dataframe as dd
import pandas as pd

def process_partition(partition: pd.DataFrame) -> pd.DataFrame:
    # Each worker runs this on its own partition
    return partition.assign(
        score=partition["amount"] * partition["weight"]
    )

df = dd.read_parquet("s3://my-bucket/data/*.parquet")
result = df.map_partitions(process_partition).compute()
```

### Dask Cluster Setup

```python
from dask.distributed import Client

# Local cluster (uses all CPU cores)
client = Client()  # automatically creates LocalCluster

# Remote cluster (Kubernetes, YARN, etc.)
client = Client("tcp://scheduler-address:8786")

print(client.dashboard_link)  # Dask dashboard for monitoring
```

### Partitioning Strategy

```python
# Repartition before a heavy groupby to co-locate related keys
df = df.repartition(npartitions=200)

# Partition by a key column (avoids shuffle during groupby)
df = df.set_index("customer_id", sorted=False)
```

---

## RAY — ML WORKLOADS AND GPU DISTRIBUTION

Ray is the correct tool for ML training, model serving, and AI pipelines. It handles GPU scheduling, actor model (stateful workers), and integrates with PyTorch/JAX.

### Parallel Data Processing with Ray

```python
import ray

ray.init()  # auto-detects local CPUs/GPUs

@ray.remote
def process_shard(shard: list[dict]) -> list[dict]:
    # Runs on a Ray worker (separate process or remote node)
    return [transform(record) for record in shard]

def run_distributed(records: list[dict], num_shards: int = 10) -> list[dict]:
    # Split into shards
    shard_size = len(records) // num_shards
    shards = [records[i:i+shard_size] for i in range(0, len(records), shard_size)]

    # Fan out (submit all tasks)
    futures = [process_shard.remote(shard) for shard in shards]

    # Fan in (collect results)
    results = ray.get(futures)
    return [r for shard_result in results for r in shard_result]
```

### Ray for GPU Training

```python
import ray

@ray.remote(num_gpus=1)
def train_on_shard(data_shard_path: str, model_config: dict) -> dict:
    model = build_model(model_config)
    data = load(data_shard_path)
    metrics = model.fit(data)
    return metrics

ray.init()
shards = ["s3://bucket/shard_0.parquet", "s3://bucket/shard_1.parquet"]
futures = [train_on_shard.remote(s, MODEL_CONFIG) for s in shards]
all_metrics = ray.get(futures)
```

### Ray Actors (stateful distributed workers)

```python
@ray.remote
class DataProcessor:
    def __init__(self):
        self.processed_count = 0
        self.db = connect_to_db()  # connection held in the actor

    def process(self, batch: list[dict]) -> int:
        valid = validate(batch)
        self.db.bulk_insert(valid)
        self.processed_count += len(valid)
        return len(valid)

    def stats(self) -> dict:
        return {"processed": self.processed_count}

# Create pool of stateful workers
workers = [DataProcessor.remote() for _ in range(4)]
futures = [workers[i % 4].process.remote(batch) for i, batch in enumerate(batches)]
ray.get(futures)
```

---

## PYSPARK — MASSIVE STRUCTURED ETL

Use PySpark when processing > 500 GB of structured data or when you need the full Spark ecosystem (Delta Lake, Spark SQL, structured streaming).

### Basic Spark Pipeline

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.types import StructType, StructField, StringType, DoubleType

schema = StructType([
    StructField("order_id", StringType(), nullable=False),
    StructField("customer_id", StringType(), nullable=False),
    StructField("amount", DoubleType(), nullable=False),
])

spark = SparkSession.builder.appName("daily-etl").getOrCreate()

df = (
    spark.read.schema(schema).parquet("s3://bucket/orders/")
    .filter(F.col("amount") > 0)
    .groupBy("customer_id")
    .agg(F.sum("amount").alias("total"), F.count("order_id").alias("orders"))
    .orderBy(F.col("total").desc())
)

df.write.mode("overwrite").parquet("s3://bucket/output/customer_revenue/")
spark.stop()
```

### Delta Lake (ACID transactions on data lake)

```python
df.write.format("delta").mode("merge").save("s3://bucket/delta/orders/")
# Supports UPDATE, DELETE, time travel, and schema evolution
```

---

## FAN-OUT / FAN-IN PATTERN

The universal distributed compute pattern: scatter work to N workers, collect results.

```python
# With Prefect + Dask backend
from prefect import flow, task
from prefect_dask import DaskTaskRunner

@task
def process_partition(partition_id: int) -> dict:
    data = load_partition(partition_id)
    return aggregate(data)

@task
def merge_results(results: list[dict]) -> dict:
    return {k: v for r in results for k, v in r.items()}

@flow(task_runner=DaskTaskRunner())
def distributed_pipeline(num_partitions: int = 100):
    futures = process_partition.map(list(range(num_partitions)))
    merged = merge_results(futures)
    return merged
```

---

## TOOL SELECTION SUMMARY

| Tool | Use case | Scale | Key strength |
|------|----------|-------|-------------|
| Dask | Analytical Python workloads | 10 GB – multi-TB | Pandas/NumPy API compatibility |
| Ray | ML training, AI agents, GPU | Any (multi-GPU) | Actor model, GPU scheduling |
| PySpark | Structured ETL, SQL-heavy | 500 GB – petabytes | Spark SQL, Delta Lake, ecosystem |

**Never use all three on the same pipeline.** Pick one based on the primary workload type.

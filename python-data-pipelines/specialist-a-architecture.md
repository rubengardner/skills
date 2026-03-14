# Specialist A — Architecture Decision Framework

## THE ONE RULE

Start with a single machine. Prove it is insufficient. Then distribute only the bottleneck.

Distributing prematurely multiplies: operational complexity, debugging surface, network failure modes, and infrastructure cost — for zero benefit if your data fits on one machine.

---

## DECISION TREE

```
1. Does your data fit in RAM (< 16 GB)?
   YES → Single-machine Polars. Done.
   NO  → Continue.

2. Can you process in chunks (streaming reads, pagination)?
   YES → Single-machine chunked pipeline with Polars or generators. Done.
   NO  → Continue.

3. Is your bottleneck CPU-bound (transforms, ML inference)?
   YES → Dask (scale-out Python) or Ray (ML/GPU). Continue to step 4.
   NO (I/O-bound) → Async I/O (asyncio + httpx). Done.

4. Is the data structured and > 500 GB?
   YES → PySpark.
   NO  → Dask cluster.

5. Do you need sub-second latency (real-time stream)?
   YES → Kafka + stream processor (Flink or Spark Streaming).
   NO  → Batch pipeline is fine.
```

---

## STACK SELECTION BY SCENARIO

### Scenario 1: ETL for a small/medium product (< 50 GB, batch)
```
Polars (transforms) + Prefect (orchestration) + PostgreSQL/S3 (storage)
```
Run on a single beefy machine (32–64 GB RAM). Simple to operate. Debug locally. No cluster needed.

### Scenario 2: Data platform with lineage requirements
```
Dagster + Polars/dbt + Snowflake or BigQuery
```
Asset-centric design gives automatic lineage, freshness SLAs, and impact analysis. Use when stakeholders need to ask "what broke downstream?"

### Scenario 3: Large-scale batch ETL (> 500 GB)
```
Airflow (or Dagster) + PySpark on EMR/Databricks
```
PySpark for the compute. Airflow for orchestration if the team already has it; Dagster otherwise.

### Scenario 4: ML platform
```
Ray (distributed training) + MLflow (tracking) + Dagster (asset orchestration)
```
Ray handles GPU distribution natively. Dagster tracks model artifacts as assets.

### Scenario 5: Real-time event ingestion
```
Kafka (broker) + Flink or Spark Streaming (processing) + Redis/Postgres (serving)
```
Use Kafka consumer groups to scale consumers independently. Each consumer group is a separate pipeline step.

### Scenario 6: App background tasks
```
Celery + Redis (broker) + Django/FastAPI (trigger)
```
Not a data pipeline — a task queue. Appropriate for: email sending, PDF generation, webhook delivery, single-record enrichment. Not appropriate for: multi-step ETL, streaming ingestion, large data transforms.

---

## ORCHESTRATION TOOL SELECTION

### Prefect 3.x — default for new projects
- Pythonic API. Minimal boilerplate. Flows are just decorated functions.
- Built-in: retries, caching, result persistence, dynamic task mapping.
- Deploy with Prefect Cloud (managed) or self-hosted worker.
- **Use when:** startup, dynamic workflows, rapid iteration, team < 10 engineers.

### Dagster — when you need data lineage
- Asset-centric: pipelines modeled as data products, not tasks.
- Superior observability: asset materialization history, freshness policies, downstream impact.
- Native dbt, Spark, Airbyte integrations.
- **Use when:** data mesh, strict lineage requirements, multiple consumers of the same data assets.

### Airflow 3.x — enterprise with existing investment
- 320M+ downloads/year. Largest ecosystem. DAG versioning added in 3.0.
- Task-centric (not asset-centric). Heavy setup.
- **Use when:** existing Airflow deployment, large enterprise team, broad provider ecosystem needed.

### Never use Luigi
- Abandoned. No activity since 2023. Use Prefect or Dagster instead.

---

## COST MODEL

| Factor | Single-machine | Distributed |
|--------|---------------|-------------|
| Infra cost | Low (1 VM) | High (cluster + orchestrator) |
| Debug ease | High (local) | Low (distributed logs) |
| Failure modes | Few | Many (network, partition, node) |
| Ops burden | Minimal | Significant |
| Scale limit | ~100 GB | Unlimited |
| Time to first run | Hours | Days–Weeks |

**Default answer:** Single-machine is correct until proven wrong by data volume or SLA requirements.

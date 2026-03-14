# Specialist I — Anti-Patterns

## THE FATAL SINS

These patterns cause silent data loss, non-deterministic failures, or un-debuggable pipelines. Treat all of them as hard errors in code review.

---

## 1. SILENT EXCEPTION SWALLOWING

**Never do this:**
```python
# WRONG: silently drops records with no trace
for record in records:
    try:
        process(record)
    except Exception:
        pass  # record lost, no log, no DLQ

# WRONG: logs but still loses the record
    except Exception as e:
        print(f"Error: {e}")  # print is not structured logging
```

**Do this instead:**
```python
for record in records:
    try:
        process(record)
    except PermanentError as e:
        logger.error("record_rejected", record_id=record["id"], error=str(e))
        dlq.send(record)
    except TransientError as e:
        logger.warning("record_transient_error", record_id=record["id"], error=str(e))
        retry_queue.put(record)
```

---

## 2. NON-IDEMPOTENT WRITES

**Never do this:**
```python
# WRONG: creates duplicates on re-run
def load(records):
    for r in records:
        db.execute("INSERT INTO orders VALUES (%s, %s)", (r["id"], r["amount"]))
```

**Do this instead:**
```python
def load(records):
    for r in records:
        db.execute("""
            INSERT INTO orders (id, amount) VALUES (%s, %s)
            ON CONFLICT (id) DO UPDATE SET amount = EXCLUDED.amount
        """, (r["id"], r["amount"]))
```

---

## 3. RUNNING ETL INSIDE A WEB REQUEST

**Never do this:**
```python
# WRONG: blocks the web request, ties up a thread for minutes
def upload_and_process(request):
    file = request.FILES["data"]
    records = parse_csv(file)       # could take seconds
    transformed = run_transforms(records)  # could take minutes
    db.bulk_insert(transformed)
    return JsonResponse({"status": "done"})
```

**Do this instead:**
```python
def upload_and_process(request):
    upload = save_upload(request.FILES["data"])
    task = process_file.delay(str(upload.id))   # async, returns immediately
    return JsonResponse({"task_id": task.id, "status": "queued"})
```

---

## 4. CELERY AS AN ETL ORCHESTRATOR

**Never do this:**
```python
# WRONG: chaining Celery tasks as a data pipeline
@shared_task
def extract(): return fetch_all_data()

@shared_task
def transform(data): return clean(data)

@shared_task
def load(data): return db.insert(data)

# Celery chains have no checkpointing, no visibility, no retry-from-checkpoint
chain(extract.s() | transform.s() | load.s()).apply_async()
```

**Do this instead:**
Use Prefect or Dagster for multi-step pipelines. Celery is for single operations, not workflows.

---

## 5. UNBOUNDED CONCURRENCY

**Never do this:**
```python
# WRONG: creates thousands of concurrent coroutines — exhausts connections and memory
async def run():
    tasks = [process(record) for record in all_100k_records]
    await asyncio.gather(*tasks)  # 100,000 concurrent tasks
```

**Do this instead:**
```python
async def run():
    semaphore = asyncio.Semaphore(50)  # max 50 concurrent

    async def bounded(record):
        async with semaphore:
            return await process(record)

    await asyncio.gather(*[bounded(r) for r in all_records])
```

---

## 6. USING LUIGI IN A NEW PROJECT

Luigi is abandoned (no commits since 2023). It lacks: async support, modern retry primitives, built-in scheduling, cloud-native deployment.

**Never do this:** Start a new project with Luigi.
**Do this instead:** Prefect 3.x (simple) or Dagster (asset-lineage).

---

## 7. SCHEMA-FREE PIPELINES

**Never do this:**
```python
# WRONG: passes raw dicts through the whole pipeline with no validation
def extract() -> list[dict]:
    return requests.get(API_URL).json()["records"]  # could be anything

def transform(records: list[dict]) -> list[dict]:
    return [{"revenue": r["amount"] * r["qty"]} for r in records]  # KeyError waiting to happen
```

**Do this instead:**
```python
class RawRecord(BaseModel):
    amount: float
    qty: int

class TransformedRecord(BaseModel):
    revenue: float

def extract() -> list[RawRecord]:
    raw = requests.get(API_URL).json()["records"]
    return [RawRecord(**r) for r in raw]  # fails loudly on bad data

def transform(records: list[RawRecord]) -> list[TransformedRecord]:
    return [TransformedRecord(revenue=r.amount * r.qty) for r in records]
```

---

## 8. RETRYING PERMANENT ERRORS

**Never do this:**
```python
# WRONG: retries an invalid schema error 5 times, wastes time, spams the DLQ
@shared_task(max_retries=5, autoretry_for=(Exception,))
def process(record_id):
    record = fetch(record_id)
    validate_schema(record)  # raises ValueError if schema is invalid
    save(record)
```

**Do this instead:**
```python
@shared_task(bind=True, max_retries=5, autoretry_for=(TransientError,))
def process(self, record_id):
    try:
        record = fetch(record_id)        # retries on TransientError
    except TransientError:
        raise  # autoretry handles this

    try:
        validate_schema(record)
    except ValueError as e:
        raise PermanentError(str(e))     # PermanentError → not in autoretry_for → DLQ

    save(record)
```

---

## 9. HARDCODED CREDENTIALS / CONFIGS

**Never do this:**
```python
KAFKA_SERVERS = "kafka.prod.internal:9092"
DB_PASSWORD = "super-secret-password-123"
```

**Do this instead:**
```python
from pydantic_settings import BaseSettings

class PipelineSettings(BaseSettings):
    kafka_bootstrap_servers: str
    db_password: str

    class Config:
        env_file = ".env"

settings = PipelineSettings()  # reads from env vars or .env file
```

---

## 10. NO CHECKPOINTING (re-runs from scratch)

**Never do this:**
```python
def run_pipeline():
    raw = extract_10_million_records()    # takes 30 min
    clean = transform(raw)                # takes 20 min
    load(clean)                           # takes 10 min
    # If load() fails after 8 minutes, you restart from extract()
```

**Do this instead:**
```python
def run_pipeline(run_id: str):
    checkpoint = PipelineCheckpoint(run_id)

    if not checkpoint.is_done("extract"):
        raw = extract_10_million_records()
        staging.save(raw, stage="raw")
        checkpoint.save("extract")

    if not checkpoint.is_done("transform"):
        raw = staging.load(stage="raw")
        clean = transform(raw)
        staging.save(clean, stage="clean")
        checkpoint.save("transform")

    if not checkpoint.is_done("load"):
        clean = staging.load(stage="clean")
        load(clean)
        checkpoint.save("load")
```

---

## 11. USING `print()` FOR PIPELINE STATUS

**Never do this:**
```python
print(f"Processing {len(records)} records...")
print("Done!")
```

**Do this instead:**
```python
logger.info("stage_started", record_count=len(records), stage="transform")
logger.info("stage_completed", stage="transform", elapsed_seconds=elapsed)
```

`print` produces unstructured output with no timestamp, no log level, no context, and no ability to query later.

---

## CODE SMELL CHECKLIST

Flag any of these as bugs in code review:

- [ ] `except Exception: pass` or `except: pass` anywhere in pipeline code
- [ ] `print()` used for status/debug output
- [ ] `asyncio.gather()` without a `Semaphore` on collections larger than 100 items
- [ ] Celery task chains used as a multi-step data pipeline
- [ ] `INSERT INTO` without `ON CONFLICT` for any table that pipelines write to
- [ ] Credentials or endpoints hardcoded in source files
- [ ] Pipeline that has no test for "re-run the same data twice"
- [ ] Task that catches all exceptions with `autoretry_for=(Exception,)`
- [ ] Pipeline that loads all data into a list before processing (no streaming/chunking)
- [ ] Any code that calls a long pipeline synchronously inside a web view

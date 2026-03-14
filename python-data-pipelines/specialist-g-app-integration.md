# Specialist G — App Integration

## THE INTEGRATION LAWS

1. **Never run a pipeline inside a web request.** Offload to a background worker. Web requests must respond in < 2 seconds.
2. **Worker processes are separate from web server processes.** Shared state via database or message broker only, never in-process memory.
3. **Pipeline triggers are events, not cron jobs.** Prefer event-driven triggers (model save, webhook) over polling where possible.
4. **Return a task ID immediately.** Let the client poll for status or use a webhook.

---

## PATTERN A: Celery for Single-Operation Background Tasks

Use when: offloading one operation from a web request (enrich, notify, resize, index).

### Complete Setup (Django + Celery + Redis)

```python
# myapp/celery.py
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
app = Celery("myapp")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

```python
# config/settings.py
CELERY_BROKER_URL = env("REDIS_URL", default="redis://localhost:6379/0")
CELERY_RESULT_BACKEND = env("REDIS_URL", default="redis://localhost:6379/0")
CELERY_TASK_SERIALIZER = "json"
CELERY_RESULT_SERIALIZER = "json"
CELERY_TASK_ACKS_LATE = True
CELERY_TASK_REJECT_ON_WORKER_LOST = True
CELERY_WORKER_PREFETCH_MULTIPLIER = 1  # process one task at a time per worker
```

```python
# myapp/tasks.py
from celery import shared_task
import structlog

logger = structlog.get_logger()

@shared_task(
    bind=True,
    max_retries=5,
    autoretry_for=(TransientError,),
    retry_backoff=True,
    retry_backoff_max=300,
    retry_jitter=True,
)
def process_uploaded_file(self, upload_id: str) -> dict:
    log = logger.bind(task_id=self.request.id, upload_id=upload_id)
    log.info("task_started")

    upload = FileUpload.objects.get(id=upload_id)
    upload.status = "processing"
    upload.save(update_fields=["status"])

    try:
        result = run_etl(upload.file_path)
        upload.status = "done"
        upload.result = result
        upload.save(update_fields=["status", "result"])
        log.info("task_completed", rows=result["row_count"])
        return result
    except PermanentError as e:
        upload.status = "failed"
        upload.error = str(e)
        upload.save(update_fields=["status", "error"])
        log.error("task_failed_permanently", error=str(e))
        raise  # do not retry
```

```python
# myapp/views.py (Django Ninja)
from ninja import Router
from .tasks import process_uploaded_file

router = Router()

@router.post("/uploads/{upload_id}/process")
def trigger_processing(request, upload_id: str):
    task = process_uploaded_file.delay(upload_id)
    return {"task_id": task.id, "status": "queued"}

@router.get("/tasks/{task_id}/status")
def get_task_status(request, task_id: str):
    from celery.result import AsyncResult
    result = AsyncResult(task_id)
    return {
        "task_id": task_id,
        "status": result.status,       # PENDING, STARTED, SUCCESS, FAILURE, RETRY
        "result": result.result if result.ready() else None,
    }
```

---

## PATTERN B: Prefect Flow Triggered from App

Use when: triggering a multi-step pipeline from a web event without needing a full message broker.

```python
# myapp/tasks.py
from prefect.deployments import run_deployment

def trigger_pipeline(source_id: str, user_id: str) -> str:
    """Trigger a Prefect flow from the Django app. Returns flow run ID."""
    flow_run = run_deployment(
        name="daily-etl/production",
        parameters={"source_id": source_id, "user_id": user_id},
        timeout=0,  # don't wait for completion
    )
    return str(flow_run.id)
```

```python
# In a Django view or Celery task:
@router.post("/pipelines/trigger")
def trigger_etl(request, payload: TriggerPayload):
    flow_run_id = trigger_pipeline(payload.source_id, str(request.user.id))
    return {"flow_run_id": flow_run_id, "status": "triggered"}
```

---

## PATTERN C: Event-Driven Ingestion via Django Signals

Use when: you want pipelines to trigger automatically on model changes without manual trigger calls.

```python
# myapp/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from .models import Order
from .tasks import sync_order_to_warehouse

@receiver(post_save, sender=Order)
def on_order_saved(sender, instance: Order, created: bool, **kwargs) -> None:
    if created or instance.status_changed:
        # Never run ETL inline — always async
        sync_order_to_warehouse.delay(str(instance.id))
```

```python
# myapp/apps.py
from django.apps import AppConfig

class MyAppConfig(AppConfig):
    def ready(self):
        import myapp.signals  # noqa: F401 — register signals on startup
```

---

## PATTERN D: Kafka Ingestion Alongside Django App

Use when: high-volume event stream needs to flow into your app's database from an external source.

### Kafka Consumer as a Standalone Management Command

```python
# myapp/management/commands/run_event_consumer.py
from django.core.management.base import BaseCommand
from confluent_kafka import Consumer, KafkaError
import json
import signal

class Command(BaseCommand):
    help = "Run the Kafka event consumer"

    def handle(self, *args, **options):
        consumer = Consumer({
            "bootstrap.servers": settings.KAFKA_BOOTSTRAP_SERVERS,
            "group.id": "myapp-consumer",
            "auto.offset.reset": "earliest",
            "enable.auto.commit": False,
        })
        consumer.subscribe(["order-events"])

        running = True
        def shutdown(sig, frame):
            nonlocal running
            running = False

        signal.signal(signal.SIGTERM, shutdown)
        signal.signal(signal.SIGINT, shutdown)

        batch = []
        try:
            while running:
                msg = consumer.poll(timeout=1.0)
                if msg is None or msg.error():
                    if batch:
                        process_batch(batch)
                        consumer.commit()
                        batch.clear()
                    continue

                batch.append(json.loads(msg.value()))

                if len(batch) >= 500:
                    process_batch(batch)
                    consumer.commit()
                    batch.clear()
        finally:
            if batch:
                process_batch(batch)
                consumer.commit()
            consumer.close()
```

### Run as a separate process (not inside Django's web process)

```yaml
# docker-compose.yml
services:
  web:
    command: gunicorn config.wsgi:application

  celery-worker:
    command: celery -A myapp worker -l info -c 4

  kafka-consumer:
    command: python manage.py run_event_consumer
```

---

## PATTERN E: Pipeline Status Webhook (push model)

Instead of polling for task status, have the pipeline call back when done.

```python
# myapp/tasks.py
@shared_task(bind=True, max_retries=3)
def run_with_callback(self, pipeline_id: str, callback_url: str) -> None:
    try:
        result = run_pipeline(pipeline_id)
        httpx.post(callback_url, json={"status": "success", "result": result}, timeout=10)
    except Exception as e:
        httpx.post(callback_url, json={"status": "failed", "error": str(e)}, timeout=10)
        raise
```

---

## ARCHITECTURE RULES

- **Web → broker → worker → database.** Data flows unidirectionally. Workers never call back into the web layer.
- **Never `celery.result.get()` in a web request.** This blocks the web thread and defeats the purpose of async.
- **Use `countdown` or `eta` for delayed tasks**, not `time.sleep()` in a task.
- **Separate queues for different priority work.** High-priority tasks (user-triggered) on `high` queue, batch jobs on `low` queue.

```python
# Priority queues
@shared_task(queue="high")
def user_triggered_export(user_id): ...

@shared_task(queue="low")
def nightly_aggregation(): ...
```

```bash
# Run separate workers per queue
celery -A myapp worker -Q high -c 8
celery -A myapp worker -Q low -c 2
```

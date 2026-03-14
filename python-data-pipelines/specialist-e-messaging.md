# Specialist E — Streaming & Messaging

## TOOL SELECTION

| Tool | Throughput | Message ordering | Persistence | Best for |
|------|-----------|-----------------|-------------|----------|
| **Kafka** | ~1M msg/sec | Per partition | Yes (log-based) | Event streaming, audit trail, replay, high throughput |
| **RabbitMQ** | ~50K msg/sec | Per queue | Yes (configurable) | Complex routing, reliable delivery, dead letters |
| **Redis Streams** | Very high | Yes | Limited (in-memory) | Ephemeral real-time, cache-adjacent use cases |
| **Celery** | Task-focused | No | Via broker | App-level async tasks (offloading from web requests) |

**Rule:** Kafka for data pipelines at scale. Celery for app-level task offloading. Never use Celery as a data pipeline orchestrator.

---

## KAFKA — EVENT STREAMING PIPELINES

### Consumer Pattern (at-least-once delivery)

```python
from confluent_kafka import Consumer, Producer, KafkaError
import json

consumer = Consumer({
    "bootstrap.servers": "kafka:9092",
    "group.id": "pipeline-group",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": False,  # manual commit for at-least-once
})
consumer.subscribe(["raw-events"])

def run_consumer(process_fn, batch_size: int = 500) -> None:
    batch = []
    try:
        while True:
            msg = consumer.poll(timeout=1.0)

            if msg is None:
                if batch:
                    flush_batch(process_fn, batch, consumer)
                    batch.clear()
                continue

            if msg.error():
                if msg.error().code() == KafkaError._PARTITION_EOF:
                    continue
                raise KafkaException(msg.error())

            batch.append(json.loads(msg.value()))

            if len(batch) >= batch_size:
                flush_batch(process_fn, batch, consumer)
                batch.clear()

    finally:
        consumer.close()

def flush_batch(process_fn, batch: list[dict], consumer: Consumer) -> None:
    process_fn(batch)
    consumer.commit()  # commit AFTER successful processing
```

### Producer Pattern (reliable delivery)

```python
from confluent_kafka import Producer
import json

producer = Producer({
    "bootstrap.servers": "kafka:9092",
    "acks": "all",               # wait for all replicas
    "retries": 5,
    "retry.backoff.ms": 200,
    "enable.idempotence": True,  # exactly-once producer semantics
})

def publish_event(topic: str, key: str, event: dict) -> None:
    def delivery_callback(err, msg):
        if err:
            # Log failure — caller must decide to retry or DLQ
            raise KafkaDeliveryError(f"Delivery failed: {err}")

    producer.produce(
        topic,
        key=key.encode(),
        value=json.dumps(event).encode(),
        callback=delivery_callback,
    )
    producer.poll(0)  # trigger callbacks

def flush_producer() -> None:
    producer.flush(timeout=10)  # wait for all outstanding messages
```

### Dead Letter Queue via Kafka

```python
DLQ_TOPIC = "pipeline.dlq"

def process_with_dlq(msg: dict, max_retries: int = 3) -> None:
    for attempt in range(max_retries):
        try:
            process(msg)
            return
        except PermanentError as e:
            # Permanent: schema invalid, missing required field
            publish_event(DLQ_TOPIC, msg["id"], {
                "original": msg,
                "error": str(e),
                "error_type": "permanent",
                "attempt": attempt,
            })
            return  # do not retry permanent errors
        except TransientError as e:
            if attempt == max_retries - 1:
                publish_event(DLQ_TOPIC, msg["id"], {
                    "original": msg,
                    "error": str(e),
                    "error_type": "exhausted",
                    "attempts": max_retries,
                })
            # else: loop retries
```

### Consumer Group Scaling

Each consumer group processes each message independently. Add consumers within a group to scale throughput (up to the number of topic partitions).

```python
# Scale by increasing partition count at topic creation
# num_consumers <= num_partitions (extra consumers idle)
# Rule: partition count = max expected parallelism

# In docker-compose or Terraform:
# kafka-topics --create --topic raw-events --partitions 20 --replication-factor 3
```

---

## CELERY — APP-LEVEL ASYNC TASKS

Celery is **not** a data pipeline orchestrator. Use it to offload single operations from web requests.

### Setup (Django example)

```python
# celery.py
from celery import Celery

app = Celery("myapp")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

```python
# settings.py
CELERY_BROKER_URL = "redis://localhost:6379/0"
CELERY_RESULT_BACKEND = "redis://localhost:6379/0"
CELERY_TASK_SERIALIZER = "json"
CELERY_TASK_ACKS_LATE = True        # ack only after successful completion
CELERY_TASK_REJECT_ON_WORKER_LOST = True  # requeue if worker crashes
```

### Task Pattern

```python
from celery import shared_task
from celery.exceptions import MaxRetriesExceededError

@shared_task(
    bind=True,
    max_retries=5,
    autoretry_for=(TransientError,),
    retry_backoff=True,           # exponential backoff
    retry_backoff_max=600,        # max 10 minutes between retries
    retry_jitter=True,            # add randomness to avoid thundering herd
)
def enrich_customer(self, customer_id: str) -> dict:
    try:
        data = fetch_from_enrichment_api(customer_id)
        Customer.objects.filter(pk=customer_id).update(**data)
        return {"customer_id": customer_id, "status": "enriched"}
    except PermanentError as e:
        # Do not retry. Log and stop.
        logger.error("permanent_error", customer_id=customer_id, error=str(e))
        raise  # will not retry (PermanentError not in autoretry_for)
```

### Triggering from Django view

```python
# views.py
from django.http import JsonResponse
from .tasks import enrich_customer

def trigger_enrichment(request, customer_id: str):
    task = enrich_customer.delay(customer_id)
    return JsonResponse({"task_id": task.id, "status": "queued"})
```

### Celery Beat (scheduled tasks)

```python
# settings.py
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    "nightly-cleanup": {
        "task": "myapp.tasks.cleanup_expired_sessions",
        "schedule": crontab(hour=2, minute=0),
    },
}
```

---

## RABBITMQ — COMPLEX ROUTING

Use RabbitMQ when you need sophisticated message routing (topic exchanges, fanout, header matching) or fine-grained acknowledgment control.

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters("rabbitmq", heartbeat=600)
)
channel = connection.channel()

channel.exchange_declare(exchange="pipeline", exchange_type="topic", durable=True)
channel.queue_declare(queue="orders.process", durable=True)
channel.queue_bind(queue="orders.process", exchange="pipeline", routing_key="orders.*")

# Set prefetch: process only 1 message at a time per consumer (backpressure)
channel.basic_qos(prefetch_count=1)

def on_message(ch, method, properties, body):
    try:
        process(json.loads(body))
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except PermanentError:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)  # → DLQ
    except TransientError:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)   # retry

channel.basic_consume(queue="orders.process", on_message_callback=on_message)
channel.start_consuming()
```

---

## REDIS STREAMS

Use Redis Streams for ephemeral, cache-adjacent real-time pipelines where message persistence beyond hours is not needed.

```python
import redis

r = redis.Redis()

# Producer
r.xadd("events", {"event_type": "order_placed", "order_id": "123", "amount": "99.9"})

# Consumer group (competing consumers)
r.xgroup_create("events", "pipeline-group", id="0", mkstream=True)

# Consumer
messages = r.xreadgroup(
    "pipeline-group", "consumer-1",
    {"events": ">"},  # ">" = undelivered messages only
    count=100,
    block=5000,       # block for 5s if no messages
)
for stream, msgs in messages:
    for msg_id, fields in msgs:
        process(fields)
        r.xack("events", "pipeline-group", msg_id)
```

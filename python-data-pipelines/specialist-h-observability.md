# Specialist H — Observability

## THE OBSERVABILITY LAWS

1. **Every pipeline stage has structured JSON logs on entry and exit.**
2. **Every log line has a correlation ID** that flows from ingestion to final write.
3. **Row counts and error counts are metrics**, not log messages.
4. **Data quality violations are observable** — bad records are counted, not silently dropped.
5. **Observability is designed in, not bolted on after.** Retrofitting is 10× harder.

---

## STRUCTURED LOGGING

Use `structlog` over standard `logging`. It produces JSON natively and supports bound loggers (correlation context flows automatically).

### Setup

```python
# config/logging.py
import structlog
import logging

def configure_logging(level: str = "INFO") -> None:
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,  # merges bound context
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.JSONRenderer(),      # structured JSON output
        ],
        wrapper_class=structlog.make_filtering_bound_logger(
            getattr(logging, level.upper())
        ),
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )
```

### Pipeline Stage Logging

```python
import structlog
from contextlib import contextmanager
import time

logger = structlog.get_logger()

@contextmanager
def pipeline_stage(name: str, correlation_id: str, **context):
    """Context manager that logs stage entry/exit and timing."""
    structlog.contextvars.bind_contextvars(
        correlation_id=correlation_id,
        stage=name,
        **context,
    )
    log = logger.bind(stage=name)
    log.info("stage_started")
    start = time.monotonic()
    try:
        yield log
        elapsed = time.monotonic() - start
        log.info("stage_completed", elapsed_seconds=round(elapsed, 3))
    except Exception as e:
        elapsed = time.monotonic() - start
        log.error("stage_failed", error=str(e), elapsed_seconds=round(elapsed, 3))
        raise
    finally:
        structlog.contextvars.unbind_contextvars("stage")

# Usage
def run_pipeline(run_id: str, source_ids: list[str]) -> None:
    with pipeline_stage("extract", correlation_id=run_id, source_count=len(source_ids)) as log:
        records = extract_all(source_ids)
        log.info("extracted", row_count=len(records))

    with pipeline_stage("transform", correlation_id=run_id, input_rows=len(records)) as log:
        clean, invalid = transform_with_validation(records)
        log.info("transformed", valid_rows=len(clean), invalid_rows=len(invalid))

    with pipeline_stage("load", correlation_id=run_id, input_rows=len(clean)) as log:
        loaded = load_to_warehouse(clean)
        log.info("loaded", rows_written=loaded)
```

### What to log at each stage

```python
# Entry: what are we processing?
log.info("stage_started", partition_id=partition_id, date=date, source=source)

# Business metrics: counts, sizes, rates
log.info("stage_completed",
    input_rows=input_count,
    output_rows=output_count,
    dropped_rows=invalid_count,
    elapsed_seconds=elapsed,
)

# Errors: full context, no stack trace swallowing
log.error("record_rejected",
    record_id=record["id"],
    error_type=type(e).__name__,
    error_message=str(e),
    stage="transform",
)
```

---

## DATA QUALITY CHECKS

Validate data quality at ingestion and between stages. Emit metrics on violations.

### Pydantic Validation at Ingestion

```python
from pydantic import BaseModel, field_validator
from dataclasses import dataclass

@dataclass
class ValidationResult:
    valid: list
    invalid: list
    error_counts: dict[str, int]

def validate_batch(records: list[dict], schema: type[BaseModel]) -> ValidationResult:
    valid, invalid, error_counts = [], [], {}

    for record in records:
        try:
            valid.append(schema(**record))
        except Exception as e:
            invalid.append({"record": record, "error": str(e)})
            error_type = type(e).__name__
            error_counts[error_type] = error_counts.get(error_type, 0) + 1

    return ValidationResult(valid=valid, invalid=invalid, error_counts=error_counts)

# After validation, always log quality metrics
def ingest_with_quality_check(records: list[dict]) -> list[OrderRecord]:
    result = validate_batch(records, OrderRecord)

    logger.info("ingestion_quality",
        total=len(records),
        valid=len(result.valid),
        invalid=len(result.invalid),
        error_breakdown=result.error_counts,
        error_rate=round(len(result.invalid) / max(len(records), 1), 4),
    )

    if result.invalid:
        for bad in result.invalid:
            dlq.send(bad)  # route to DLQ

    return result.valid
```

---

## OPENTELEMETRY TRACING

For distributed pipelines, use OpenTelemetry to trace execution across stages and services.

### Setup

```python
# pip install opentelemetry-sdk opentelemetry-exporter-otlp
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

def configure_tracing(service_name: str) -> None:
    provider = TracerProvider()
    exporter = OTLPSpanExporter(endpoint="http://otel-collector:4317")
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)

tracer = trace.get_tracer("data-pipeline")
```

### Instrument Pipeline Stages

```python
from opentelemetry import trace
from opentelemetry.trace import Status, StatusCode

tracer = trace.get_tracer("data-pipeline")

def extract(source_id: str) -> list[dict]:
    with tracer.start_as_current_span("extract") as span:
        span.set_attribute("source_id", source_id)
        try:
            records = fetch_from_api(source_id)
            span.set_attribute("row_count", len(records))
            span.set_status(Status(StatusCode.OK))
            return records
        except Exception as e:
            span.set_status(Status(StatusCode.ERROR, str(e)))
            span.record_exception(e)
            raise

def transform(records: list[dict]) -> list[dict]:
    with tracer.start_as_current_span("transform") as span:
        span.set_attribute("input_rows", len(records))
        result = [clean(r) for r in records]
        span.set_attribute("output_rows", len(result))
        return result
```

---

## PIPELINE METRICS

Export row counts, error rates, and stage latency as metrics (Prometheus or StatsD).

```python
# pip install prometheus-client
from prometheus_client import Counter, Histogram, Gauge

ROWS_PROCESSED = Counter("pipeline_rows_processed_total", "Rows processed", ["pipeline", "stage", "status"])
STAGE_LATENCY = Histogram("pipeline_stage_duration_seconds", "Stage duration", ["pipeline", "stage"])
ACTIVE_PIPELINES = Gauge("pipeline_active_runs", "Currently running pipelines", ["pipeline"])

def instrument_stage(pipeline_name: str, stage_name: str):
    def decorator(fn):
        def wrapper(*args, **kwargs):
            ACTIVE_PIPELINES.labels(pipeline=pipeline_name).inc()
            with STAGE_LATENCY.labels(pipeline=pipeline_name, stage=stage_name).time():
                try:
                    result = fn(*args, **kwargs)
                    ROWS_PROCESSED.labels(pipeline=pipeline_name, stage=stage_name, status="success").inc()
                    return result
                except Exception:
                    ROWS_PROCESSED.labels(pipeline=pipeline_name, stage=stage_name, status="error").inc()
                    raise
                finally:
                    ACTIVE_PIPELINES.labels(pipeline=pipeline_name).dec()
        return wrapper
    return decorator

@instrument_stage("daily-etl", "extract")
def extract(source_id: str) -> list[dict]: ...
```

---

## ALERTING RULES

Define alerts in your metrics system:

```yaml
# Prometheus alert rules
groups:
  - name: pipeline_alerts
    rules:
      - alert: PipelineStageFailed
        expr: increase(pipeline_rows_processed_total{status="error"}[5m]) > 100
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Pipeline {{ $labels.pipeline }}/{{ $labels.stage }} error rate elevated"

      - alert: PipelineStuck
        expr: pipeline_active_runs > 0 and pipeline_stage_duration_seconds_sum / pipeline_stage_duration_seconds_count > 3600
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Pipeline {{ $labels.pipeline }} stage running > 1 hour"

      - alert: DLQBacklogGrowing
        expr: increase(dlq_records_total[15m]) > 500
        labels:
          severity: warning
```

---

## CORRELATION ID PROPAGATION

Every record processed by a pipeline carries a correlation ID from ingestion to final write. This enables full traceability.

```python
import uuid

def generate_run_id() -> str:
    return str(uuid.uuid4())

# Attach run_id to every record at ingestion
def ingest(source_records: list[dict], run_id: str) -> list[dict]:
    return [{**r, "_run_id": run_id, "_ingested_at": utcnow().isoformat()} for r in source_records]

# Propagate through all stages — never strip _run_id
def transform(records: list[dict]) -> list[dict]:
    return [{**clean(r), "_run_id": r["_run_id"]} for r in records]
```

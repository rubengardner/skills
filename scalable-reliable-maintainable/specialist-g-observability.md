# Specialist G — Observability

## MONITORING vs OBSERVABILITY

**Monitoring:** Watch predefined metrics. Alert when thresholds breach. Answers: "Is it broken?"

**Observability:** Ability to ask arbitrary questions about a system's internal state from its external outputs — without redeploying or adding new instrumentation. Answers: "Why is it broken, and what's the exact path through the system that caused it?"

A system you cannot observe is a system you cannot safely operate or evolve.

---

## THE THREE PILLARS

### Pillar 1: Metrics

Numeric time-series measurements. Low storage cost. Fast queries. Best for alerting and trending.

**Types:**
- **Counter**: always increases (total requests, total errors, total events processed)
- **Gauge**: can go up or down (active connections, queue depth, current memory usage)
- **Histogram**: distribution of values (request latency percentiles — p50, p95, p99)

**The four golden signals (Google SRE Book):**
```
Latency    — how long requests take (distinguish slow errors from slow successes)
Traffic    — demand on the system (requests/sec, events/sec, queries/sec)
Errors     — rate of failed requests (explicit: 5xx; implicit: wrong content)
Saturation — how "full" your service is (CPU %, memory %, queue depth)
```

**Implementation (Prometheus + Python):**

```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server

REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status_code"],
)
REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds",
    "HTTP request latency",
    ["endpoint"],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
)
ACTIVE_REQUESTS = Gauge("http_requests_active", "Active HTTP requests")

# Middleware (Django/FastAPI)
import time

def metrics_middleware(request, get_response):
    ACTIVE_REQUESTS.inc()
    start = time.monotonic()
    response = get_response(request)
    elapsed = time.monotonic() - start

    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.path,
        status_code=response.status_code,
    ).inc()
    REQUEST_LATENCY.labels(endpoint=request.path).observe(elapsed)
    ACTIVE_REQUESTS.dec()
    return response
```

---

### Pillar 2: Structured Logs

Chronological records of events. Use JSON. Never plain text.

**Why structured (JSON) over plain text:**
- Machine-queryable: `filter(level="error", service="order-service")`
- Consistent fields: every log line has `timestamp`, `level`, `service`, `correlation_id`
- Log aggregators (Loki, Elasticsearch, Datadog) index JSON fields

```python
# pip install structlog
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
)

logger = structlog.get_logger()

# Every log line carries context automatically
def process_order(order_id: str, user_id: str) -> None:
    structlog.contextvars.bind_contextvars(
        correlation_id=generate_id(),
        order_id=order_id,
        user_id=user_id,
    )

    logger.info("order_processing_started")

    try:
        order = fetch_order(order_id)
        result = fulfill(order)
        logger.info("order_processing_completed", result=result, duration_ms=elapsed)
    except Exception as e:
        logger.error("order_processing_failed", error=str(e), error_type=type(e).__name__)
        raise
```

**What every log line must have:**
- `timestamp` (ISO 8601)
- `level` (debug/info/warning/error)
- `service` name
- `correlation_id` (trace through all related log lines)
- `message` (human-readable event name, not a sentence)
- Relevant context fields (IDs, counts, durations)

**What logs must never contain:** passwords, tokens, PII (names, emails, SSNs) unless explicitly required and protected.

---

### Pillar 3: Distributed Traces

A trace records how a single request flows through multiple services. Composed of "spans" — one per unit of work per service.

```
Trace: request abc-123
  Span: api-gateway            [0ms — 245ms]
    Span: order-service        [10ms — 220ms]
      Span: inventory-service  [15ms — 80ms]  ← bottleneck
      Span: payment-service    [90ms — 200ms]
      Span: postgres query     [205ms — 215ms]
```

Without traces, you know the request was slow. With traces, you know *which service and which operation* was slow.

**OpenTelemetry (the standard):**

```python
# pip install opentelemetry-sdk opentelemetry-exporter-otlp
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

def configure_tracing(service_name: str) -> None:
    provider = TracerProvider()
    provider.add_span_processor(
        BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4317"))
    )
    trace.set_tracer_provider(provider)

tracer = trace.get_tracer("order-service")

def process_payment(order_id: str, amount: float) -> dict:
    with tracer.start_as_current_span("process_payment") as span:
        span.set_attribute("order_id", order_id)
        span.set_attribute("amount", amount)
        try:
            result = payment_gateway.charge(order_id, amount)
            span.set_attribute("transaction_id", result["id"])
            return result
        except Exception as e:
            span.record_exception(e)
            span.set_status(trace.StatusCode.ERROR, str(e))
            raise
```

**Propagate trace context across service boundaries:**

```python
# Inject context into outbound HTTP calls
from opentelemetry.propagate import inject

headers = {}
inject(headers)  # adds traceparent header
response = httpx.post("http://inventory-service/reserve", headers=headers, json=payload)
```

---

## OBSERVABILITY STACK

| Tool | Role | Hosted option |
|------|------|--------------|
| **Prometheus** | Metrics collection and storage | Grafana Cloud |
| **Grafana** | Metrics and logs dashboards | Grafana Cloud |
| **Loki** | Log aggregation | Grafana Cloud |
| **Tempo / Jaeger** | Distributed tracing | Grafana Cloud / self-hosted |
| **OpenTelemetry** | Vendor-neutral instrumentation standard | (agents only) |
| **Datadog** | All-in-one commercial (metrics + logs + traces) | Datadog SaaS |
| **SigNoz** | Open-source all-in-one | Self-hosted |

**Recommendation for most teams:** OpenTelemetry for instrumentation (vendor-neutral), Grafana stack (Prometheus + Loki + Tempo) for storage and visualization, or Datadog if budget allows and operational simplicity matters.

---

## ALERTING

**Alert on symptoms, not causes.** "Error rate > 1%" is a symptom. "Redis connection pool full" is a cause. Page humans on symptoms (user impact). Use cause-based alerts only for automated self-healing.

**Alert on SLO burn rate** (see specialist-d-sre.md), not raw thresholds.

```yaml
# Prometheus alerting rules
groups:
  - name: slo_alerts
    rules:
      - alert: HighErrorBurnRate
        # 5-minute window burning at > 14.4× budget rate
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[5m]))
            /
            sum(rate(http_requests_total[5m]))
          ) > (1 - 0.999) * 14.4
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error budget burning at critical rate"

      - alert: HighP99Latency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 2.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "p99 latency above 2 seconds"
```

---

## THE CORRELATION ID PATTERN

A correlation ID ties together all log lines, metrics, and traces from a single request across all services.

```python
import uuid
from contextvars import ContextVar

correlation_id_var: ContextVar[str] = ContextVar("correlation_id", default="")

# Middleware: extract from header or generate
def correlation_middleware(request, get_response):
    cid = request.headers.get("X-Correlation-ID", str(uuid.uuid4()))
    correlation_id_var.set(cid)
    structlog.contextvars.bind_contextvars(correlation_id=cid)

    response = get_response(request)
    response["X-Correlation-ID"] = cid  # return to client for support debugging
    return response

# Outbound calls: propagate the same ID
def call_service(url: str, data: dict) -> dict:
    return httpx.post(url, json=data, headers={
        "X-Correlation-ID": correlation_id_var.get(),
    }).json()
```

With this pattern: one correlation ID in a bug report → find every log line, every span, every metric tag across every service for that request.

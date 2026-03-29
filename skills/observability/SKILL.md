---
name: observability
description: Observability patterns — Prometheus/Grafana, OpenTelemetry traces, SLO/SLI/SLA definitions, alerting rules, dashboards as code, and structured logging.
origin: custom
---

# Observability Patterns

Production observability: metrics, traces, logs, and alerting.

## When to Activate

- Setting up Prometheus + Grafana monitoring
- Instrumenting Python applications with OpenTelemetry
- Defining SLOs, SLIs, and error budgets
- Writing alerting rules and dashboards as code
- Debugging production incidents

## Three Pillars

| Pillar | Tool | Purpose |
|--------|------|---------|
| **Metrics** | Prometheus + Grafana | Quantitative system health over time |
| **Traces** | OpenTelemetry + Jaeger/Tempo | Request flow across services |
| **Logs** | Structured JSON + Loki / CloudWatch | Event details and context |

## Prometheus — Python Application Instrumentation

```python
# src/monitoring.py
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time
import functools
from typing import Callable

# Define metrics
REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status_code"],
)

REQUEST_DURATION = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration",
    ["method", "endpoint"],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0],
)

ACTIVE_REQUESTS = Gauge(
    "http_requests_active",
    "Active HTTP requests",
)

DB_QUERY_DURATION = Histogram(
    "db_query_duration_seconds",
    "Database query duration",
    ["query_type", "table"],
    buckets=[0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0],
)

MODEL_PREDICTION_LATENCY = Histogram(
    "ml_prediction_duration_seconds",
    "ML model prediction latency",
    ["model_name", "model_version"],
)
```

```python
# FastAPI middleware integration
from fastapi import FastAPI, Request
import time

app = FastAPI()

@app.middleware("http")
async def prometheus_middleware(request: Request, call_next: Callable):
    ACTIVE_REQUESTS.inc()
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start

    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.url.path,
        status_code=response.status_code,
    ).inc()

    REQUEST_DURATION.labels(
        method=request.method,
        endpoint=request.url.path,
    ).observe(duration)

    ACTIVE_REQUESTS.dec()
    return response


@app.get("/metrics")
async def metrics():
    from prometheus_client import generate_latest, CONTENT_TYPE_LATEST
    from fastapi.responses import Response
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)
```

## OpenTelemetry — Distributed Tracing (Python)

```python
# src/telemetry.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor
from opentelemetry.sdk.resources import Resource


def setup_telemetry(service_name: str, otlp_endpoint: str = "http://otel-collector:4317") -> None:
    resource = Resource.create({
        "service.name": service_name,
        "service.version": "1.2.0",
        "deployment.environment": "production",
    })

    provider = TracerProvider(resource=resource)
    exporter = OTLPSpanExporter(endpoint=otlp_endpoint, insecure=True)
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)

    # Auto-instrument frameworks
    FastAPIInstrumentor.instrument()
    SQLAlchemyInstrumentor().instrument()
    RedisInstrumentor().instrument()


# Manual span creation
tracer = trace.get_tracer(__name__)

async def process_order(order_id: str) -> dict:
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("order.source", "api")

        with tracer.start_as_current_span("validate_inventory"):
            result = await check_inventory(order_id)
            span.set_attribute("inventory.available", result["available"])

        with tracer.start_as_current_span("charge_payment"):
            payment = await charge(order_id)
            if not payment["success"]:
                span.record_exception(PaymentFailedError(order_id))
                span.set_status(trace.Status(trace.StatusCode.ERROR))

        return {"status": "processed"}
```

## SLO / SLI / SLA Definitions

```yaml
# slo-config.yaml — Pyrra / Sloth format
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: api-availability
  namespace: monitoring
spec:
  description: "API availability — 99.9% of requests succeed"
  target: "99.9"    # 99.9% = 43.8 min downtime/month budget
  window: 30d

  indicator:
    ratio:
      errors:
        metric: http_requests_total{status_code=~"5.."}
      total:
        metric: http_requests_total
---
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: api-latency
  namespace: monitoring
spec:
  description: "95% of requests complete in < 500ms"
  target: "95"
  window: 30d

  indicator:
    ratio:
      errors:
        metric: http_request_duration_seconds_bucket{le="0.5"}
      total:
        metric: http_request_duration_seconds_count
```

```python
# SLI calculation
def calculate_error_budget_remaining(
    total_requests: int,
    error_requests: int,
    slo_target: float = 0.999,
    window_days: int = 30,
) -> dict:
    """Calculate remaining error budget."""
    current_availability = 1 - (error_requests / total_requests)
    allowed_errors = total_requests * (1 - slo_target)
    remaining_budget = allowed_errors - error_requests
    burn_rate = error_requests / allowed_errors

    return {
        "availability": f"{current_availability * 100:.4f}%",
        "slo_target": f"{slo_target * 100:.1f}%",
        "error_budget_remaining": remaining_budget,
        "burn_rate": round(burn_rate, 2),
        "status": "ok" if current_availability >= slo_target else "breached",
    }
```

## Prometheus Alerting Rules

```yaml
# alerts/application.yaml
groups:
  - name: application
    rules:
      # Availability SLO — error rate > 0.1% (SLO burn)
      - alert: HighErrorRate
        expr: |
          (
            rate(http_requests_total{status_code=~"5.."}[5m])
            /
            rate(http_requests_total[5m])
          ) > 0.001
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "High error rate on {{ $labels.endpoint }}"
          description: "Error rate {{ $value | humanizePercentage }} > 0.1% SLO threshold"
          runbook: "https://wiki.myorg.com/runbooks/high-error-rate"

      # Latency SLO — p99 > 1s
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            rate(http_request_duration_seconds_bucket[5m])
          ) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "p99 latency {{ $value | humanizeDuration }} > 1s"

      # Pod restarts
      - alert: PodCrashLooping
        expr: |
          rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 3
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} is crash-looping"
          description: "{{ $value }} restarts in 15 minutes"

      # Error budget burn rate (fast burn — 1h)
      - alert: ErrorBudgetFastBurn
        expr: |
          (
            rate(http_requests_total{status_code=~"5.."}[1h])
            /
            rate(http_requests_total[1h])
          ) > 14.4 * 0.001    # 14.4x burn rate = 1h depletes 2% of monthly budget
        for: 2m
        labels:
          severity: critical
          page: "true"
        annotations:
          summary: "Fast error budget burn — paging on-call"
```

## Grafana Dashboard as Code (JSON)

```python
# scripts/create_dashboard.py — using grafonnet or grafanalib
from grafanalib.core import (
    Dashboard, Row, Target, TimeSeries,
    GaugePanel, Stat, RowPanel,
)

dashboard = Dashboard(
    title="Application SLO",
    uid="app-slo-v1",
    refresh="30s",
    time={"from": "now-1h", "to": "now"},
    panels=[
        Stat(
            title="Availability (30d)",
            targets=[Target(
                expr='1 - (sum(increase(http_requests_total{status_code=~"5.."}[30d])) / sum(increase(http_requests_total[30d])))',
                legendFormat="Availability",
            )],
            unit="percentunit",
            thresholds=[
                {"value": 0.999, "color": "green"},
                {"value": 0.99,  "color": "yellow"},
                {"value": 0,     "color": "red"},
            ],
        ),
        TimeSeries(
            title="Request Rate",
            targets=[
                Target(
                    expr='sum(rate(http_requests_total[5m])) by (endpoint)',
                    legendFormat="{{ endpoint }}",
                )
            ],
        ),
        TimeSeries(
            title="p50 / p95 / p99 Latency",
            targets=[
                Target(expr='histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))', legendFormat="p50"),
                Target(expr='histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))', legendFormat="p95"),
                Target(expr='histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))', legendFormat="p99"),
            ],
        ),
    ],
).auto_panel_ids()
```

## Structured Logging (Python)

```python
# src/logging_config.py
import logging
import json
import sys
from datetime import datetime, timezone


class JSONFormatter(logging.Formatter):
    """Structured JSON log formatter."""

    def format(self, record: logging.LogRecord) -> str:
        log_entry = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "level": record.levelname,
            "message": record.getMessage(),
            "logger": record.name,
            "module": record.module,
            "line": record.lineno,
        }
        # Include extra fields (trace_id, request_id, user_id, etc.)
        for key, value in record.__dict__.items():
            if key not in ("msg", "args", "levelname", "levelno", "name",
                           "pathname", "filename", "module", "exc_info",
                           "exc_text", "stack_info", "lineno", "funcName",
                           "created", "msecs", "relativeCreated", "thread",
                           "threadName", "processName", "process", "message"):
                log_entry[key] = value

        if record.exc_info:
            log_entry["exception"] = self.formatException(record.exc_info)

        return json.dumps(log_entry)


def setup_logging(level: str = "INFO") -> None:
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JSONFormatter())

    logging.basicConfig(level=level, handlers=[handler])
    logging.getLogger("uvicorn.access").handlers = [handler]


# Usage
import logging
logger = logging.getLogger(__name__)

logger.info(
    "Order processed",
    extra={
        "order_id": "ord-123",
        "user_id": "usr-456",
        "amount": 99.99,
        "trace_id": "abc123",
    },
)
```

## Prometheus Stack (Helm)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values values-monitoring.yaml \
  --version 58.2.2
```

```yaml
# values-monitoring.yaml
grafana:
  adminPassword: ${GRAFANA_PASSWORD}
  dashboardProviders:
    dashboardproviders.yaml:
      providers:
        - name: default
          folder: App
          type: file
          options:
            path: /var/lib/grafana/dashboards

prometheus:
  prometheusSpec:
    retention: 30d
    retentionSize: "50GB"
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          resources:
            requests:
              storage: 50Gi

alertmanager:
  config:
    receivers:
      - name: slack-critical
        slack_configs:
          - api_url: ${SLACK_WEBHOOK_URL}
            channel: "#alerts-critical"
            title: "{{ .GroupLabels.alertname }}"
            text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"
    route:
      receiver: slack-critical
      group_by: [alertname, severity]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
```

## Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| ❌ No metrics on ML models | Track prediction latency, error rate, input drift |
| ❌ Alerting on symptoms not causes | Alert on SLO burn rate, not CPU% |
| ❌ Alert fatigue | Only page on fast SLO burn; use low-urgency channels for slow burn |
| ❌ No runbooks linked in alerts | Add `runbook` annotation to every alert |
| ❌ Logs without structure | JSON formatter — always; never `print()` or unstructured strings |
| ❌ No distributed tracing | OpenTelemetry from day one — retrofitting is painful |
| ❌ Dashboards clicked manually | Dashboards as code (JSON in git) — reproducible and reviewed |


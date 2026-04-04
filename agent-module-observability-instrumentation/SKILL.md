---
name: agent-module-observability-instrumentation
description: "Module: Observability and Instrumentation Standard"
---
# Module: Observability and Instrumentation Standard

## 1. Module Metadata
- Module ID: `agent.module.observability-instrumentation`
- Version: `1.0.0`
- Maturity: `production`
- Scope: Unified logging, distributed tracing, metrics collection, and alerting strategy across all services.
- Primary outcomes:
  - Deterministic structured logging with correlation IDs across services.
  - Distributed tracing for request flow visibility.
  - Real-time metrics for operational and product visibility.
  - Actionable alerts tied to runbooks.

## 2. Mission and Applicability
Use this module to instrument all backend services, frontends, and async workers with production-grade observability.

Apply when:
- System runs in production and must be debugged remotely.
- Multiple services communicate asynchronously.
- On-call teams need to understand failures without access to logs.

Do not apply directly when:
- System is single-process, single-threaded development script.
- Observability is not required by compliance or operational policy.

## 3. Observability Pillars
All systems must instrument all four pillars:

### A. Structured Logging
**Purpose:** Record discrete events with context for search and debugging.
**Format:** JSON with consistent schema.
**Correlation:** All logs from one request carry same `correlation_id`.
**Retention:** Searchable for 90 days; archived for 2 years.

### B. Distributed Tracing
**Purpose:** Visualize request flow across services and components.
**Format:** OpenTelemetry spans with timing and parent-child relationships.
**Sampling:** 100% for errors and high-latency requests; 1% for normal requests.
**Retention:** 30 days searchable; 1 year archived.

### C. Metrics (Timeseries Data)
**Purpose:** Track operational and product KPIs at scale.
**Format:** Timeseries (e.g., Prometheus) with dimensions/tags.
**Cardinality:** Keep tag combinations < 10M to avoid metric explosion.
**Retention:** 1 month high-resolution; 1 year downsampled.

### D. Alerting
**Purpose:** Notify operators before users detect failures.
**Trigger:** Metrics, error rates, or log patterns.
**Severity:** SEV1 (page immediately), SEV2 (within 1 hour), SEV3 (next business day).
**Runbook:** Every alert has a linked runbook.

---

## 4. Structured Logging Contract

### Log Record Schema
Every log record must conform to this schema:

```json
{
  "timestamp": "2026-04-04T12:34:56.789Z",
  "level": "INFO",
  "logger": "service_name.module.component",
  "message": "User signup completed",
  "correlation_id": "req-abc123def456",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "service_name": "user-service",
  "service_version": "1.2.3",
  "environment": "production",
  "user_id": "user-xyz",
  "resource_id": "signup-session-001",
  "action": "user.signup",
  "status": "success",
  "duration_ms": 245,
  "error_code": null,
  "error_message": null,
  "context": {
    "email_domain": "gmail.com",
    "referrer": "[REDACTED]",
    "ip_address": "[REDACTED]"
  },
  "tags": {
    "customer_tier": "free",
    "region": "us-east-1"
  }
}
```

**Field definitions:**
- `timestamp` – ISO 8601 UTC timestamp (server-generated, not client).
- `level` – Log level: `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL`.
- `logger` – Hierarchical logger name (e.g., `service.module.component`).
- `message` – Human-readable event description (< 200 chars, actionable).
- `correlation_id` – Unique request ID (UUID or snowflake); same across all logs for one request.
- `trace_id` – OpenTelemetry W3C trace ID for distributed tracing.
- `span_id` – OpenTelemetry span ID for this component.
- `service_name` – Name of service emitting log.
- `service_version` – Semantic version of service.
- `environment` – Deployment environment (`development`, `staging`, `production`).
- `user_id` – User ID if request is authenticated (redact if PII policy requires).
- `resource_id` – ID of primary resource being operated on (e.g., session ID, order ID).
- `action` – Dot-separated action code (e.g., `user.signup`, `session.message.create`).
- `status` – Operation outcome: `success`, `failure`, `partial`.
- `duration_ms` – Elapsed time for synchronous operations.
- `error_code` – Machine-readable error code from error taxonomy (e.g., `VALIDATION_EMAIL_FORMAT`).
- `error_message` – Redacted error message safe for logs (no raw user input).
- `context` – Additional context as key-value pairs (redact sensitive fields).
- `tags` – Dimensional tags for metrics aggregation (low-cardinality only).

### Log Levels and Guidelines
| Level | When to Use | Example |
|---|---|---|
| `TRACE` | Extremely detailed debugging (disabled in prod) | `Entering validation loop iteration 3` |
| `DEBUG` | Detailed execution flow for developers | `Parsed email from payload: example@gmail.com` |
| `INFO` | Significant business events | `User signup completed`, `Payment processed` |
| `WARN` | Unexpected but recoverable condition | `Retry attempt 2 of 3`, `Fallback to cached data` |
| `ERROR` | Recoverable failure requiring action | `External API timeout`, `Validation failed` |
| `FATAL` | Unrecoverable failure; service stopping | `Database connection lost permanently` |

**Principle:** INFO is the default; DEBUG and TRACE are for development. WARN+ should be actionable and resolvable.

---

## 5. Distributed Tracing Contract

### Trace Structure
Every request generates one trace spanning multiple services:

```
Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736

Span: nginx ingress (entry)
├─ Span: user-service (authenticate)
├─ Span: user-service (process signup)
│  ├─ Span: postgres (execute query)
│  └─ Span: email-service (send confirmation)
│     └─ Span: smtp provider (deliver)
└─ Span: nginx (return response)
```

### Required Span Attributes
| Attribute | Example | Purpose |
|---|---|---|
| `trace_id` | `4bf92f3577b34da6a3ce929d0e0e4736` | Unique request ID across services |
| `span_id` | `00f067aa0ba902b7` | Unique span within trace |
| `parent_span_id` | `f9d56ff0ad6f2c5e` | Parent span ID (links span hierarchy) |
| `operation_name` | `user-service.signup` | Service and operation name |
| `service_name` | `user-service` | Service emitting span |
| `duration_ms` | `245` | Elapsed time |
| `status` | `ok`, `error` | Span outcome |
| `error_code` | `EXTERNAL_SERVICE_TIMEOUT` | Machine code if error |
| `http.method` | `POST` | HTTP method (for HTTP spans) |
| `http.url` | `/api/v1/users/signup` | HTTP endpoint |
| `http.status_code` | `201` | HTTP response status |
| `db.operation` | `INSERT` | Database operation type |
| `db.statement` | `INSERT INTO users ...` | SQL query (redact sensitive values) |

### Trace Context Propagation
Every outgoing request must include W3C Trace Context headers:

```http
POST /api/v1/users/signup HTTP/1.1
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-f9d56ff0ad6f2c5e-01
tracestate: user-service=1
correlation-id: req-abc123def456
```

**Principle:** Trace context flows automatically through HTTP, message queues, and async calls.

---

## 6. Metrics Collection Standard

### RED Metrics (Request-based Services)
For every service endpoint, collect:

| Metric | Type | Dimensions | Example |
|---|---|---|---|
| `requests_total` | Counter | service, endpoint, method, status | `user_service_requests_total{endpoint="/signup", status="201"}` |
| `requests_duration_ms` | Histogram | service, endpoint, status | `user_service_requests_duration_ms{endpoint="/signup", quantile="0.95"}` |
| `requests_errors_total` | Counter | service, endpoint, error_code | `user_service_requests_errors_total{endpoint="/signup", error_code="VALIDATION_EMAIL_FORMAT"}` |

### USE Metrics (Resource-based Services)
For databases, caches, and infrastructure:

| Metric | Type | Dimensions | Example |
|---|---|---|---|
| `resource_utilization` | Gauge | resource, type | `postgres_db_utilization_percent{resource="connections"}` |
| `resource_saturation` | Gauge | resource, type | `redis_cache_evictions_total` |
| `resource_errors` | Counter | resource, operation | `postgres_query_errors_total{operation="INSERT"}` |

### Business Metrics
Track domain-specific KPIs:

| Metric | Type | Dimensions | Example |
|---|---|---|---|
| `signups_total` | Counter | referrer, plan_tier | `signups_total{referrer="google", plan_tier="free"}` |
| `conversion_rate` | Gauge | funnel_stage | `conversion_rate{stage="signup_to_payment"}` |
| `active_sessions` | Gauge | region, client_type | `active_sessions{region="us-east-1", client_type="web"}` |

### Metric Cardinality Control
**Rule:** Unique combinations of tag values across all metrics must stay < 10M.

**Bad (high cardinality):**
```
requests_total{endpoint="/users/{id}", user_id="..."}  # user_id = millions of values
```

**Good (controlled cardinality):**
```
requests_total{endpoint="/users/{id}", user_tier="free|paid"}  # user_tier = 2 values
```

---

## 7. Implementation Workflow

### Phase A: Logging Infrastructure and SDK
1. Choose logging library (Python: `structlog` or `python-json-logger`, Node: `winston` or `bunyan`).
2. Configure JSON output with schema validation.
3. Implement automatic correlation ID injection (middleware or context-local storage).
4. Add redaction rules for sensitive fields (passwords, tokens, PII).

Example (Python with structlog):
```python
import structlog
from uuid import uuid4
from middleware import get_correlation_id

structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

# Middleware that adds correlation ID
def logging_middleware(request, call_next):
    correlation_id = request.headers.get("X-Correlation-ID", str(uuid4()))
    request.state.correlation_id = correlation_id
    
    # Add to all logs from this request
    with structlog.contextvars.bound_contextvars(
        correlation_id=correlation_id,
        service_name="user-service",
        service_version="1.2.3",
        environment="production"
    ):
        response = call_next(request)
    
    response.headers["X-Correlation-ID"] = correlation_id
    return response
```

Exit criteria:
- All logs are JSON and contain correlation ID.
- Sensitive fields are redacted.

### Phase B: Distributed Tracing Instrumentation
1. Choose tracing library (OpenTelemetry).
2. Instrument HTTP clients, database queries, and async calls.
3. Propagate trace context through all service boundaries.
4. Configure sampler (100% for errors, 1% for normal requests).

Example:
```python
from opentelemetry import trace, metrics
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

jaeger_exporter = JaegerExporter(agent_host_name="localhost", agent_port=6831)
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(BatchSpanProcessor(jaeger_exporter))

tracer = trace.get_tracer(__name__)

# Automatic instrumentation
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

FastAPIInstrumentor.instrument_app(app)
SQLAlchemyInstrumentor().instrument()

# Manual span creation
with tracer.start_as_current_span("user.signup") as span:
    span.set_attribute("user_id", user_id)
    span.set_attribute("email_domain", email.split("@")[1])
    result = create_user(user_data)
```

Exit criteria:
- Request flow visible in tracing UI (Jaeger, Zipkin).
- Trace context propagates to all dependent services.

### Phase C: Metrics Collection and Exposition
1. Choose metrics library (Prometheus client for Python).
2. Instrument critical paths with RED metrics.
3. Expose metrics on `/metrics` endpoint.
4. Configure scrape frequency (default: every 15 seconds).

Example:
```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server

# RED metrics
requests_total = Counter(
    "requests_total",
    "Total requests",
    ["service", "endpoint", "method", "status"]
)

requests_duration_ms = Histogram(
    "requests_duration_ms",
    "Request duration in ms",
    ["service", "endpoint", "status"],
    buckets=(10, 50, 100, 250, 500, 1000, 2500, 5000)
)

requests_errors = Counter(
    "requests_errors_total",
    "Total request errors",
    ["service", "endpoint", "error_code"]
)

# Business metrics
signups_total = Counter(
    "signups_total",
    "Total signups",
    ["referrer", "plan_tier"]
)

# Middleware to populate metrics
def metrics_middleware(request, call_next):
    start_time = time.time()
    response = call_next(request)
    duration = (time.time() - start_time) * 1000
    
    requests_total.labels(
        service="user-service",
        endpoint=request.url.path,
        method=request.method,
        status=response.status_code
    ).inc()
    
    requests_duration_ms.labels(
        service="user-service",
        endpoint=request.url.path,
        status=response.status_code
    ).observe(duration)
    
    return response

# Expose metrics
start_http_server(8001)  # /metrics available at :8001/metrics
```

Exit criteria:
- Key endpoints emit RED metrics.
- Metrics are queryable in Prometheus/Grafana.

### Phase D: Alerting Rules and Runbooks
1. Define alert conditions based on metrics and log patterns.
2. Classify alerts by severity (SEV1, SEV2, SEV3).
3. Link every alert to a runbook.
4. Configure notification channels (Slack, PagerDuty, email).

Example alert rules (Prometheus):
```yaml
groups:
  - name: user-service
    interval: 30s
    rules:
      # Error rate > 5% for 5 minutes
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(requests_errors_total{service="user-service"}[5m]))
            /
            sum(rate(requests_total{service="user-service"}[5m]))
          ) > 0.05
        for: 5m
        labels:
          severity: SEV1
        annotations:
          summary: "User service error rate > 5%"
          runbook: "https://wiki.company.com/runbooks/user-service-high-error-rate"
      
      # Response time p95 > 500ms
      - alert: SlowResponseTime
        expr: histogram_quantile(0.95, requests_duration_ms{service="user-service"}) > 500
        for: 5m
        labels:
          severity: SEV2
        annotations:
          summary: "User service p95 latency > 500ms"
          runbook: "https://wiki.company.com/runbooks/user-service-slow-latency"
      
      # Database connection pool utilization > 80%
      - alert: DatabaseConnectionPoolExhaustion
        expr: postgres_connections_usage{service="user-service"} > 0.8
        for: 2m
        labels:
          severity: SEV1
        annotations:
          summary: "Database connection pool utilization > 80%"
          runbook: "https://wiki.company.com/runbooks/db-connection-pool-exhaustion"
```

Runbook example:
```markdown
# High Error Rate in User Service

## Severity: SEV1 (Page immediately)

## Symptoms
- Error rate spike in requests_errors_total
- Users unable to sign up or login

## Diagnosis
1. Check `/api/v1/users/signup` error rate trend
2. Look for common error codes:
   - `VALIDATION_*` – Input validation errors (user issue)
   - `EXTERNAL_SERVICE_TIMEOUT` – Third-party API down
   - `INTERNAL_DATABASE_ERROR` – Database issue
3. Correlate error spike with recent deployments or infrastructure changes

## Immediate Action
1. Check on-call status page for known incidents
2. Query logs: `grep "ERROR" logs | tail -100`
3. If database-related: Page database team
4. If external API: Check their status page
5. If deployment-related: Consider rollback

## Resolution Steps
- [If validation errors] Check input validation rules in code
- [If external service] Wait for service recovery; monitor recovery
- [If database] Page database team; check connection pool saturation
- Verify fix: Error rate < 1% for 10 minutes
```

Exit criteria:
- All critical alerts have runbooks.
- Alerts are tuned to minimize false positives.

### Phase E: Dashboard and Observability Culture
1. Create dashboards for service health (RED metrics).
2. Create dashboards for business KPIs.
3. Link dashboards in runbooks and on-call wiki.
4. Schedule regular reviews of alerts and dashboards.

Example dashboard:
```
User Service Health Dashboard

[Request Rate]  [Error Rate]  [p95 Latency]
  5.2k/sec       0.3%           245ms

Requests by Endpoint
 /signup    3.1k/sec
 /login     1.8k/sec
 /verify    0.3k/sec

Error Rate by Code
 VALIDATION_EMAIL_FORMAT  40%
 EXTERNAL_SERVICE_TIMEOUT  35%
 CONFLICT_DUPLICATE_EMAIL  25%

Database Performance
 Connection Usage  45%
 Query Latency p95  120ms
 Slow Queries     2 in 24h
```

Exit criteria:
- On-call team can understand system health at a glance.

---

## 8. Decision Framework

| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| Log format | JSON with structured schema | Plain text | Use JSON for parseability and searchability. |
| Correlation ID | UUID in request header | Implicit from trace ID | Use explicit correlation ID for non-tracing use cases. |
| Trace sampling | 100% errors + 1% normal | 100% all | Use selective sampling to control cost. |
| Metrics library | Prometheus client | StatsD | Use Prometheus for rich dimensionality. |
| Alert severity | SEV1 (page now), SEV2 (1h), SEV3 (next day) | Binary (page/no-page) | Use tiered severity for proportional response. |
| Dashboard storage | Version-controlled as code | UI-only | Store as code (Jsonnet, YAML) for reproducibility. |

---

## 9. Validation Strategy

### Logging Validation
- Unit tests verify correlation IDs are present in logs.
- Integration tests verify trace context flows across services.
- Audit logs for sensitive data leakage.

### Tracing Validation
- Request flow visible in tracing UI.
- Span timing matches wall-clock time within 10%.
- No orphan spans (all spans belong to a trace).

### Metrics Validation
- Metrics appear in Prometheus within 15 seconds of emission.
- Cardinality < 10M unique tag combinations.
- No label explosion from unbounded values.

### Alert Validation
- All alerts have linked runbooks.
- Alert false positive rate < 5%.
- Mean time to detection (MTD) for real incidents < 2 minutes.

---

## 10. Benchmarks and SLO Targets
- Log ingestion latency p99: `<= 5 seconds`
- Trace ingestion latency p99: `<= 2 seconds`
- Metrics scrape success rate: `>= 99.9%`
- Log searchability latency (typical query): `<= 30 seconds`
- Alert notification latency (alert fire to notification): `<= 1 minute`
- Alert false positive rate: `<= 5%`
- Observability system availability: `>= 99%`

---

## 11. Risks and Controls
- Risk: Log volume explodes; storage costs become unaffordable.
  - Control: Set log level to INFO in production; TRACE/DEBUG only in development.
- Risk: Sensitive data (passwords, tokens) leaked in logs.
  - Control: Redaction filters; regular audits for PII in logs.
- Risk: Trace sampling misses important slow requests.
  - Control: Adaptive sampling; always 100% for errors and slow requests.
- Risk: Metrics cardinality explosion from unbounded tags.
  - Control: Whitelist tag values; enforce < 100 unique values per tag.
- Risk: Alerts are noisy; team ignores them (alert fatigue).
  - Control: Tune thresholds based on historical data; require runbook before alerting.
- Risk: On-call team can't interpret dashboards or find logs.
  - Control: Train team on observability tools; maintain up-to-date runbooks.

---

## 12. Agent Execution Checklist
- [ ] Logging library configured with JSON output and schema validation.
- [ ] Correlation ID injection middleware in place.
- [ ] Redaction rules for sensitive fields implemented.
- [ ] Distributed tracing library (OpenTelemetry) integrated.
- [ ] Trace context propagation across services implemented.
- [ ] HTTP, database, and async call instrumentation in place.
- [ ] RED metrics instrumented for all endpoints.
- [ ] Metrics endpoint (`/metrics`) exposed and scraped.
- [ ] Prometheus scrape configuration deployed.
- [ ] Alert rules defined with runbooks for each alert.
- [ ] Notification channels (Slack, PagerDuty) configured.
- [ ] Dashboards created for service health and business KPIs.
- [ ] On-call team trained on observability tools.
- [ ] Log retention and archival policies implemented.

---

## 13. Reuse Notes
This module is reusable across all backend services and applies to infrastructure as well. Adapt only:
- Log levels and verbosity by environment.
- Trace sampling rate by traffic profile and cost constraints.
- Metrics and alert thresholds by service criticality and SLO.
- Runbook links and escalation procedures by organizational structure.
- Dashboard layouts by team needs and focus areas.

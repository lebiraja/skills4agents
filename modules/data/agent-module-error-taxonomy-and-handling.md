# Module: Error Taxonomy and Unified Handling Standard

## 1. Module Metadata
- Module ID: `agent.module.error-taxonomy-and-handling`
- Version: `1.0.0`
- Maturity: `production`
- Scope: Standardized error code registry, classification, client-facing messaging, and retry semantics across all backend services.
- Primary outcomes:
  - Deterministic, reusable error codes across domains.
  - Clear distinction between client recoverable (transient) and permanent failures.
  - Consistent HTTP status codes and structured error responses.
  - Operator-actionable error context without leaking sensitive data.

## 2. Mission and Applicability
Use this module to standardize error handling across all backend services, APIs, and async workflows.

Apply when:
- System returns error responses to clients or logs failures to operators.
- Multiple services need to communicate errors consistently.
- Clients must decide retry vs fallback vs user notification based on error signals.

Do not apply directly when:
- System is single-service with no cross-service error contracts.
- Error handling is not user-facing (internal telemetry only).

## 3. Error Classification Framework
All errors fall into four mutually exclusive categories:

### A. Client Errors (HTTP 4xx)
**Cause:** Client request is malformed, unauthorized, or violates constraints.
**Client action:** Fix request or escalate to operator; retrying without change is futile.
**HTTP range:** `400–499`

**Subcategories:**
1. **Request Validation** (`400`) – Malformed payload, missing required fields, invalid formats.
2. **Authentication Failure** (`401`) – Missing, expired, or invalid credentials.
3. **Authorization Failure** (`403`) – Credentials valid but insufficient permissions.
4. **Resource Not Found** (`404`) – Resource does not exist or access denied.
5. **Conflict/Constraint Violation** (`409`) – Business rule violation, duplicate, or state conflict.

### B. Server Errors – Transient (HTTP 5xx, Retryable)
**Cause:** Temporary infrastructure, dependency, or resource exhaustion.
**Client action:** Retry with exponential backoff; likely to succeed on retry.
**HTTP range:** `500–599` (specific codes vary)

**Subcategories:**
1. **Service Unavailable** (`503`) – Dependency (DB, cache, external service) temporarily down.
2. **Timeout** (`504`) – Request exceeded deadline; likely transient.
3. **Rate Limited** (`429`) – Client quota/rate limit exceeded; back off and retry.

### C. Server Errors – Permanent (HTTP 5xx, Non-Retryable)
**Cause:** Bug, configuration error, or permanent data inconsistency.
**Client action:** Do not retry; escalate to operator immediately.
**HTTP range:** `500` (with `Retry-After: never` or explicit non-retryable marker)

**Subcategories:**
1. **Internal Server Error** (`500`) – Unhandled exception, invariant violation, or data corruption.
2. **Not Implemented** (`501`) – Feature not available (API version mismatch, incomplete deployment).

### D. Async/Background Job Errors
**Cause:** Job execution failure in queue/worker context.
**Retry behavior:** Transient failures enter backoff queue; permanent failures escalate or dead-letter.
**Status codes:** Job-specific status enum (not HTTP codes).

---

## 4. Standard Error Code Taxonomy
Every error must have a machine-readable code following the pattern:

```
<DOMAIN>_<ERROR_TYPE>_<SPECIFIC_REASON>
```

Examples:
- `AUTH_INVALID_TOKEN` – Token format or signature invalid
- `AUTH_TOKEN_EXPIRED` – Token lifetime exceeded
- `AUTH_INSUFFICIENT_PERMISSIONS` – Valid token lacks required scope
- `VALIDATION_EMAIL_FORMAT` – Email field failed format check
- `VALIDATION_PHONE_E164` – Phone field failed E.164 validation
- `RESOURCE_NOT_FOUND` – Requested resource ID does not exist
- `CONFLICT_DUPLICATE_EMAIL` – Email already registered
- `CONFLICT_STATE_TRANSITION` – State change violates workflow rules
- `EXTERNAL_SERVICE_TIMEOUT` – Third-party API did not respond in time
- `EXTERNAL_SERVICE_UNAVAILABLE` – Third-party service is down
- `RATE_LIMITED_BURST` – Client exceeded request rate
- `INTERNAL_DATABASE_ERROR` – Database connection or query failure (permanent)
- `INTERNAL_ASSERTION_FAILED` – Code invariant violated (permanent)

**Domain prefixes (extensible):**
- `AUTH` – Authentication/authorization
- `VALIDATION` – Input validation
- `RESOURCE` – Resource lifecycle and lookup
- `CONFLICT` – Business constraint and state machine violations
- `EXTERNAL` – Third-party service failures
- `RATE_LIMIT` – Quota/throttling
- `INTERNAL` – Server-side bugs and invariants

---

## 5. Retry-Ability Classification
Every error code must be classified:

| Error Code | Category | HTTP Status | Retryable | Backoff Strategy |
|---|---|---|---|---|
| `AUTH_INVALID_TOKEN` | Client | 401 | No | None |
| `AUTH_TOKEN_EXPIRED` | Client | 401 | No | None |
| `VALIDATION_*` | Client | 400 | No | None |
| `RESOURCE_NOT_FOUND` | Client | 404 | No | None |
| `CONFLICT_*` | Client | 409 | No | None |
| `RATE_LIMITED_BURST` | Transient | 429 | Yes | Exponential with `Retry-After` |
| `EXTERNAL_SERVICE_TIMEOUT` | Transient | 504 | Yes | Exponential (3 attempts, 1s–10s) |
| `EXTERNAL_SERVICE_UNAVAILABLE` | Transient | 503 | Yes | Exponential with `Retry-After` |
| `INTERNAL_DATABASE_ERROR` | Permanent | 500 | No | Alert operator |
| `INTERNAL_ASSERTION_FAILED` | Permanent | 500 | No | Alert operator |

---

## 6. Structured Error Response Contract

### HTTP API Error Response
All 4xx and 5xx responses must conform to this structure:

```json
{
  "error": {
    "code": "VALIDATION_EMAIL_FORMAT",
    "message": "Email address is invalid.",
    "http_status": 400,
    "retryable": false,
    "timestamp": "2026-04-04T12:34:56Z",
    "correlation_id": "req-abc123def456",
    "fields": [
      {
        "field_name": "email",
        "rejected_value": "[REDACTED]",
        "constraint_violated": "format"
      }
    ],
    "context": {
      "endpoint": "/api/v1/users",
      "user_id": "user-xyz"
    }
  }
}
```

**Field definitions:**
- `code` – Machine-readable error code (always present).
- `message` – Human-readable message safe for client display (no PII).
- `http_status` – HTTP status code (redundant with header but explicit in body).
- `retryable` – Boolean; client uses to decide retry logic.
- `timestamp` – ISO 8601 server timestamp for ordering.
- `correlation_id` – Unique request ID for log correlation.
- `fields` – Array of validation errors (present only for 400/422 validation failures).
  - `field_name` – Name of field that failed validation.
  - `rejected_value` – Sanitized/redacted value (never raw user input if sensitive).
  - `constraint_violated` – Machine code for which constraint failed (e.g., `format`, `required`, `uniqueness`).
- `context` – Optional operator context (endpoint, user_id, resource_id) for debugging; redacted in external-facing responses.

### Response Headers
```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Correlation-ID: req-abc123def456
Retry-After: <not present for 400>

HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Correlation-ID: req-abc123def456
Retry-After: 60

HTTP/1.1 503 Service Unavailable
Content-Type: application/json
Correlation-ID: req-abc123def456
Retry-After: 30
```

---

## 7. Implementation Workflow

### Phase A: Error Code Registry and Domain Definition
1. Define domain and error subcategories for your system.
2. Create centralized error code enum/registry (reusable across services).
3. Pair each code with HTTP status, retryable flag, and operator runbook reference.
4. Version the registry; breaking changes require versioning.

Example implementation (Python):
```python
from enum import Enum
from dataclasses import dataclass

@dataclass
class ErrorDefinition:
    code: str
    message: str
    http_status: int
    retryable: bool
    retry_backoff: str  # "exponential", "fixed", or "none"

class ErrorRegistry:
    VALIDATION_EMAIL_FORMAT = ErrorDefinition(
        code="VALIDATION_EMAIL_FORMAT",
        message="Email address is invalid.",
        http_status=400,
        retryable=False,
        retry_backoff="none"
    )
    
    AUTH_TOKEN_EXPIRED = ErrorDefinition(
        code="AUTH_TOKEN_EXPIRED",
        message="Session expired. Please log in again.",
        http_status=401,
        retryable=False,
        retry_backoff="none"
    )
    
    EXTERNAL_SERVICE_TIMEOUT = ErrorDefinition(
        code="EXTERNAL_SERVICE_TIMEOUT",
        message="Request timeout. Please try again.",
        http_status=504,
        retryable=True,
        retry_backoff="exponential"
    )
```

Exit criteria:
- Centralized error registry exists in shared library.
- Every domain service imports and uses registry.

### Phase B: Error Response Builder and Serialization
1. Create `ErrorResponse` class that constructs structured error payloads.
2. Implement sanitization/redaction of sensitive fields.
3. Attach correlation IDs and timestamps automatically.
4. Expose as HTTP middleware and exception handler.

Example:
```python
from dataclasses import dataclass, asdict
from uuid import uuid4
from datetime import datetime

@dataclass
class FieldError:
    field_name: str
    rejected_value: str  # sanitized
    constraint_violated: str

@dataclass
class ErrorResponse:
    code: str
    message: str
    http_status: int
    retryable: bool
    correlation_id: str
    timestamp: str
    fields: list[FieldError] = None
    context: dict = None
    
    @staticmethod
    def from_registry(definition: ErrorDefinition, fields: list[FieldError] = None, context: dict = None):
        return ErrorResponse(
            code=definition.code,
            message=definition.message,
            http_status=definition.http_status,
            retryable=definition.retryable,
            correlation_id=str(uuid4()),
            timestamp=datetime.utcnow().isoformat() + "Z",
            fields=fields,
            context=context
        )
    
    def to_dict(self, include_context: bool = False):
        data = asdict(self)
        if not include_context:
            data.pop("context", None)
        return {"error": data}
```

Exit criteria:
- All error responses use builder; no ad-hoc error payloads.
- Correlation IDs flow through logs and responses.

### Phase C: Exception Hierarchy and Handlers
1. Define application exception classes that inherit from `ApplicationError`.
2. Map exceptions to error codes and HTTP status.
3. Implement request/response middleware to catch and format exceptions.
4. Log exceptions with full context; sanitize messages before response.

Example:
```python
class ApplicationError(Exception):
    def __init__(self, error_def: ErrorDefinition, fields: list[FieldError] = None, context: dict = None):
        self.error_def = error_def
        self.fields = fields
        self.context = context
        super().__init__(error_def.message)

class ValidationError(ApplicationError):
    pass

class AuthenticationError(ApplicationError):
    pass

class ExternalServiceError(ApplicationError):
    pass

# FastAPI exception handler
@app.exception_handler(ApplicationError)
async def application_error_handler(request: Request, exc: ApplicationError):
    # Log with full context
    logger.error("Application error", extra={
        "error_code": exc.error_def.code,
        "path": request.url.path,
        "fields": exc.fields,
        "context": exc.context
    })
    
    response = ErrorResponse.from_registry(
        exc.error_def,
        fields=exc.fields,
        context=exc.context if is_internal_request(request) else None
    )
    
    return JSONResponse(
        status_code=exc.error_def.http_status,
        content=response.to_dict()
    )
```

Exit criteria:
- All exceptions produce structured error responses.
- Operator context (stack traces, full errors) stays in logs, not responses.

### Phase D: Retry Logic and Client Resilience
1. Implement exponential backoff for retryable errors.
2. Parse `Retry-After` header and respect it.
3. Classify errors in client SDKs for automatic retry eligibility.
4. Define max retry attempts and timeout budgets per endpoint.

Example retry logic:
```python
import asyncio
from typing import Callable, TypeVar

T = TypeVar('T')

async def retry_with_backoff(
    func: Callable[[], T],
    max_attempts: int = 3,
    initial_delay: float = 1.0,
    max_delay: float = 10.0,
    jitter: bool = True
) -> T:
    for attempt in range(max_attempts):
        try:
            return await func()
        except ApplicationError as e:
            if not e.error_def.retryable or attempt == max_attempts - 1:
                raise
            
            delay = min(initial_delay * (2 ** attempt), max_delay)
            if jitter:
                delay *= random.uniform(0.5, 1.0)
            
            logger.warning(f"Retry attempt {attempt + 1}/{max_attempts} after {delay}s", extra={
                "error_code": e.error_def.code,
                "attempt": attempt + 1
            })
            
            await asyncio.sleep(delay)
```

Exit criteria:
- Retryable failures are auto-retried with exponential backoff.
- Non-retryable failures fail fast.

### Phase E: Error Taxonomy Documentation and Runbooks
1. Maintain a public error code reference document.
2. Link each error code to an operator runbook.
3. Include common causes, detection signals, and remediation steps.
4. Keep runbook links in error definitions.

Example runbook structure:
```markdown
## Error: EXTERNAL_SERVICE_TIMEOUT

**Code:** `EXTERNAL_SERVICE_TIMEOUT`
**HTTP Status:** `504`
**Retryable:** Yes (exponential backoff, 3 attempts)

### Common Causes
- Third-party API service degradation or outage
- High network latency
- Client request timeout too tight for actual service latency

### Detection and Alerting
- Alert on 429 and 504 error rates > 1% for 5 minutes
- Check external service status page
- Correlate with network latency metrics

### Mitigation
1. Check third-party service status page
2. Increase request timeout temporarily (if safe)
3. Route to fallback service or graceful degradation
4. Page on-call if outage > 15 minutes

### Resolution Steps
- Wait for external service recovery (typical 5–30 min)
- Validate integrations after service is back up
- Run post-incident review if recurrence pattern detected
```

Exit criteria:
- All error codes have documented runbooks.
- Team can troubleshoot errors from code + runbook.

---

## 8. Decision Framework

| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| Error classification | Taxonomy with retryable flag | Enum with HTTP status only | Use taxonomy for client resilience and operator clarity. |
| Error response format | Structured JSON with code + message | Plain text message only | Use structured format for client parsing and logging. |
| Retry strategy | Exponential backoff with jitter | Fixed backoff or no backoff | Use exponential to avoid thundering herd. |
| Sensitive field exposure | Redacted/sanitized in response | Raw value in response | Always redact; server logs retain full context. |
| Operator context visibility | Include in logs, exclude from API response | Include in all responses | Prevent information leakage to untrusted clients. |

---

## 9. Validation Strategy

### Functional Validation
- Unit tests for each error code definition (code, message, HTTP status, retryable flag).
- Contract tests verifying error response structure matches schema.
- Validation tests ensuring all application exceptions map to registry codes.

### Failure Validation
- Tests for transient error retry behavior (exponential backoff, max attempts).
- Tests for non-retryable errors failing fast without retry.
- Tests for `Retry-After` header parsing and respect.
- Tests for field error arrays in 400/422 responses.

### Security/Observability Validation
- Verify no sensitive data (passwords, tokens, full email addresses, IPs) leak into error messages or responses.
- Verify correlation IDs are unique and flow end-to-end (logs, API responses).
- Verify operator context (full exceptions, stack traces) is logged but never exposed to external clients.
- Fuzz error response payloads to ensure they never crash clients.

---

## 10. Benchmarks and SLO Targets
- Error response schema compliance: `100%` for all 4xx/5xx responses
- Error code registry coverage: `100%` of application exceptions
- Correlation ID propagation completeness: `>= 99.9%` of requests
- Sensitive data leakage incidents: `0` per quarter
- Mean time to operator runbook from error log: `<= 2 minutes`
- Client retry behavior correctness on transient errors: `100%` in integration tests

---

## 11. Risks and Controls
- Risk: Developers create ad-hoc errors without registry.
  - Control: Linter rule and code review gate requiring registry usage.
- Risk: Error messages expose sensitive data.
  - Control: Automated secret scanning in CI and runtime error audit sampling.
- Risk: Clients ignore `retryable` flag and retry non-retryable errors.
  - Control: Explicit HTTP 4xx (no retry) vs 5xx (maybe retry) convention; clear documentation.
- Risk: Retry storms and cascading failures.
  - Control: Exponential backoff with jitter, max attempt ceiling, circuit breaker on persistent failures.
- Risk: Operators cannot debug errors from response alone.
  - Control: Correlation IDs and full logs; runbooks link to log query examples.

---

## 12. Agent Execution Checklist
- [ ] Error taxonomy and domain structure defined.
- [ ] Error code registry implemented and shared across services.
- [ ] HTTP status mapping and retryable classification complete.
- [ ] Structured error response builder implemented.
- [ ] Exception hierarchy and handlers in place.
- [ ] Retry logic with exponential backoff implemented for transient errors.
- [ ] Error codes documented with operator runbooks.
- [ ] Contract tests verify error response schema compliance.
- [ ] Security tests verify no sensitive data leakage.
- [ ] Correlation IDs instrumented end-to-end.
- [ ] CI enforces registry usage (no ad-hoc errors).

---

## 13. Reuse Notes
This module is reusable across all backend services. Adapt only:
- Domain-specific error codes and messages.
- Retry backoff parameters (initial delay, max delay) by endpoint criticality.
- Operator runbook links and escalation procedures.
- Sensitive field redaction rules by data classification.

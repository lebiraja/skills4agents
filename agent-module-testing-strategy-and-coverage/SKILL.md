---
name: agent-module-testing-strategy-and-coverage
description: "Module: Testing Strategy and Coverage Standard"
---
# Module: Testing Strategy and Coverage Standard

## 1. Module Metadata
- Module ID: `agent.module.testing-strategy-and-coverage`
- Version: `1.0.0`
- Maturity: `production`
- Scope: Unified testing pyramid, coverage targets, test categorization, quality gates, and CI/CD enforcement across all services.
- Primary outcomes:
  - Consistent test categorization and execution strategy across layers.
  - Measurable coverage targets aligned to criticality and risk.
  - Automated CI quality gates that prevent regressions and untested code.
  - Fast developer feedback loops with deterministic test results.

## 2. Mission and Applicability
Use this module to establish a production-grade testing discipline for backend services, frontend applications, and AI integration points.

Apply when:
- Code must be shipped to production with confidence.
- Multiple developers contribute to shared codebases.
- Failures in production require post-incident analysis and regression prevention.

Do not apply directly when:
- Project is single-developer exploration without production requirements.
- Regulatory or compliance testing requirements exceed this baseline.

## 3. Testing Pyramid Architecture
All projects adopt this canonical pyramid (bottom = fastest and most, top = slowest and least):

```
          E2E Tests (5–10% of test count)
       /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
      /  Contract Tests        \
     /   (Integration)          \
    /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
   /  Service/Unit Tests         \
  /   (60–70% of test count)      \
 /________________________________\
```

**Test layer definitions:**

### A. Unit Tests (Base Layer)
**Scope:** Individual functions, classes, and modules in isolation.
**Mocking:** Mock all external dependencies (DB, services, APIs).
**Speed:** < 100 ms per test; thousands run in seconds.
**Count:** 60–70% of all tests.

**Coverage:** Business logic, algorithms, edge cases, error paths.

Example domains:
- Validation logic and normalization
- State machine transitions
- Algorithm correctness
- Error classification and mapping

### B. Integration/Service Tests (Middle Layer)
**Scope:** Multiple components working together (e.g., service + in-process DB, service + mock external API).
**Mocking:** Mock external services; use real or test containers for databases.
**Speed:** 100 ms–2 s per test; hundreds run in seconds to minutes.
**Count:** 20–30% of all tests.

**Coverage:** Happy paths, failure paths, error handling, contract fulfillment.

Example domains:
- ORM queries and transactions
- API endpoints with realistic payloads
- Job queue processing
- State transitions with side effects

### C. Contract Tests (Middle-Upper Layer)
**Scope:** Service API contracts between independent services.
**Mocking:** Mock other services; verify request/response shape and semantics.
**Speed:** 100 ms–1 s per test; tens to hundreds.
**Count:** 5–10% of all tests.

**Coverage:** Request/response schema, error handling, versioning.

Example domains:
- REST API contract (request shape, response codes, error format)
- Async message contract (event schema, payload structure)
- SDK/client library contract (method signatures, return types)

### D. End-to-End Tests (Top Layer)
**Scope:** Full system: frontend → BFF → backend → database → external services.
**Mocking:** Minimize mocking; use real or stubbed external services (with careful handling).
**Speed:** 1–10 s per test; tens run in minutes.
**Count:** 5–10% of all tests.

**Coverage:** Critical user journeys, cross-service workflows, rollback scenarios.

Example domains:
- User signup → email verification → login
- Create session → send message → auto-rename → list sessions
- Payment processing → webhook notification → subscription update

---

## 4. Coverage Targets by Criticality

### Tier 1 (Critical Path – Payment, Auth, Data Integrity)
- **Unit test coverage:** `>= 90%` of cyclomatic complexity
- **Integration coverage:** All happy paths + all documented error cases
- **E2E coverage:** All critical user journeys
- **Contract test coverage:** All service boundaries
- **Target:** Confidence that bugs reach production < 0.1%

### Tier 2 (High Impact – Core Features)
- **Unit test coverage:** `>= 80%` of lines
- **Integration coverage:** Happy path + major error cases
- **E2E coverage:** Primary user journey per feature
- **Target:** Confidence that bugs reach production < 1%

### Tier 3 (Standard – Common Features)
- **Unit test coverage:** `>= 70%` of lines
- **Integration coverage:** Happy path + expected error handling
- **Target:** Confidence that bugs reach production < 2%

### Tier 4 (Low Impact – UI, Analytics, Non-Critical)
- **Unit test coverage:** `>= 50%` of lines
- **Target:** Confidence that bugs reach production < 5%

**Calculation:** Use coverage tools; configure to fail CI if targets not met.

---

## 5. Test Data Management

### A. Factory Pattern for Realistic Data
Use factory libraries (e.g., FactoryBoy in Python, factory_bot in Ruby) to generate realistic, deterministic test data.

```python
import factory
from users.models import User

class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = User
    
    email = factory.Sequence(lambda n: f"user{n}@example.com")
    phone = factory.Sequence(lambda n: f"+1234567890{n % 10}")
    first_name = factory.Faker('first_name')
    last_name = factory.Faker('last_name')
    is_active = True

# Usage:
user = UserFactory()
users = UserFactory.create_batch(10)
```

### B. Fixture Management
Keep fixtures minimal; prefer factories for dynamic generation.

```python
# Good: Small, reusable fixture
@pytest.fixture
def base_user():
    return UserFactory(email="base@example.com")

# Avoid: Large fixture databases
@pytest.fixture(scope="session")
def all_test_data():
    # Creates 10k records – slow, hard to reason about
    ...
```

### C. Determinism and Seeding
Ensure all tests are deterministic:
- Seed random number generators in tests.
- Use explicit dates instead of `now()`.
- Avoid test ordering dependencies.

```python
import random
from datetime import datetime, timedelta

@pytest.fixture(autouse=True)
def seed_rng():
    random.seed(42)

def test_user_created_at():
    now = datetime(2026, 4, 4, 12, 0, 0)
    user = UserFactory(created_at=now)
    assert user.created_at == now
```

---

## 6. Implementation Workflow

### Phase A: Unit Test Coverage and Organization
1. Organize tests by module: `tests/unit/<domain>/test_<module>.py`
2. Use descriptive test names: `test_<function>_<scenario>_<expected_result>`
3. Follow AAA pattern: Arrange → Act → Assert.
4. Test both happy path and all documented error cases.
5. Mock all external dependencies; use dependency injection.

Example structure:
```
src/
  users/
    models.py
    validators.py
    service.py
tests/
  unit/
    users/
      test_validators.py
      test_service.py
```

Example test:
```python
# tests/unit/users/test_validators.py
import pytest
from users.validators import validate_email

class TestValidateEmail:
    def test_valid_gmail_address(self):
        # Arrange
        email = "user@gmail.com"
        
        # Act & Assert
        assert validate_email(email) is True
    
    def test_non_gmail_rejected(self):
        # Arrange
        email = "user@hotmail.com"
        
        # Act & Assert
        with pytest.raises(ValidationError) as exc_info:
            validate_email(email)
        
        assert exc_info.value.error_code == "VALIDATION_EMAIL_DOMAIN"
    
    def test_invalid_format_rejected(self):
        email = "invalid-email"
        
        with pytest.raises(ValidationError) as exc_info:
            validate_email(email)
        
        assert exc_info.value.error_code == "VALIDATION_EMAIL_FORMAT"
```

Exit criteria:
- All modules have unit tests with `>= 70%` line coverage.
- All error paths are tested.

### Phase B: Integration and Service Tests
1. Organize tests by endpoint/workflow: `tests/integration/test_<endpoint>.py`
2. Use real or containerized databases (e.g., testcontainers).
3. Test realistic payloads and edge cases.
4. Mock only external services (third-party APIs, payment providers).
5. Verify error responses, status codes, and side effects.

Example:
```python
# tests/integration/test_user_signup.py
import pytest
from django.test import Client
from users.models import User

@pytest.fixture
def client():
    return Client()

@pytest.fixture
def db():
    # Use real test database
    pass

class TestUserSignupFlow:
    def test_signup_success_creates_user(self, client, db):
        # Arrange
        payload = {
            "email": "newuser@gmail.com",
            "first_name": "John",
            "last_name": "Doe"
        }
        
        # Act
        response = client.post("/api/v1/users/signup", json=payload)
        
        # Assert
        assert response.status_code == 201
        assert response.json()["user_id"]
        
        user = User.objects.get(email="newuser@gmail.com")
        assert user.first_name == "John"
    
    def test_signup_duplicate_email_rejected(self, client, db):
        # Arrange
        UserFactory(email="existing@gmail.com")
        payload = {"email": "existing@gmail.com", "first_name": "Jane"}
        
        # Act
        response = client.post("/api/v1/users/signup", json=payload)
        
        # Assert
        assert response.status_code == 409
        assert response.json()["error"]["code"] == "CONFLICT_DUPLICATE_EMAIL"
    
    def test_signup_invalid_email_rejected(self, client, db):
        # Arrange
        payload = {"email": "invalid-email", "first_name": "Jane"}
        
        # Act
        response = client.post("/api/v1/users/signup", json=payload)
        
        # Assert
        assert response.status_code == 400
        assert response.json()["error"]["code"] == "VALIDATION_EMAIL_FORMAT"
```

Exit criteria:
- All endpoints have integration tests covering happy path and main error cases.
- Database state is verified after each operation.

### Phase C: Contract/API Schema Tests
1. Define request/response schemas using OpenAPI or JSON Schema.
2. Generate contract tests that verify payloads match schema.
3. Test API versioning and backward compatibility.

Example (using Pydantic):
```python
# tests/integration/test_user_api_contract.py
from pydantic import BaseModel, Field, validator
import pytest

class UserSignupRequest(BaseModel):
    email: str = Field(..., regex=r"^[a-z0-9]+@gmail\.com$")
    first_name: str = Field(..., min_length=1, max_length=100)
    last_name: str = Field(..., min_length=1, max_length=100)

class UserResponse(BaseModel):
    user_id: str
    email: str
    created_at: str  # ISO 8601

def test_signup_response_matches_schema():
    payload = {"email": "user@gmail.com", "first_name": "John", "last_name": "Doe"}
    response = client.post("/api/v1/users/signup", json=payload)
    
    # This raises if response doesn't match schema
    user_response = UserResponse(**response.json()["user"])
    assert user_response.user_id
```

Exit criteria:
- All public API contracts defined in schema.
- Tests verify request/response compliance.

### Phase D: End-to-End Test Suite
1. Define critical user journeys (signup, login, core feature, payment).
2. Use real or well-stubbed external services.
3. Run in staging/test environment; avoid flakiness.
4. Each E2E test exercises multiple layers.

Example:
```python
# tests/e2e/test_user_signup_and_login.py
import pytest
from selenium import webdriver

@pytest.fixture
def browser():
    driver = webdriver.Chrome()
    yield driver
    driver.quit()

def test_signup_and_login_workflow(browser):
    # Step 1: Navigate to signup
    browser.get("https://test.example.com/signup")
    
    # Step 2: Fill signup form
    browser.find_element("email").send_keys("newuser@gmail.com")
    browser.find_element("first_name").send_keys("John")
    browser.find_element("last_name").send_keys("Doe")
    browser.find_element("submit").click()
    
    # Step 3: Verify success
    assert browser.find_element("success_message")
    
    # Step 4: Login
    browser.get("https://test.example.com/login")
    browser.find_element("email").send_keys("newuser@gmail.com")
    browser.find_element("password").send_keys("password123")
    browser.find_element("submit").click()
    
    # Step 5: Verify dashboard loaded
    assert browser.find_element("dashboard")
```

Exit criteria:
- All critical user journeys have E2E tests.
- E2E tests pass consistently (flakiness < 1%).

### Phase E: Failure Path and Chaos Testing
1. Test all documented error cases (validation, authorization, conflicts).
2. Test failure modes (external service down, database unavailable, rate limit).
3. Run chaos engineering tests (kill pods, inject latency, corrupt data).

Example:
```python
# tests/integration/test_error_paths.py
import pytest
from unittest.mock import patch

class TestErrorHandling:
    def test_external_service_timeout_handled_gracefully(self):
        with patch('external_api.call') as mock_call:
            mock_call.side_effect = TimeoutError("Service timeout")
            
            response = client.post("/api/v1/enrichment", json={"text": "hello"})
            
            assert response.status_code == 504
            assert response.json()["error"]["code"] == "EXTERNAL_SERVICE_TIMEOUT"
            assert response.json()["error"]["retryable"] is True
    
    def test_rate_limit_returns_429(self):
        with patch('rate_limiter.allow_request', return_value=False):
            response = client.post("/api/v1/enrichment", json={"text": "hello"})
            
            assert response.status_code == 429
            assert "Retry-After" in response.headers
```

Exit criteria:
- All error codes have corresponding failure path tests.
- Chaos experiments pass and are documented.

### Phase F: Performance and Load Testing
1. Define performance baselines (latency p95, throughput).
2. Run load tests before major releases.
3. Compare before/after to catch regressions.

Example (using Locust or k6):
```python
# load_tests/users_api.py
from locust import HttpUser, task, between

class UsersAPIUser(HttpUser):
    wait_time = between(1, 3)
    
    @task(2)
    def list_users(self):
        self.client.get("/api/v1/users")
    
    @task(1)
    def create_user(self):
        self.client.post("/api/v1/users", json={
            "email": "user@gmail.com",
            "first_name": "John"
        })
```

Exit criteria:
- Baseline latency/throughput documented.
- Load test passes before production release.

### Phase G: CI/CD Test Gating and Enforcement
1. Run unit tests on every commit (blocking).
2. Run integration tests on PR (blocking).
3. Run E2E tests on staging before merge (blocking).
4. Run load tests before production release (advisory).
5. Enforce coverage thresholds (block if below target).
6. Fail build on flaky tests (investigate and fix).

Example CI configuration:
```yaml
# .github/workflows/test.yml
name: Test Suite

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - run: pip install -r requirements.txt
      - run: pytest tests/unit/ -v --cov=src --cov-fail-under=70
  
  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: password
    steps:
      - uses: actions/checkout@v3
      - run: pytest tests/integration/ -v
  
  e2e-tests:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v3
      - run: docker-compose -f docker-compose.test.yml up -d
      - run: pytest tests/e2e/ -v
      - run: docker-compose -f docker-compose.test.yml down
```

Exit criteria:
- All tests pass in CI before merge.
- Coverage targets enforced.
- No flaky tests permitted.

---

## 7. Test Data and Fixture Isolation

### Database Isolation
- Use test database (separate from production/staging).
- Roll back transactions after each test (or use fixtures that auto-clean).
- Use factories to create fresh data per test.

```python
@pytest.fixture
def db_session(db):
    # db fixture from pytest-django provides isolated DB
    yield db

@pytest.fixture(autouse=True)
def cleanup_after_test(db):
    yield
    db.session.rollback()  # Auto-cleanup
```

### External Service Mocking
- Mock third-party APIs (payment, SMS, email) to avoid side effects.
- Use VCR or responses library to record and replay HTTP interactions.

```python
import responses

@responses.activate
def test_payment_processing():
    responses.add(
        responses.POST,
        "https://payment-api.example.com/charge",
        json={"charge_id": "ch_123"},
        status=200
    )
    
    result = payment_service.charge(amount=100)
    assert result.charge_id == "ch_123"
```

---

## 8. Decision Framework

| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| Test organization | By domain/feature | By test type | Use domain-based organization for discoverability. |
| DB for integration tests | Real test DB | In-memory/SQLite | Use real DB to catch SQL/transaction bugs. |
| External API mocking | Recorded responses (VCR) | Full mock library | Use VCR to maintain realistic payloads. |
| Coverage enforcement | CI gate (fail build if below target) | Advisory-only | Use CI gate for critical paths; advisory for others. |
| E2E framework | Browser-based (Selenium/Playwright) | API-only | Use browser-based for UX-heavy flows. |
| Test data generation | Factories (FactoryBoy) | Fixed fixtures | Use factories for flexibility and realism. |

---

## 9. Validation Strategy

### Test Quality Tests
- Code review gate: Require tests for every feature.
- Mutation testing: Intentionally break code; verify tests catch it.
- Coverage tools: Measure line/branch coverage.

### Flakiness Detection
- Run tests multiple times; fail if results differ.
- Track flaky tests in dashboard; escalate for fix.
- CI should re-run flaky tests automatically.

### Performance Regression Detection
- Baseline latency before each release.
- Alert if new tests or changes degrade performance > 10%.

---

## 10. Benchmarks and SLO Targets
- Unit test execution time: `<= 30 seconds` for entire suite
- Integration test execution time: `<= 5 minutes` for entire suite
- E2E test execution time: `<= 15 minutes` for critical journeys
- Unit test coverage (Tier 1): `>= 90%`
- Unit test coverage (Tier 2): `>= 80%`
- Unit test coverage (Tier 3): `>= 70%`
- Test flakiness rate: `<= 0.5%` (re-run pass rate >= 99.5%)
- CI test failure detection rate: `100%` for regressions
- Mean time to fix broken tests: `<= 2 hours`

---

## 11. Risks and Controls
- Risk: Tests too slow, developers skip running locally.
  - Control: Measure test latency; keep unit tests < 30s total.
- Risk: Flaky tests cause false negatives; team loses confidence.
  - Control: Dashboard for flaky test tracking; escalate to fix.
- Risk: Coverage targets met but tests don't validate behavior.
  - Control: Code review gate; require meaningful assertions.
- Risk: Integration tests fail sporadically due to shared state.
  - Control: Use database isolation; roll back after each test.
- Risk: Developers bypass tests and push untested code.
  - Control: Require passing CI before merge (branch protection).

---

## 12. Agent Execution Checklist
- [ ] Test pyramid defined (unit, integration, contract, E2E).
- [ ] Coverage targets defined per criticality tier.
- [ ] Unit tests written for all business logic and error paths.
- [ ] Integration tests cover all API endpoints and main workflows.
- [ ] Contract tests verify API request/response schemas.
- [ ] E2E tests cover critical user journeys.
- [ ] Test data factories implemented for realism and determinism.
- [ ] External service mocking strategy implemented.
- [ ] CI/CD test gates configured and enforced.
- [ ] Coverage tools configured; CI fails below threshold.
- [ ] Flaky test detection and escalation process in place.
- [ ] Performance baseline established.
- [ ] Team trained on testing standards and best practices.

---

## 13. Reuse Notes
This module is reusable across all backend and frontend projects. Adapt only:
- Coverage targets by product tier and criticality.
- Test frameworks and languages by tech stack.
- CI/CD configuration by infrastructure provider.
- External service mocking strategy by dependency profile.

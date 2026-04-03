# Module: Strict Input Validation Standard (Frontend/User Input -> Database)

## 1. Module Metadata
- Module ID: `agent.module.input-validation-standard`
- Version: `1.0.0`
- Maturity: `production`
- Scope: End-to-end validation, sanitization, rejection handling, and enforcement for all user-supplied inputs before persistence.
- Primary outcomes:
  - Invalid or malformed input is rejected deterministically.
  - Validation behavior is consistent across API, service, and database layers.
  - Validation quality is verifiable through automated tests and measurable controls.

## 2. Mission and Applicability
Use this module to enforce strict input validation for any data originating from frontend forms, public APIs, internal UIs, or user-submitted files.

Apply when:
- System accepts user-provided data and persists it in relational storage.
- Data quality and integrity are critical for downstream workflows.
- Inputs include contact fields (phone, email) and identity-like attributes.

Do not apply directly when:
- Input source is fully trusted machine-generated data with schema contracts already enforced upstream.
- Storage layer is append-only telemetry where malformed records are intentionally quarantined outside operational tables.

## 3. Architecture Pattern
- Pattern: `Layered validation with fail-fast rejection`
- Core design rules:
  - Validate at ingress boundary before business logic execution.
  - Re-validate at domain/service boundary for defense in depth.
  - Enforce final integrity using database constraints.
  - Treat sanitization as normalization, never as silent data repair.
  - Return structured, field-level validation errors for every reject.

Validation flow:
1. Request payload parsing and schema validation.
2. Field normalization (trim, canonical case, unicode normalization where needed).
3. Deterministic format checks (regex and parser-level checks).
4. Cross-field and business-rule checks.
5. Persist only validated canonical values under DB constraints.

## 4. Implementation Workflow
### Phase A: Validation Contract and Canonical Schema
1. Define typed request schema for every write endpoint.
2. Classify each field as required, optional, nullable, or immutable.
3. Define canonical representation for persisted values.
4. Version schema contracts when adding/changing validation rules.

Exit criteria:
- Every write path has an explicit validation contract and canonical field definitions.

### Phase B: Field Validation and Sanitization Rules
1. Implement allowlist-based validation for each field type.
2. Normalize input before regex checks (e.g., trim whitespace).
3. Reject unknown/disallowed fields for strict endpoints.
4. Keep regex patterns centralized and reused by all services.

Required example rules:
- Mobile phone:
  - Accept only strict mobile-number format required by product policy.
  - Recommended baseline (E.164): `^\+[1-9]\d{9,14}$`
  - Reject separators, letters, local shorthand, and partial numbers.
- Email (Gmail-only policy):
  - Canonicalize to lowercase.
  - Accept only `gmail.com` addresses (or `googlemail.com` if policy allows aliasing).
  - Baseline regex: `^[a-z0-9](?:[a-z0-9._%+-]{0,62}[a-z0-9])?@gmail\.com$`
  - Reject non-Gmail domains and malformed local parts.

Exit criteria:
- Shared validation library contains deterministic regex/parsing logic and normalization helpers.

### Phase C: Rejection Handling and Error Taxonomy
1. Return machine-readable validation errors (field, code, message, rejected_value metadata policy).
2. Use stable error codes (e.g., `INVALID_FORMAT`, `REQUIRED_FIELD_MISSING`, `DISALLOWED_DOMAIN`).
3. Return HTTP 400/422 consistently per API contract.
4. Log rejects with safe redaction and correlation IDs.

Exit criteria:
- Invalid inputs never reach write repositories and clients receive actionable field-level errors.

### Phase D: Database-Level Enforcement
1. Add `NOT NULL`, `CHECK`, `UNIQUE`, and FK constraints aligned with app validation.
2. Add domain-specific DB checks for canonical format where feasible.
3. Block writes that bypass app layer via DB constraint enforcement.
4. Map DB constraint failures to consistent application validation errors.

Exit criteria:
- Persistence layer rejects invalid records even if application validation is bypassed.

### Phase E: Validation Testing and Release Controls
1. Add unit tests for each field validator and sanitizer.
2. Add integration tests for endpoint reject/accept behavior.
3. Add property/fuzz tests for regex edge cases and parser robustness.
4. Add regression suites for previously rejected malformed payloads.
5. Enforce validation tests in CI as blocking gates.

Exit criteria:
- Validation behavior is reproducible, coverage-backed, and release-gated.

## 5. Decision Framework
| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| Validation location | API + service + DB layers | API-only checks | Use layered validation for defense in depth and bypass resistance. |
| Phone format | Strict E.164-like allowlist | Locale-specific permissive parsing | Use strict format unless product explicitly requires multi-locale loose inputs. |
| Email domain policy | Gmail-only allowlist | Any RFC-like email | Use Gmail-only only when business requirement explicitly mandates it. |
| Unknown fields | Reject payload | Ignore silently | Reject for contract clarity and client correctness. |
| Sanitization strategy | Normalize + validate + reject | Auto-correct and persist | Reject ambiguous inputs; avoid silent mutation. |

## 6. Validation Strategy
### Functional Validation
- Positive tests for valid phone and Gmail inputs across supported lengths.
- Positive tests for canonicalization (whitespace trim, lowercase email).
- Contract tests verifying field-level error response structure.

### Failure Validation
- Negative tests for malformed phones (letters, separators, short length, missing `+`).
- Negative tests for non-Gmail or malformed emails.
- Tests for unknown fields, missing required fields, and nullability violations.

### Contract/Security/Performance Validation
- Verify no raw secrets/PII are leaked in validation logs or error payloads.
- Verify DB constraints and app validators stay aligned after migrations.
- Verify validation latency overhead remains within API SLO budget.

## 7. Benchmarks and SLO Targets
- Invalid input rejection correctness: `100%` on defined negative test corpus
- False rejection rate on valid corpus: `<= 0.1%`
- Validation error response consistency (schema-compliant): `100%`
- Added p95 latency from validation on write endpoints: `<= 20 ms`
- DB constraint mismatch incidents per release: `0`

## 8. Risks and Controls
- Risk: App and DB validation drift.
  - Control: Migration-time contract checks and integration tests that assert parity.
- Risk: Overly strict regex rejects legitimate user data.
  - Control: Policy-driven rule review with staged rollout and monitored reject metrics.
- Risk: Error messages expose sensitive values.
  - Control: Redaction policy and safe error serialization defaults.
- Risk: Teams bypass shared validator utilities.
  - Control: Central validation package plus static checks/code review gate.

## 9. Agent Execution Checklist
- [ ] Validation schema defined for every write path.
- [ ] Central regex and normalization rules implemented and reused.
- [ ] Phone and Gmail validation rules enforced with deterministic errors.
- [ ] Unknown field rejection policy implemented.
- [ ] DB constraints aligned with application validation.
- [ ] Unit, integration, and failure-path tests added.
- [ ] CI blocks release on validation test failures.
- [ ] Reject telemetry and dashboards instrumented.

## 10. Reuse Notes
Adapt only:
- Field catalog and business-specific allowlists.
- Regional phone constraints when business policy changes.
- Domain allowlist policy (e.g., Gmail-only vs broader enterprise policy).
- Error code catalog extensions while preserving compatibility.

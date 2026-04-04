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
5. Maintain explicit schema versions (e.g., `v1`, `v2`) with compatibility guarantees.
6. Define backward-compatibility windows and deprecation policy for older clients.
7. Roll out stricter validation rules gradually using version-scoped policy toggles.

Exit criteria:
- Every write path has an explicit validation contract and canonical field definitions.
- Versioned schemas, compatibility behavior, and rollout policy are documented and test-covered.

### Phase B: Field Validation and Sanitization Rules
1. Implement allowlist-based validation for each field type.
2. Normalize input before regex checks (e.g., trim whitespace).
3. Reject unknown/disallowed fields for strict endpoints.
4. Keep regex patterns centralized and reused by all services.
5. Enforce additional field constraints for names, addresses, files, UUIDs/IDs, and ISO dates.

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
- Name fields:
  - Enforce character allowlist policy (letters, approved separators, locale-specific characters by policy).
  - Reject control characters, unsupported symbols, and invisible unicode separators.
- Address fields:
  - Enforce per-line length limits and approved charset policy.
  - Reject binary/control characters and overlong payloads.
- File uploads:
  - Validate MIME type, extension allowlist, and max size.
  - Perform content validation (magic-byte/type signature check) and reject mismatched or unsafe payloads.
- UUID/ID fields:
  - Enforce strict expected format (e.g., UUID v4 pattern) and reject non-canonical IDs.
- Date fields:
  - Accept ISO 8601 only; reject locale-specific ambiguous date strings.

Exit criteria:
- Shared validation library contains deterministic regex/parsing logic and normalization helpers.
- Expanded field-type validators are centralized, reusable, and covered by tests.

### Phase C: Rejection Handling and Error Taxonomy
1. Return machine-readable validation errors (field, code, message, rejected_value metadata policy).
2. Use stable error codes (e.g., `INVALID_FORMAT`, `REQUIRED_FIELD_MISSING`, `DISALLOWED_DOMAIN`).
3. Return HTTP 400/422 consistently per API contract.
4. Log rejects with safe redaction and correlation IDs.

#### Cross-Context Validation
1. Apply conditional validation rules based on business context and workflow state.
2. Encode context predicates explicitly (e.g., field required only when another field or action flag is present).
3. Prevent contradictory rule sets by validating context rules in schema build pipelines.
4. Emit deterministic context-specific error codes for failed conditional requirements.

Exit criteria:
- Invalid inputs never reach write repositories and clients receive actionable field-level errors.
- Conditional and cross-field validation rules are deterministic, test-covered, and versioned.

### Phase D: Database-Level Enforcement
1. Add `NOT NULL`, `CHECK`, `UNIQUE`, and FK constraints aligned with app validation.
2. Add domain-specific DB checks for canonical format where feasible.
3. Block writes that bypass app layer via DB constraint enforcement.
4. Map DB constraint failures to consistent application validation errors.
5. Use database domain types (e.g., PostgreSQL `DOMAIN`) for reusable, centralized field constraints.
6. Add partial indexes for conditional constraints that apply only to subset populations.
7. Use triggers for complex invariants not expressible with standard checks, while preserving deterministic failure semantics.

Exit criteria:
- Persistence layer rejects invalid records even if application validation is bypassed.
- DB-level reusable domains, conditional constraints, and trigger-based invariants are aligned with app-level contracts.

### Phase E: Validation Testing and Release Controls
1. Add unit tests for each field validator and sanitizer.
2. Add integration tests for endpoint reject/accept behavior.
3. Add property/fuzz tests for regex edge cases and parser robustness.
4. Add regression suites for previously rejected malformed payloads.
5. Enforce validation tests in CI as blocking gates.

#### Shadow Validation Mode
1. Run new validation rules in non-blocking mode on live traffic.
2. Record would-fail decisions with rule ID, endpoint, and field metadata.
3. Compare shadow-failure rate against production acceptance baseline.
4. Promote rules to blocking enforcement only after confidence thresholds are met.

Exit criteria:
- Validation behavior is reproducible, coverage-backed, and release-gated.
- New rules graduate from shadow mode to enforcement based on measured confidence and operational review.

### Phase F: Abuse Protection & Rate Limiting
1. Apply rate limiting policies on validation-heavy endpoints by IP, user, and credential context.
2. Detect repeated invalid payload signatures and high-frequency reject patterns.
3. Throttle or temporarily block abusive clients using configurable policy thresholds.
4. Integrate IP/user-based anomaly detection and escalation workflows.

Exit criteria:
- Abuse-driven invalid traffic is contained without degrading legitimate request success rates.

## 5. Security Validation Layer
- Validate and sanitize inputs against XSS vectors using context-aware output encoding strategy.
- Reject script tags and unsafe HTML where rich-text input is not explicitly allowed.
- Enforce parameterized queries for all persistence paths to eliminate SQL injection risk.
- Validate JSON payloads strictly against schemas to prevent over-posting and mass assignment.
- Restrict writable fields through explicit allowlists at DTO/model mapping boundaries.
- Apply payload size limits and depth limits to mitigate parser/resource abuse.

## 6. Decision Framework
| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| Validation location | API + service + DB layers | API-only checks | Use layered validation for defense in depth and bypass resistance. |
| Phone format | Strict E.164-like allowlist | Locale-specific permissive parsing | Use strict format unless product explicitly requires multi-locale loose inputs. |
| Email domain policy | Gmail-only allowlist | Any RFC-like email | Use Gmail-only only when business requirement explicitly mandates it. |
| Unknown fields | Reject payload | Ignore silently | Reject for contract clarity and client correctness. |
| Sanitization strategy | Normalize + validate + reject | Auto-correct and persist | Reject ambiguous inputs; avoid silent mutation. |
| Schema rollout | Versioned schemas (`v1`, `v2`) + staged enforcement | Global in-place rule switch | Use versioning when validation strictness changes could break existing clients. |
| New rule activation | Shadow validation then enforce | Immediate hard reject | Use shadow mode for high-risk rule changes to reduce production regressions. |
| Abuse mitigation | Adaptive rate limit + anomaly detection | Static global limits only | Use adaptive controls for endpoints prone to invalid payload bursts. |

## 7. Validation Strategy
### Functional Validation
- Positive tests for valid phone and Gmail inputs across supported lengths.
- Positive tests for canonicalization (whitespace trim, lowercase email).
- Contract tests verifying field-level error response structure.
- Positive tests for names, addresses, UUID formats, ISO 8601 dates, and allowed file uploads.
- Context-rule tests for conditionally required fields under all supported business states.

### Failure Validation
- Negative tests for malformed phones (letters, separators, short length, missing `+`).
- Negative tests for non-Gmail or malformed emails.
- Tests for unknown fields, missing required fields, and nullability violations.
- Negative tests for invalid UUID/date formats, oversized files, unsafe MIME/content mismatches, and disallowed charset usage.
- Abuse tests for repeated invalid payload patterns to verify throttling/blocking behavior.

### Contract/Security/Performance Validation
- Verify no raw secrets/PII are leaked in validation logs or error payloads.
- Verify DB constraints and app validators stay aligned after migrations.
- Verify validation latency overhead remains within API SLO budget.
- Validate XSS and injection rejection paths, including unsafe HTML/script payload tests.
- Validate strict JSON schema behavior for over-posting/mass-assignment prevention.
- Run load benchmarks to confirm regex safety (no catastrophic backtracking) and stable latency under concurrency.

## 8. Validation Observability
- Track validation rejection rate per endpoint, client class, and schema version.
- Track most frequently failing fields and error codes over rolling windows.
- Alert on sudden spikes in validation failures, shadow-fail rates, and abuse-triggered throttles.
- Emit dashboard-ready metrics and structured events for operational and product analytics.
- Maintain correlation between validation failures and downstream DB constraint rejects for drift detection.

## 9. Benchmarks and SLO Targets
- Invalid input rejection correctness: `100%` on defined negative test corpus
- False rejection rate on valid corpus: `<= 0.1%`
- Validation error response consistency (schema-compliant): `100%`
- Added p95 latency from validation on write endpoints: `<= 20 ms`
- DB constraint mismatch incidents per release: `0`
- Shadow validation false-positive delta before enforcement: `<= 0.2%`
- Abuse detection precision for high-volume invalid traffic: `>= 95%`
- Validation metrics ingestion completeness: `100%` for blocking endpoints

## 10. Performance Considerations
- Precompile regex patterns and reuse compiled validators across requests.
- Design regex expressions to avoid catastrophic backtracking and exponential execution paths.
- Cap input lengths before regex execution to bound compute cost.
- Benchmark validation latency and throughput under realistic concurrent write load.
- Profile high-cardinality validation paths and optimize hotspot serializers/parsers.

## 11. Idempotency & Duplicate Protection
- Enforce idempotency keys for write endpoints that can be retried by clients.
- Store idempotency key fingerprints with bounded TTL and deterministic replay behavior.
- Reject duplicate submissions with stable error semantics where idempotency is required.
- Combine uniqueness constraints and application-level deduplication safeguards for race-safe duplicate prevention.

## 12. Developer Experience
- Provide a shared validation SDK/library used by all services and clients where applicable.
- Auto-generate validator scaffolding and contracts from canonical schema definitions.
- Standardize validation error response formatting and error code registries.
- Publish rule catalogs and migration guidance for schema version changes.
- Provide local validation test harnesses to accelerate developer feedback loops.

## 13. Risks and Controls
- Risk: App and DB validation drift.
  - Control: Migration-time contract checks and integration tests that assert parity.
- Risk: Overly strict regex rejects legitimate user data.
  - Control: Policy-driven rule review with staged rollout and monitored reject metrics.
- Risk: Error messages expose sensitive values.
  - Control: Redaction policy and safe error serialization defaults.
- Risk: Teams bypass shared validator utilities.
  - Control: Central validation package plus static checks/code review gate.
- Risk: Rate-limit abuse controls block legitimate clients.
  - Control: Tiered thresholds, exception workflows, and continuous precision/recall tuning.
- Risk: Shadow-mode metrics are ignored, causing unsafe enforcement transitions.
  - Control: Explicit promotion gates and release sign-off criteria tied to shadow metrics.

## 14. Agent Execution Checklist
- [ ] Validation schema defined for every write path.
- [ ] Central regex and normalization rules implemented and reused.
- [ ] Phone and Gmail validation rules enforced with deterministic errors.
- [ ] Expanded field-type validation (name, address, file, UUID, ISO date) implemented.
- [ ] Unknown field rejection policy implemented.
- [ ] Cross-context/conditional validation rules implemented and tested.
- [ ] DB constraints aligned with application validation.
- [ ] DB domain types and conditional/complex constraints designed where applicable.
- [ ] Unit, integration, and failure-path tests added.
- [ ] Shadow validation mode enabled for new high-risk rules before enforcement.
- [ ] Abuse rate-limiting and anomaly detection controls deployed.
- [ ] CI blocks release on validation test failures.
- [ ] Reject telemetry and dashboards instrumented.
- [ ] Idempotency and duplicate-protection strategy implemented for retryable writes.

## 15. Reuse Notes
Adapt only:
- Field catalog and business-specific allowlists.
- Regional phone constraints when business policy changes.
- Domain allowlist policy (e.g., Gmail-only vs broader enterprise policy).
- Error code catalog extensions while preserving compatibility.
- Schema version lifecycle windows and rollout strategy by client ecosystem.
- Abuse-threshold and anomaly rules calibrated for endpoint traffic profile.

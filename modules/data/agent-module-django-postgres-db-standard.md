# Module: Relational Database Creation Standard (Django + PostgreSQL)

## 1. Module Metadata
- Module ID: `agent.module.db-design-django-postgres`
- Version: `1.0.0`
- Maturity: `production`
- Scope: Schema design, validation, indexing, ORM patterns, and quality controls for operational systems.
- Primary outcomes:
  - Transaction-safe relational core.
  - Query-aligned indexing and predictable performance.
  - Layered validation and ingestion reliability.

## 2. Mission and Applicability
Use this module to design and implement production-grade PostgreSQL schemas in Django-based systems.

Apply when:
- Domain requires strong consistency for transactional workflows.
- APIs demand filtered list views, auditability, and repeatable ingestion.
- Team uses Django ORM with DRF or equivalent service layer.

Do not apply directly when:
- Workload is append-only analytics without transactional constraints.
- Primary storage is fully document/columnar/graph-native.

## 3. Canonical Architecture Pattern
- Pattern: `Relational source of truth + operational telemetry tables`
- Data domains:
  - Identity and access
  - Core operational entities
  - Ingestion/sync state and error logs
  - Audit trail and compliance events
- Persistence principles:
  - FK relationships for ownership and accountability
  - JSON fields only for bounded semi-structured payloads
  - Business key upsert logic for external data feeds

## 4. Implementation Workflow

### Phase A: Domain and Table Modeling
1. Define domain aggregates and lifecycle boundaries.
2. Choose PK strategy (`BigAutoField` or UUID by scaling profile).
3. Encode business keys and uniqueness invariants.
4. Define delete behavior (`SET_NULL`, `PROTECT`, `CASCADE`) intentionally.

Exit criteria:
- ER model finalized with explicit cardinality and ownership semantics.

### Phase B: Constraint and Integrity Layer
1. Implement DB-level primary, unique, FK constraints.
2. Add nullable unique handling rules (`'' -> NULL`) in ingestion paths.
3. Add application-level validators for format and canonicalization.
4. Add immutable audit entries for high-value transitions.

Exit criteria:
- Invalid or duplicate operational data cannot silently persist.

### Phase C: Index Strategy and Query Alignment
1. List all high-frequency filters, sorts, and lookups.
2. Add single and composite indexes aligned to real query paths.
3. Include descending timestamp indexes for latest-first dashboards.
4. Validate index utility via `EXPLAIN ANALYZE` on representative queries.

Exit criteria:
- Core endpoints hit index-friendly execution plans.

### Phase D: ORM and Transaction Patterns
1. Use `select_related` and `prefetch_related` to avoid N+1.
2. Keep writes atomic around multi-step state transitions.
3. Handle idempotent import/sync with conflict-aware logic.
4. Capture `IntegrityError` and convert to deterministic app responses.

Exit criteria:
- Data writes are correct under concurrency and replay scenarios.

### Phase E: Evolution and Migration Discipline
1. Move from auto-create to migration-based lifecycle (`makemigrations`/`migrate`, Alembic equivalent where applicable).
2. Add migration checks to CI.
3. Include rollback/runbook notes for irreversible schema changes.

Exit criteria:
- Schema evolution is auditable, reproducible, and safe to deploy.

## 5. Decision Framework

| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| Relationship retention | `SET_NULL` for historical references | `CASCADE` | Use `SET_NULL` when history must survive user deletion. |
| Validation location | DB + app layered validation | App-only validation | Use layered validation for ingestion-heavy systems. |
| Semi-structured payloads | Bounded `JSONField` | Fully normalized relational expansion | Use JSON for evolving AI/import payloads, normalize if query-heavy. |
| Payment/ingestion linking | Business keys with idempotent checks | Strict FK coupling | Use business keys when pipelines are asynchronous and independent. |

## 6. Validation Strategy

### Schema Validation
- Enforce uniqueness and FK constraints in migrations.
- Validate nullable unique behavior in tests.

### Ingestion Validation
- Regex/parsing normalization tests for identifiers, emails, and phone numbers.
- Conflict and replay tests for repeated file/sheet imports.

### Query Validation
- Performance tests on list/filter endpoints.
- Assert no N+1 regressions in serializers/views.

### Audit Validation
- Ensure issuance/revoke/sync operations produce immutable audit rows.

## 7. Benchmarks and SLO Targets
- Core list endpoint latency p95 (indexed filters, typical payload): `<= 250 ms`
- Import deduplication correctness: `100%` across replayed batches
- N+1 query regressions: `0` in critical endpoints
- Migration success rate in CI/staging: `100%` before production deploy
- Audit logging coverage for high-value actions: `100%`

## 8. Quality Bar for Production Readiness
A schema implementation is production-ready only when all are true:
- Constraints encode real business invariants.
- Indexes match actual query patterns, not hypothetical ones.
- Ingestion paths are idempotent and observable.
- Audit and sync error tables support operational remediation.
- Migration workflow is deterministic and version-controlled.

## 9. Agent Execution Checklist
- [ ] Domain ER model finalized and reviewed.
- [ ] DB constraints and FK actions implemented.
- [ ] Validation and normalization layer implemented.
- [ ] Query-aligned indexes added and benchmarked.
- [ ] ORM query optimization applied to hot paths.
- [ ] Migration and rollback procedures validated.
- [ ] Audit and sync telemetry verified end-to-end.

## 10. Reuse Notes
This module is reusable across back-office and workflow-heavy systems. Adapt only:
- Entity names and business keys.
- Index combinations by API usage patterns.
- Latency/SLO thresholds by workload scale.

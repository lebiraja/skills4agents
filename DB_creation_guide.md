# Database Creation Guide

## 1. Purpose and Scope

This document describes how the database layer is implemented in the Mindkraft Dashboard backend (`backend/`) and converts those implementation decisions into a reusable, production-ready database creation standard.

Focus areas covered:
- Schema design and table structures
- Relationships and constraints
- Indexing strategy and query optimization alignment
- Validation mechanisms (regex, parsing, application rules)
- ORM integration, query patterns, and transaction handling
- Data flow from ingestion to operational analytics

Tech stack context:
- Framework: Django + Django REST Framework
- Database engine: PostgreSQL (`django.db.backends.postgresql`)
- ORM: Django ORM

---

## 2. Database Architecture Overview

### 2.1 Logical Domains

The schema is organized into three functional domains:
- Identity and access: custom user model (`accounts_user`)
- Registration operations: participant records, sync state, sync errors, and audit trail
- Payment operations: imported payment records and payment upload audit

### 2.2 App-to-DB Mapping

- `accounts` app owns authentication users.
- `registrations` app owns participant lifecycle, Google Sheet sync bookkeeping, analytics aggregations, and payment ingestion records.

### 2.3 Persistence Characteristics

- Strongly relational core with referential links to user table via `ForeignKey(..., on_delete=SET_NULL)`.
- Uses JSON columns (`JSONField`) for semi-structured payloads (raw sync rows, error lists, audit details, sync results).
- Most writes occur through ORM create/update workflows; no raw SQL usage.

---

## 3. Physical Schema Specification

## 3.1 `accounts_user`

Source: `backend/accounts/models.py`

Purpose:
- Authentication principal for admin/coordinator roles.

Key columns:
- `id` (`BigAutoField`, PK)
- Standard Django auth fields (`username`, `password`, etc.)
- `role` (`CharField(20)`, choices: `admin`, `coordinator`, default `coordinator`)

Constraints and behavior:
- `username` unique (inherited from `AbstractUser`)
- Custom manager enforces superuser invariants:
  - `is_staff=True`
  - `is_superuser=True`
  - `role='admin'`

Table name:
- `accounts_user`

---

## 3.2 `registrations_internal`

Source: `backend/registrations/models.py`

Purpose:
- Internal participant master records (Karunya participants).

Key columns:
- `id` PK
- `name` (`CharField(255)`, indexed)
- `registration_number` (`CharField(50)`, unique)
- `payment_receipt_number` (`CharField(100)`, nullable, unique)
- `mobile_no` (`CharField(15)`)
- `email` (`EmailField`)
- `division_name` (`CharField(200)`, indexed)
- `year_of_study` (`CharField(50)`, indexed)
- `is_registered` (`BooleanField`, indexed)
- `registered_at` (`DateTimeField`, nullable)
- `registered_by` (FK to `accounts_user`, `SET_NULL`, nullable)
- `source` (`upload` or `gsheet`)
- `created_at`, `updated_at`

Indexes:
- Single-column:
  - `name`
  - `division_name`
  - `year_of_study`
  - `is_registered`
  - descending `created_at` (`internal_created_desc_idx`)
- Composite:
  - `(name, is_registered)` -> `internal_name_status_idx`
  - `(is_registered, -created_at)` -> `internal_status_date_idx`
  - `(division_name, is_registered)` -> `internal_div_status_idx`
  - `(year_of_study, is_registered)` -> `internal_year_status_idx`

Design intent:
- Supports high-frequency filters/searches by status/date/name/division/year.
- Separates source tracking from event registration state.

---

## 3.3 `registrations_external`

Source: `backend/registrations/models.py`

Purpose:
- External participant master records.

Key columns:
- `id` PK
- `name` (`CharField(255)`, indexed)
- `institution_name` (`CharField(255)`, indexed)
- `designation` (`CharField(100)`)
- `college_id` (`CharField(100)`, nullable, unique)
- `mobile_no` (`CharField(15)`)
- `email` (`EmailField`)
- `receipt_payment_id` (`CharField(100)`, nullable, unique)
- `is_registered` (`BooleanField`, indexed)
- `registered_at` (`DateTimeField`, nullable)
- `registered_by` (FK to `accounts_user`, `SET_NULL`, nullable)
- `source` (`upload` or `gsheet`)
- `created_at`, `updated_at`

Indexes:
- Single-column:
  - `name`
  - `institution_name`
  - `is_registered`
  - descending `created_at` (`external_created_desc_idx`)
- Composite:
  - `(name, is_registered)` -> `external_name_status_idx`
  - `(is_registered, -created_at)` -> `external_status_date_idx`
  - `(institution_name, is_registered)` -> `external_inst_status_idx`

Design intent:
- Optimized for operator lookup by receipt/payment ID and institution-based dashboarding.

---

## 3.4 `sync_config`

Purpose:
- Per participant type sync metadata and operational lock state.

Key columns:
- `participant_type` unique (`internal` or `external`)
- `sheet_url`
- `last_synced_at`
- `last_sync_result` (`JSONField`)
- `is_syncing` (`BooleanField`) for concurrency gate
- `updated_by` FK -> `accounts_user` (`SET_NULL`)
- `created_at`, `updated_at`

Table name:
- `sync_config`

Design intent:
- Keeps sync state idempotent and globally visible.
- Serves as both configuration and operational telemetry record.

---

## 3.5 `sync_errors`

Purpose:
- Stores hard-rejected rows during sheet sync.

Key columns:
- `participant_type` (indexed)
- `sync_session_id` (`UUID`, indexed)
- `row_number` (`PositiveIntegerField`)
- `raw_data` (`JSONField`)
- `errors` (`JSONField`)
- `resolved` (`BooleanField`, indexed)
- `created_at`

Ordering:
- `-created_at`, `row_number`

Table name:
- `sync_errors`

Design intent:
- Supports admin remediation loops and traceable batch rejects.

---

## 3.6 `audit_log`

Purpose:
- Immutable event history for high-value operations.

Key columns:
- `user` FK -> `accounts_user` (`SET_NULL`)
- `action` (`ID_ISSUED`, `ID_REVOKED`, `SYNC_TRIGGERED`, `SHEET_URL_CHANGED`) indexed
- `participant_type` indexed
- `participant_id`
- `participant_name`
- `details` (`JSONField`)
- `created_at` indexed

Table name:
- `audit_log`

Design intent:
- Event sourcing lite: explicit audit rows for all issuance/revoke/sync actions.

---

## 3.7 `payments_internal`

Purpose:
- Internal payment evidence imported from EduServ files.

Key columns:
- `payment_date` (indexed)
- `registration_number` (indexed)
- `student_name`
- `event_name`
- `numbers` (`PositiveIntegerField`)
- `amount` (`DecimalField(10,2)`)
- `source`
- `created_at`, `updated_at`

Constraints:
- `UniqueConstraint(registration_number, payment_date)` -> `unique_internal_payment`

Indexes:
- `int_pay_regno_idx` on `registration_number`
- `int_pay_date_idx` on `payment_date`

Table name:
- `payments_internal`

---

## 3.8 `payments_external`

Purpose:
- External payment evidence imported from EduServ files.

Key columns:
- `payment_date` (indexed)
- `student_name` (indexed)
- `event_name`
- `numbers` (`PositiveIntegerField`)
- `amount` (`DecimalField(10,2)`)
- `source`
- `created_at`, `updated_at`

Constraints:
- `UniqueConstraint(student_name, payment_date)` -> `unique_external_payment`

Indexes:
- `ext_pay_name_idx` on `student_name`
- `ext_pay_date_idx` on `payment_date`

Table name:
- `payments_external`

---

## 3.9 `payment_upload_log`

Purpose:
- Upload run audit for payment imports.

Key columns:
- `uploaded_by` FK -> `accounts_user` (`SET_NULL`)
- `upload_date`
- `file_name`
- `participant_type`
- `records_processed`
- `records_created`
- `records_skipped`
- `errors` (`JSONField`, list)

Table name:
- `payment_upload_log`

Design intent:
- Supports operator observability and replay diagnostics for ingestion batches.

---

## 4. Relationships and Referential Rules

### 4.1 Foreign Key Graph

- `registrations_internal.registered_by -> accounts_user.id` (`SET_NULL`)
- `registrations_external.registered_by -> accounts_user.id` (`SET_NULL`)
- `sync_config.updated_by -> accounts_user.id` (`SET_NULL`)
- `audit_log.user -> accounts_user.id` (`SET_NULL`)
- `payment_upload_log.uploaded_by -> accounts_user.id` (`SET_NULL`)

### 4.2 Relationship Design Rationale

- `SET_NULL` is consistently chosen to preserve historical records if users are deleted.
- Participant/payment entities intentionally do not FK to each other; they are linked by business keys (registration number, student name, receipt ids) to tolerate independent import pipelines.

---

## 5. Constraint Strategy

### 5.1 Database-level Constraints

Implemented constraints:
- Primary keys (`BigAutoField`) on all tables
- Unique constraints:
  - `registrations_internal.registration_number`
  - `registrations_internal.payment_receipt_number` (nullable unique)
  - `registrations_external.college_id` (nullable unique)
  - `registrations_external.receipt_payment_id` (nullable unique)
  - Composite uniques on payment tables
- Foreign key constraints with `SET_NULL`
- Type/domain constraints from Django field types (date, decimal, positive integer)

Not implemented at DB level:
- CHECK constraints for regex/format (email/receipt/phone)
- Cross-table FK between participant and payment tables

### 5.2 Application-level Integrity Guards

- Input normalization converts blank unique keys to `None`, preventing empty-string uniqueness collisions.
- Upsert-like sync behavior updates existing rows by stable business keys.
- Payment sync pre-checks existence before insert and catches `IntegrityError` to enforce idempotency.
- Audit entries are generated for sensitive state transitions.

---

## 6. Indexing Strategy and Query Alignment

### 6.1 Explicit Index Design

The project intentionally aligns indexes with API filters and sorting patterns:
- Descending timestamp indexes (`-created_at`) support latest-first list views.
- Composite status + dimension indexes support dashboard queries and list filters.
- Payment indexes support cross-verification lookups by registration number/name/date.

### 6.2 Query Patterns Mapped to Indexes

- Internal list/lookup:
  - filters by `is_registered`, `division_name`, `year_of_study`
  - search on `name`, `registration_number`, `payment_receipt_number`
- External list/lookup:
  - filters by `is_registered`, `institution_name`
  - search on `name`, `receipt_payment_id`, `college_id`
- Sync error review:
  - `participant_type`, `resolved`, `sync_session_id`
- Payment exploration:
  - `registration_number` or `student_name`, plus payment date ranges

### 6.3 Performance-aware ORM usage

- `select_related('registered_by')` is used on participant list/lookup endpoints to avoid N+1 when returning `registered_by.username`.
- Aggregations use `values(...).annotate(Count(..., filter=Q(...)))` for DB-side group computation.

---

## 7. Validation Architecture

Validation follows a layered approach:
- Parsing and schema mapping
- Normalization and auto-resolution
- Hard validation and reject logging
- Database constraints as final guardrail

## 7.1 Regex and Pattern Rules

Defined regexes:
- `RECEIPT_RE = r'^[RE]\d{7}$'`
  - Intended receipt format (R/E + 7 digits)
  - Currently strict enforcement is disabled in `_validate_receipt` (`pass`), indicating business-policy relaxation
- `EMAIL_RE = r'^[^@\s]+@[^@\s]+\.[^@\s]+$'`
  - Lightweight email structure check
- `PHONE_STRIP_RE = r'[\s\-\(\)\+]'`
  - Strips separators and plus symbols before numeric validation
- Internal payment reg no pattern:
  - `_REG_NUMBER_RE = r'^(URK|ULK|PRK|RRK|UTK)\d{2}[A-Z]{2}\d{3,4}$'`

## 7.2 Input Validation and Auto-resolution Rules

Internal/external sync validators (`validators.py`) do:
- String coercion and trimming
- Float-to-int string conversion for numeric spreadsheet cells
- Phone canonicalization (remove symbols, normalize `91` prefix)
- Lowercase email normalization
- Uppercase participant names
- Institution alias canonicalization (large mapping dictionary)
- Length truncation by model max lengths with warning capture
- Unique field nullification (`'' -> None`) for nullable unique columns

Hard-required fields:
- Internal: `name`, `registration_number`, `payment_receipt_number`
- External: `name`, `receipt_payment_id`

Hard-reject behavior:
- Missing required fields -> rejected row (`SyncError` persisted)
- Invalid email/phone -> kept as issues (not hard reject unless policy escalates)
- Receipt format currently not hard-enforced due disabled validator function

## 7.3 File Parsing Validation (Payment Import)

`payment_parser.py` applies:
- Required column checks by participant type
- Multi-format date parsing
- Decimal amount parsing with currency symbol cleanup
- Positive integer quantity checks
- Non-empty student name checks
- Internal registration format regex validation

Invalid rows are returned as parse errors and written to upload log.

---

## 8. Data Flow and Write Paths

## 8.1 Internal/External Participant Sync Flow

1. API receives Google Sheet URL.
2. URL validation and role checks.
3. Sheet fetched via service account.
4. Column mapping to canonical field names.
5. Row validation + normalization.
6. Rejected rows stored in `sync_errors` with session UUID.
7. Valid rows upserted into participant table.
8. Sync metadata persisted in `sync_config.last_sync_result`.
9. Previously unresolved rows auto-marked resolved if no longer rejected in later runs.

## 8.2 Registration Toggle Flow

1. Operator toggles participant registration state.
2. `is_registered`, `registered_at`, `registered_by` updated.
3. `audit_log` row inserted with contextual details.

## 8.3 Bulk ID Issue Flow (Internal)

1. CSV/Excel uploaded and normalized.
2. Existing participants fetched by registration numbers.
3. New participants prepared for create; existing pending participants prepared for update.
4. In one `transaction.atomic()` block:
   - `bulk_update`
   - `bulk_create`
   - `audit_log.bulk_create`

This is the primary explicit transactional unit ensuring all-or-nothing behavior for bulk issuance.

## 8.4 Payment Import Flow

1. Upload CSV/Excel.
2. Parse and validate row-level fields.
3. Deduplicate by business key:
   - Internal: (`registration_number`, `payment_date`)
   - External: (`student_name`, `payment_date`)
4. Insert accepted rows.
5. Persist batch result into `payment_upload_log`.

## 8.5 Lookup and Cross-verification Flow

Internal lookup cross-checks participants with `payments_internal` by registration number/name to show operational verification context.

---

## 9. ORM Usage Patterns

Common patterns used:
- Retrieval:
  - `.filter()`, `.exclude()`, `.get()`, `.first()`, `.exists()`
- Projection/Aggregation:
  - `.values(...).annotate(...)`
- Relation optimization:
  - `.select_related('registered_by')`
- Pagination:
  - slice-based (`queryset[start:end]`)
- Batch operations:
  - `.bulk_create()`, `.bulk_update()`
- Upsert-like operations:
  - `.get_or_create()` for sync config
- Set-based update:
  - `.update(...)` for lock/unlock and resolve marks

No usage observed:
- Raw SQL
- Advanced pessimistic locking (`select_for_update`)

---

## 10. Transaction and Concurrency Handling

### 10.1 Explicit Transactions

- `internal_bulk_issue` uses `transaction.atomic()` to keep updates/creates/audit logs consistent.

### 10.2 Concurrency Controls

- External sync endpoint uses `SyncConfig.is_syncing` as an application-level lock.
- Internal sync endpoint uses rate limiting but does not apply the same `is_syncing` lock pattern.
- Sync error de-duplication deletes prior unresolved row-level duplicates before inserting new error entries.

### 10.3 Idempotency Approach

- Participant sync updates existing records by business key and skips unchanged rows.
- Payment import checks existence and relies on unique constraints + `IntegrityError` handling.

---

## 11. Migration Evolution and Design Decisions

Migration sequence (`registrations`):
- `0001_initial`: core participant/sync tables
- `0002...`: added audit log, sync flags, composite indexes (also temporary flag fields)
- `0003...`: added payment models and upload log
- `0004...`: removed obsolete participant flag fields

Key decisions reflected:
- Shift toward auditability (`audit_log`, `payment_upload_log`)
- Query-driven index additions (composite indexes introduced after baseline)
- Controlled rollback/removal of temporary flag mechanism

---

## 12. Standardized Database Creation Blueprint (Reusable)

Use this as the default blueprint for future production-grade systems.

## 12.1 Modeling Standard

1. Identify stable business keys early.
2. Keep operational state (`is_registered`, `is_syncing`) explicit and indexed.
3. Use nullable unique fields only when blank values are normalized to `NULL`.
4. Separate event/audit records from mutable master records.
5. Prefer JSON columns only for truly variable payloads, not core query dimensions.

## 12.2 Constraint Standard

1. Enforce entity identity with unique constraints.
2. Add composite unique constraints for ingestion idempotency.
3. Use FK `SET_NULL` for historical/audit survivability.
4. Push critical invariants to DB constraints where feasible.
5. Keep application validations aligned with DB rules to avoid drift.

## 12.3 Index Standard

1. Create indexes from real query predicates, not speculation.
2. Add composite indexes for common conjunctions (`status + dimension`, `status + date`).
3. Verify sort-heavy endpoints have supporting order indexes.
4. Revisit index selectivity as data volume grows.

## 12.4 Validation Standard

1. Stage validation into parse -> normalize -> hard validate -> persist.
2. Keep regexes centralized and versioned.
3. Log rejected rows with raw payload + explicit error list.
4. Preserve soft warning telemetry for data quality operations.
5. Make policy toggles explicit (for example strict vs relaxed receipt format).

## 12.5 Transaction Standard

1. Wrap multi-table or multi-step writes in `transaction.atomic()`.
2. For concurrency-sensitive flows, use lock rows/state flags or DB row-level locks.
3. Design all ingestion endpoints to be idempotent under retries.

## 12.6 Operational Observability Standard

1. Persist per-run metadata (`session_id`, counts, timestamps).
2. Capture user attribution for all admin actions.
3. Maintain dedicated error/retry tables for failed ingestions.

---

## 13. Implementation Notes for Agents

When using this guide to design a new system:
- Start with explicit business keys and ingestion idempotency constraints.
- Design indexes directly from anticipated API filters and sorting.
- Add an audit log table from day one.
- Build validator pipeline before implementing bulk ingestion endpoints.
- Use explicit transactions for batch write paths.
- Document every regex, required field rule, and null-handling decision as part of schema governance.

This project demonstrates a practical production pattern: relational core + JSON for operational metadata + deterministic validation pipeline + query-driven indexing.

# TCS Automation — SQL Schema (DDL)

**Version:** 0.1 — Design Draft
**Status:** Working Draft
**Baseline:** Domain Model & Database Schema v0.3 / Test Run State Machine v0.4

## 1. Purpose and Scope

This document translates the entities, relationships, invariants, and state rules already agreed in the Domain Model (v0.3) and Test Run State Machine (v0.4) into executable SQL. Nothing here introduces new *behavior* — every constraint below enforces a rule that was already decided in an earlier document, with a citation back to it. Where this document makes a new decision (mainly: exact types, since that was explicitly deferred), it's called out as such.

## 2. Assumptions Made (resolving previously-deferred items)

The Domain Model left several items open (Sec 17). This DDL needs concrete answers to compile, so the following defaults are chosen — all easily revisited without touching the conceptual model:

| Open item | Decision made here | Why |
|---|---|---|
| Database engine | **PostgreSQL syntax**, with SQLite adjustment notes (Sec 12) | V1 is local-only, so SQLite is the likely real target; Postgres syntax is clearer to read/review and the two are close enough that the translation is mechanical. |
| ID strategy | **UUID, stored as `TEXT`** | Avoids exposing sequential IDs across a future multi-user/centralized deployment; generated in application code (not DB-generated), keeping ID generation portable across engines. |
| Timestamps | `TIMESTAMPTZ` (Postgres) / stored as UTC | Avoids local-timezone ambiguity in a tool whose evidence/audit trail is the whole point. |
| JSON fields (template definitions, dataset parameters) | Stored as `TEXT` holding JSON, validated at the application layer | Keeps the schema portable to SQLite (no native `JSONB`); exact structure remains a deferred design decision. |

Everything else below is a direct, traceable translation of Domain Model v0.3 Sec 4–9.

## 3. Conventions

- All primary keys: `id TEXT PRIMARY KEY` (UUID text).
- All foreign keys default to `ON DELETE RESTRICT` — nothing here cascades-deletes, per Domain Model Sec 15 ("archive/retire, don't delete").
- Enum-like columns are implemented as `TEXT` with a `CHECK (... IN (...))` constraint rather than native `ENUM` types, so the schema ports cleanly to SQLite without modification.
- `created_at` / `updated_at` default to `now()` where applicable (Postgres); SQLite equivalent noted in Sec 12.
- Every table name is plural, `snake_case`, matching Domain Model Sec 5's proposed table list exactly.

## 4. Project and Template Tables

```sql
CREATE TABLE projects (
    id                  TEXT PRIMARY KEY,
    project_code        TEXT NOT NULL UNIQUE,
    name                TEXT NOT NULL,
    description         TEXT,
    status              TEXT NOT NULL DEFAULT 'ACTIVE'
                             CHECK (status IN ('ACTIVE','ARCHIVED')),
    created_by          TEXT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Future multi-user readiness (Domain Model Sec 16) — may hold only the
-- owner in V1, but the table exists so no later migration is needed
-- just to introduce the concept.
CREATE TABLE project_users (
    project_id          TEXT NOT NULL REFERENCES projects(id),
    user_identity        TEXT NOT NULL,
    role                TEXT NOT NULL,
    added_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (project_id, user_identity)
);

CREATE TABLE tcs_templates (
    id                  TEXT PRIMARY KEY,
    project_id          TEXT NOT NULL REFERENCES projects(id),
    name                TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'ACTIVE'
                             CHECK (status IN ('ACTIVE','RETIRED')),
    current_revision_id TEXT REFERENCES tcs_template_revisions(id)
                             DEFERRABLE INITIALLY DEFERRED
);

CREATE TABLE tcs_template_revisions (
    id                  TEXT PRIMARY KEY,
    template_id         TEXT NOT NULL REFERENCES tcs_templates(id),
    revision_number     INTEGER NOT NULL,
    definition          TEXT NOT NULL,      -- JSON, application-validated
    created_by          TEXT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    change_summary      TEXT,
    UNIQUE (template_id, revision_number)
);
```

> `current_revision_id` on `tcs_templates` and `tcs_template_revisions.template_id` form a circular reference at creation time; the `DEFERRABLE` FK (or, on SQLite, a two-step insert) resolves this. Alternative: drop `current_revision_id` entirely and derive "current" as `MAX(revision_number)` — simpler, and recommended unless the application layer needs O(1) lookup of the current revision. **Open question for implementation, not a design gap.**

## 5. TCS / ATC / Dataset Lineage

Implements Domain Model Sec 4.4–4.6 and the immutability guarantee of Sec 2 Principle 2/5.

```sql
CREATE TABLE tcs_revisions (
    id                    TEXT PRIMARY KEY,
    project_id            TEXT NOT NULL REFERENCES projects(id),
    template_revision_id  TEXT NOT NULL REFERENCES tcs_template_revisions(id),
    source_file_name      TEXT NOT NULL,
    source_file_hash      TEXT NOT NULL,
    revision_number       INTEGER NOT NULL,
    imported_by           TEXT NOT NULL,
    imported_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    status                TEXT NOT NULL DEFAULT 'ACTIVE'
                               CHECK (status IN ('ACTIVE','RETIRED')),
    UNIQUE (project_id, revision_number)
);

-- Immutable once created (Domain Model Sec 4.5 "Immutability" note).
-- No UPDATE statements should ever target title/objective/preconditions/
-- expected_result on an existing row after initial INSERT; enforce this
-- at the application layer (a DB-level immutability trigger is optional
-- hardening, not required for V1).
CREATE TABLE atc_revisions (
    id                  TEXT PRIMARY KEY,
    tcs_revision_id     TEXT NOT NULL REFERENCES tcs_revisions(id),
    atc_key             TEXT NOT NULL,
    revision_number     INTEGER NOT NULL,
    title               TEXT NOT NULL,
    objective           TEXT,
    preconditions       TEXT,
    expected_result     TEXT,
    status              TEXT NOT NULL DEFAULT 'ACTIVE'
                             CHECK (status IN ('ACTIVE','RETIRED')),
    created_by          TEXT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tcs_revision_id, atc_key, revision_number)
);

-- parameters may contain credential-like values — Domain Model Sec 4.6
-- open item #11 (masking/encryption) is NOT resolved here; stored as
-- plain TEXT/JSON pending that decision.
CREATE TABLE atc_datasets (
    id                  TEXT PRIMARY KEY,
    atc_revision_id     TEXT NOT NULL REFERENCES atc_revisions(id),
    dataset_key         TEXT NOT NULL,
    name                TEXT NOT NULL,
    parameters          TEXT NOT NULL,      -- JSON, application-validated
    revision_number     INTEGER NOT NULL,
    status              TEXT NOT NULL DEFAULT 'ACTIVE'
                             CHECK (status IN ('ACTIVE','RETIRED')),
    UNIQUE (atc_revision_id, dataset_key, revision_number)
);
```

## 6. Application, Module, Target

Implements Domain Model Sec 4.7–4.11, including the `Module`/`module_type` fix from v0.3 that gives firmware/software components a proper home.

```sql
CREATE TABLE applications (
    id                  TEXT PRIMARY KEY,
    project_id          TEXT NOT NULL REFERENCES projects(id),
    name                TEXT NOT NULL,
    executable_name     TEXT,
    description         TEXT,
    UNIQUE (project_id, name)
);

CREATE TABLE application_versions (
    id                  TEXT PRIMARY KEY,
    application_id      TEXT NOT NULL REFERENCES applications(id),
    version             TEXT NOT NULL,
    build_number        TEXT,
    artifact_hash       TEXT,
    notes               TEXT,
    UNIQUE (application_id, version, build_number)
);

CREATE TABLE modules (
    id                  TEXT PRIMARY KEY,
    project_id          TEXT NOT NULL REFERENCES projects(id),
    parent_module_id    TEXT REFERENCES modules(id),   -- nullable, top-level
    name                TEXT NOT NULL,
    module_code         TEXT NOT NULL,
    module_type         TEXT NOT NULL
                             CHECK (module_type IN ('HARDWARE','FIRMWARE','SOFTWARE')),
    UNIQUE (project_id, module_code)
);

CREATE TABLE module_versions (
    id                  TEXT PRIMARY KEY,
    module_id           TEXT NOT NULL REFERENCES modules(id),
    version             TEXT NOT NULL,
    build_number        TEXT,
    artifact_hash       TEXT,
    notes               TEXT,
    UNIQUE (module_id, version, build_number)
);

CREATE TABLE targets (
    id                  TEXT PRIMARY KEY,
    project_id          TEXT NOT NULL REFERENCES projects(id),
    name                TEXT NOT NULL,
    target_type         TEXT NOT NULL,
    host_identifier      TEXT,
    network_identifier   TEXT,
    notes               TEXT
);
```

## 7. Object Mapping

Implements Domain Model Sec 4.12–4.13, including the `application_version_id` diagnostic link added in v0.2.

```sql
CREATE TABLE object_mappings (
    id                    TEXT PRIMARY KEY,
    project_id            TEXT NOT NULL REFERENCES projects(id),
    atc_key               TEXT NOT NULL,
    logical_object_key     TEXT NOT NULL,
    adapter_type          TEXT NOT NULL
                               CHECK (adapter_type IN ('WINDOWS','EMBEDDED_LINUX_QT')),
    current_revision_id    TEXT REFERENCES mapping_revisions(id)
                               DEFERRABLE INITIALLY DEFERRED,
    UNIQUE (project_id, atc_key, logical_object_key, adapter_type)
);

CREATE TABLE mapping_revisions (
    id                    TEXT PRIMARY KEY,
    mapping_id            TEXT NOT NULL REFERENCES object_mappings(id),
    revision_number       INTEGER NOT NULL,
    target_locator         TEXT NOT NULL,      -- JSON, adapter-specific
    identification_method  TEXT NOT NULL,
    confidence_state       TEXT NOT NULL
                               CHECK (confidence_state IN
                                   ('STRONG','CANDIDATE','UNRESOLVED','MANUAL_CONFIRMED')),
    application_version_id TEXT REFERENCES application_versions(id),
    validation_notes       TEXT,
    validated_by           TEXT,
    validated_at           TIMESTAMPTZ,
    UNIQUE (mapping_id, revision_number)
);
```

## 8. Test Configuration

Implements Domain Model Sec 4.14–4.15.

```sql
CREATE TABLE test_configurations (
    id                    TEXT PRIMARY KEY,
    project_id            TEXT NOT NULL REFERENCES projects(id),
    name                  TEXT NOT NULL,
    description           TEXT,
    application_id         TEXT NOT NULL REFERENCES applications(id),
    application_version_id TEXT NOT NULL REFERENCES application_versions(id),
    target_id             TEXT NOT NULL REFERENCES targets(id),
    adapter_type          TEXT NOT NULL
                               CHECK (adapter_type IN ('WINDOWS','EMBEDDED_LINUX_QT')),
    status                TEXT NOT NULL DEFAULT 'ACTIVE'
                               CHECK (status IN ('ACTIVE','RETIRED')),
    UNIQUE (project_id, name)
);

CREATE TABLE test_configuration_modules (
    test_configuration_id  TEXT NOT NULL REFERENCES test_configurations(id),
    module_id              TEXT NOT NULL REFERENCES modules(id),
    module_version_id       TEXT NOT NULL REFERENCES module_versions(id),
    role                   TEXT,
    PRIMARY KEY (test_configuration_id, module_id)
);
```

## 9. Test Run — Lifecycle, Lineage, Outcome

Implements State Machine v0.4 Sec 3–4 directly. This is the table where the safety-critical State/Outcome separation lives.

```sql
CREATE TABLE test_runs (
    id                  TEXT PRIMARY KEY,
    project_id          TEXT NOT NULL REFERENCES projects(id),
    parent_test_run_id   TEXT REFERENCES test_runs(id),   -- State Machine Sec 8/13
    run_name            TEXT NOT NULL,
    run_number          INTEGER NOT NULL,
    status              TEXT NOT NULL DEFAULT 'DRAFT'
                             CHECK (status IN (
                                 'DRAFT','READY_FOR_VALIDATION','VALIDATION_FAILED',
                                 'READY','RUNNING','PAUSED',
                                 'COMPLETED','STOPPED','INTERRUPTED')),
    outcome             TEXT
                             CHECK (outcome IS NULL OR outcome IN (
                                 'PASS','FAIL','BLOCKED','PARTIAL','NOT_EVALUATED')),
    adapter_type        TEXT NOT NULL
                             CHECK (adapter_type IN ('WINDOWS','EMBEDDED_LINUX_QT')),
    created_by          TEXT NOT NULL,
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, run_number),

    -- State Machine Sec 4: outcome is set exactly when (and only when)
    -- the run has reached a terminal status.
    CONSTRAINT outcome_set_iff_terminal CHECK (
        (status IN ('COMPLETED','STOPPED','INTERRUPTED') AND outcome IS NOT NULL)
        OR
        (status NOT IN ('COMPLETED','STOPPED','INTERRUPTED') AND outcome IS NULL)
    )
);

-- State Machine Sec 7: every validation cycle is recorded, not just the latest.
CREATE TABLE test_run_validations (
    id                  TEXT PRIMARY KEY,
    test_run_id          TEXT NOT NULL REFERENCES test_runs(id),
    validated_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    validator_version     TEXT NOT NULL,
    result               TEXT NOT NULL CHECK (result IN ('PASSED','FAILED')),
    findings             TEXT NOT NULL,      -- JSON: structured blocking/non-blocking list
    superseded           BOOLEAN NOT NULL DEFAULT false
);

CREATE TABLE test_run_configuration_snapshots (
    id                          TEXT PRIMARY KEY,
    test_run_id                  TEXT NOT NULL REFERENCES test_runs(id) UNIQUE,
    source_test_configuration_id  TEXT REFERENCES test_configurations(id),
    source_application_version_id TEXT REFERENCES application_versions(id),
    source_target_id              TEXT REFERENCES targets(id),
    application_name              TEXT NOT NULL,
    application_version           TEXT NOT NULL,
    target_snapshot               TEXT NOT NULL,   -- JSON
    adapter_type                  TEXT NOT NULL,
    adapter_version                TEXT,
    environment_snapshot           TEXT,             -- JSON
    created_at                    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE test_run_module_snapshots (
    id                  TEXT PRIMARY KEY,
    snapshot_id          TEXT NOT NULL REFERENCES test_run_configuration_snapshots(id),
    module_id            TEXT REFERENCES modules(id),
    module_name          TEXT NOT NULL,
    module_version        TEXT NOT NULL,
    role                 TEXT
);
```

**Why `outcome_set_iff_terminal` matters:** this single constraint is the database-level enforcement of the entire State/Outcome separation principle (State Machine Sec 4; Domain Model Design Principle 12). It makes it structurally impossible to insert a row where a `RUNNING` run already has a computed `outcome`, or a `COMPLETED` run has none — both of which would silently reintroduce the exact ambiguity that took three review rounds to design out.

## 10. Test Run Selection — ATCs, Datasets, Mapping Resolutions

Implements Domain Model Sec 4.19–4.20a, including the zero-dataset placeholder pattern and the dataset snapshot fix from the latest revision.

```sql
CREATE TABLE test_run_atcs (
    id                  TEXT PRIMARY KEY,
    test_run_id          TEXT NOT NULL REFERENCES test_runs(id),
    atc_revision_id       TEXT NOT NULL REFERENCES atc_revisions(id),
    sequence_no          INTEGER NOT NULL,
    selection_status      TEXT NOT NULL DEFAULT 'SELECTED'
                             CHECK (selection_status IN ('SELECTED','DESELECTED')),
    UNIQUE (test_run_id, sequence_no)
);

CREATE TABLE test_run_datasets (
    id                      TEXT PRIMARY KEY,
    test_run_atc_id          TEXT NOT NULL REFERENCES test_run_atcs(id),
    dataset_id              TEXT REFERENCES atc_datasets(id),   -- nullable
    is_default_placeholder    BOOLEAN NOT NULL DEFAULT false,
    dataset_snapshot          TEXT,      -- JSON; NULL iff is_default_placeholder
    dataset_snapshot_hash      TEXT,
    sequence_no              INTEGER NOT NULL,
    selection_status          TEXT NOT NULL DEFAULT 'SELECTED'
                                 CHECK (selection_status IN ('SELECTED','DESELECTED')),
    UNIQUE (test_run_atc_id, sequence_no),

    -- Domain Model Sec 4.20: exactly one of (real dataset + snapshot)
    -- or (placeholder, no dataset) — never both, never neither.
    CONSTRAINT dataset_or_placeholder CHECK (
        (is_default_placeholder = false AND dataset_id IS NOT NULL AND dataset_snapshot IS NOT NULL)
        OR
        (is_default_placeholder = true AND dataset_id IS NULL AND dataset_snapshot IS NULL)
    )
);

-- State Machine Sec 5.2 / Domain Model Sec 4.20a: an in-run CANDIDATE
-- confirmation is run-scoped and never mutates the canonical mapping_revisions row.
CREATE TABLE test_run_mapping_resolutions (
    id                    TEXT PRIMARY KEY,
    test_run_id            TEXT NOT NULL REFERENCES test_runs(id),
    mapping_id             TEXT NOT NULL REFERENCES object_mappings(id),
    mapping_revision_id     TEXT NOT NULL REFERENCES mapping_revisions(id),
    resolved_confidence     TEXT NOT NULL
                               CHECK (resolved_confidence IN ('MANUAL_CONFIRMED','REJECTED')),
    resolved_by             TEXT NOT NULL,
    resolved_at             TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolution_reason       TEXT
);
```

## 11. Execution — Attempts, Steps, Failures, Evidence

This is the safety-critical core: implements State Machine v0.4 Sec 5–6, 9, and I-01 through I-12 as literal database constraints, not just application logic.

```sql
CREATE TABLE execution_attempts (
    id                    TEXT PRIMARY KEY,
    test_run_dataset_id    TEXT NOT NULL REFERENCES test_run_datasets(id),
    attempt_number         INTEGER NOT NULL,
    status                TEXT NOT NULL DEFAULT 'NOT_EXECUTED'
                               CHECK (status IN (
                                   'NOT_EXECUTED','EXECUTING','PASS','FAIL',
                                   'BLOCKED','STOPPED','INTERRUPTED')),
    started_at             TIMESTAMPTZ,      -- nullable: see constraint below
    completed_at           TIMESTAMPTZ,
    executed_by             TEXT,
    execution_mode          TEXT
                               CHECK (execution_mode IS NULL OR execution_mode IN ('AUTOMATIC','MANUAL')),
    failure_summary         TEXT,
    UNIQUE (test_run_dataset_id, attempt_number),

    -- State Machine I-03: started_at is NULL only for the pre-execution
    -- BLOCKED path (mapping never resolved). Every other terminal state
    -- requires the Attempt to have actually entered EXECUTING.
    CONSTRAINT started_at_required_unless_pre_execution_blocked CHECK (
        started_at IS NOT NULL
        OR status IN ('NOT_EXECUTED', 'BLOCKED')
        -- Note: a BLOCKED row with started_at NULL is only valid via the
        -- mapping-resolution gate; enforced jointly with the Failure table's
        -- category = 'MAPPING_EXCEPTION' at the application layer, since a
        -- portable cross-table CHECK isn't expressible here. See Sec 11.1.
    ),

    -- State Machine I-10 (bounded case): a completed Attempt cannot be
    -- PASS unless it actually executed something.
    CONSTRAINT pass_requires_execution CHECK (
        status != 'PASS' OR started_at IS NOT NULL
    )
);

CREATE TABLE step_results (
    id                    TEXT PRIMARY KEY,
    attempt_id             TEXT NOT NULL REFERENCES execution_attempts(id),
    step_sequence          INTEGER NOT NULL,
    action_type            TEXT NOT NULL,
    target_reference        TEXT,
    mapping_revision_id     TEXT REFERENCES mapping_revisions(id),   -- nullable
    resolved_confidence     TEXT
                               CHECK (resolved_confidence IS NULL OR resolved_confidence IN
                                   ('STRONG','MANUAL_CONFIRMED')),
    expected_result         TEXT,
    actual_result           TEXT,
    result_state            TEXT NOT NULL DEFAULT 'NOT_EXECUTED'
                               CHECK (result_state IN (
                                   'NOT_EXECUTED','PASS','FAIL','BLOCKED','STOPPED','INTERRUPTED')),
    started_at              TIMESTAMPTZ,
    completed_at            TIMESTAMPTZ,
    error_code               TEXT,
    error_message            TEXT,
    UNIQUE (attempt_id, step_sequence)
);

CREATE TABLE failures (
    id                    TEXT PRIMARY KEY,
    attempt_id             TEXT REFERENCES execution_attempts(id),     -- nullable
    step_result_id          TEXT REFERENCES step_results(id),           -- nullable
    category               TEXT NOT NULL
                               CHECK (category IN (
                                   'TEST_FAILURE','AUTOMATION_FAILURE','MAPPING_EXCEPTION',
                                   'COMMUNICATION_FAILURE','CONFIGURATION_FAILURE','ENVIRONMENT_FAILURE')),
    code                   TEXT,
    message                 TEXT NOT NULL,
    technical_details        TEXT,
    resolved                BOOLEAN NOT NULL DEFAULT false,
    resolution_notes         TEXT,

    -- Domain Model Sec 4.23: exactly one of attempt_id / step_result_id,
    -- never both, never neither. This replaces the old ambiguous "or" phrasing.
    CONSTRAINT exactly_one_failure_target CHECK (
        (attempt_id IS NOT NULL AND step_result_id IS NULL)
        OR
        (attempt_id IS NULL AND step_result_id IS NOT NULL)
    )
    -- Note: the State Machine I-04 rule ("FAIL only ever pairs with
    -- category = TEST_FAILURE") is a cross-table rule against
    -- execution_attempts.status and cannot be expressed as a single-table
    -- CHECK here. See Sec 11.1 for the enforcement strategy.
);

CREATE TABLE evidence (
    id                  TEXT PRIMARY KEY,
    project_id           TEXT NOT NULL REFERENCES projects(id),
    test_run_id           TEXT NOT NULL REFERENCES test_runs(id),
    attempt_id            TEXT NOT NULL REFERENCES execution_attempts(id),
    step_result_id         TEXT REFERENCES step_results(id),   -- nullable
    evidence_type         TEXT NOT NULL CHECK (evidence_type IN ('SCREENSHOT','ATTACHMENT')),
    file_path             TEXT NOT NULL,
    file_hash             TEXT NOT NULL,
    captured_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    description            TEXT
);
```

### 11.1 On enforcing FAIL ↔ TEST_FAILURE at the database level

`execution_attempts.status = 'FAIL'` should only ever coexist with a linked `failures` row where `category = 'TEST_FAILURE'` (State Machine I-04, Sec 5.3 — the core safety rule of the whole project). This is a **cross-table** rule, so it cannot be expressed as a single-table `CHECK` constraint in standard SQL. Two implementation options, in order of preference:

1. **Application-layer enforcement + a periodic integrity check query** (recommended for V1): the execution engine is the single place that writes `Attempt.status` and its `Failure.category` together, in the same transaction, so the invariant is naturally maintained by construction. A read-only integrity query (`SELECT * FROM execution_attempts a JOIN failures f ON f.attempt_id = a.id OR f.step_result_id IN (SELECT id FROM step_results WHERE attempt_id = a.id) WHERE a.status = 'FAIL' AND f.category != 'TEST_FAILURE'`) can run as a startup/backup-time sanity check.
2. **Database trigger** (`BEFORE INSERT/UPDATE` on `execution_attempts` or `failures`) that rejects the write if the pairing is violated — stronger guarantee, more implementation effort, engine-specific syntax (works differently in Postgres vs. SQLite).

**Recommendation for V1:** option 1. The single-user, single-execution-engine architecture (State Machine I-07/I-08) means there's exactly one code path that ever writes these fields, which is a much weaker assumption to break than it would be in a multi-writer system. Revisit option 2 if/when centralized multi-user deployment happens (Domain Model Sec 16).

## 12. Audit and Backup

Implements Domain Model Sec 4.25–4.26.

```sql
CREATE TABLE audit_events (
    id                  TEXT PRIMARY KEY,
    project_id           TEXT NOT NULL REFERENCES projects(id),
    user_identity         TEXT NOT NULL,
    "timestamp"          TIMESTAMPTZ NOT NULL DEFAULT now(),
    action               TEXT NOT NULL,
    entity_type           TEXT NOT NULL,
    entity_id             TEXT NOT NULL,
    before_value          TEXT,      -- JSON, optional
    after_value           TEXT,      -- JSON, optional
    details              TEXT
);

CREATE TABLE backup_records (
    id                  TEXT PRIMARY KEY,
    project_id           TEXT NOT NULL REFERENCES projects(id),
    backup_type          TEXT NOT NULL CHECK (backup_type IN ('MANUAL','SCHEDULED')),
    file_path            TEXT NOT NULL,
    file_hash            TEXT NOT NULL,
    created_by            TEXT NOT NULL,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    restore_status         TEXT CHECK (restore_status IS NULL OR restore_status IN
                               ('NOT_RESTORED','RESTORED','RESTORE_FAILED')),
    notes                TEXT
);
```

## 13. Indexes

Directly implements Domain Model Sec 14.

```sql
-- project_id on all project-owned tables (representative subset shown;
-- apply the same pattern to every table with a project_id column)
CREATE INDEX idx_tcs_templates_project        ON tcs_templates(project_id);
CREATE INDEX idx_tcs_revisions_project        ON tcs_revisions(project_id);
CREATE INDEX idx_applications_project         ON applications(project_id);
CREATE INDEX idx_modules_project              ON modules(project_id);
CREATE INDEX idx_targets_project              ON targets(project_id);
CREATE INDEX idx_object_mappings_project      ON object_mappings(project_id);
CREATE INDEX idx_test_configurations_project  ON test_configurations(project_id);
CREATE INDEX idx_test_runs_project            ON test_runs(project_id);
CREATE INDEX idx_evidence_project             ON evidence(project_id);
CREATE INDEX idx_audit_events_project         ON audit_events(project_id);
CREATE INDEX idx_backup_records_project       ON backup_records(project_id);

-- TCS/ATC revision lineage
CREATE INDEX idx_atc_revisions_tcs            ON atc_revisions(tcs_revision_id);
CREATE INDEX idx_atc_datasets_atc             ON atc_datasets(atc_revision_id);

-- test_run_id across execution-related tables
CREATE INDEX idx_test_run_validations_run     ON test_run_validations(test_run_id);
CREATE INDEX idx_test_run_mapping_res_run     ON test_run_mapping_resolutions(test_run_id);
CREATE INDEX idx_test_run_atcs_run            ON test_run_atcs(test_run_id);
CREATE INDEX idx_evidence_run                 ON evidence(test_run_id);

-- parent_test_run_id lineage lookups (State Machine Sec 8)
CREATE INDEX idx_test_runs_parent             ON test_runs(parent_test_run_id);

-- attempt_id on step results / evidence / failures
CREATE INDEX idx_step_results_attempt         ON step_results(attempt_id);
CREATE INDEX idx_evidence_attempt             ON evidence(attempt_id);
CREATE INDEX idx_failures_attempt             ON failures(attempt_id);
CREATE INDEX idx_failures_step_result          ON failures(step_result_id);

-- version lookup by application/module
CREATE INDEX idx_application_versions_app     ON application_versions(application_id);
CREATE INDEX idx_module_versions_module       ON module_versions(module_id);

-- mapping lookup by logical object key + adapter type
CREATE INDEX idx_object_mappings_lookup       ON object_mappings(project_id, atc_key, adapter_type);

-- audit lookup by entity/user/time
CREATE INDEX idx_audit_events_entity          ON audit_events(entity_type, entity_id);
CREATE INDEX idx_audit_events_user            ON audit_events(user_identity);
CREATE INDEX idx_audit_events_time            ON audit_events("timestamp");
```

Exact indexes will still be tuned after real volume/query-pattern data is available (Domain Model Sec 14) — this set covers every lookup pattern named in the requirements so far, not a final performance-tuned set.

## 14. SQLite Adaptation Notes

If SQLite is the chosen engine (likely, given V1's local-only deployment, Architecture Sec 7.1):

- Replace `TIMESTAMPTZ` with `TEXT` storing ISO-8601 UTC strings (e.g., `2026-08-22T10:15:00Z`); SQLite has no native timestamp type.
- Replace `now()` defaults with `DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))`.
- `DEFERRABLE INITIALLY DEFERRED` foreign keys are not supported — resolve the `current_revision_id` circular reference either by allowing a temporary `NULL` and a follow-up `UPDATE`, or by dropping `current_revision_id` in favor of `MAX(revision_number)` lookups (Sec 4's open question).
- `BOOLEAN` becomes `INTEGER` (0/1) by convention — SQLite has no native boolean type, though it accepts the keyword loosely.
- All `CHECK` constraints used here are supported as-is by SQLite.

## 15. What This Document Does Not Do

- Does not pick the final database engine (Sec 2).
- Does not specify the exact JSON structure of `parameters`, `definition`, `target_locator`, `target_snapshot`, `environment_snapshot`, or `findings` — those remain a deferred design decision (Domain Model Sec 17 #4).
- Does not resolve dataset credential masking/encryption (Domain Model Sec 17 #11) — `atc_datasets.parameters` and `test_run_datasets.dataset_snapshot` are plain text pending that decision.
- Does not implement the FAIL↔TEST_FAILURE cross-table trigger (Sec 11.1) — recommends application-layer enforcement for V1 with the reasoning stated.
- Does not include migration/versioning tooling (Domain Model Sec 17 #10) — this is a first-version schema, not a migration script.

## 16. Acceptance Questions for Review

1. Does every constraint here trace back to a specific, already-agreed rule in the Domain Model or State Machine, rather than introducing new behavior?
2. Does `outcome_set_iff_terminal` actually make it impossible to insert a non-terminal run with a computed outcome, or a terminal run without one?
3. Does `dataset_or_placeholder` correctly forbid both the "real dataset with no snapshot" and "placeholder with a snapshot" invalid states?
4. Is the FAIL↔TEST_FAILURE gap (Sec 11.1) an acceptable V1 risk given the single-writer architecture, or does it need the trigger now?
5. Are there any State Machine invariants (I-01 through I-12) that don't yet have a corresponding constraint or documented enforcement strategy here?

---

**Status:** SQL Schema (DDL) Draft v0.1 — ready for review.
**Next artifact after review:** Adapter Interface Contract.

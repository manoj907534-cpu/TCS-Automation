# TCS Automation — SQL Schema (DDL)

**Version:** 0.1 — Design Draft (revised in place per DDL-executability review; not bumped to v0.2)
**Status:** Working Draft
**Baseline:** Domain Model & Database Schema v0.3 / Test Run State Machine v0.4

**Revisions made within v0.1 (no version bump, per the standing decision to avoid churn on corrections):**
- **Fixed two circular foreign keys that would have failed to execute as written.** `tcs_templates.current_revision_id → tcs_template_revisions` and `object_mappings.current_revision_id → mapping_revisions` each referenced a table that didn't exist yet at `CREATE TABLE` time. Resolved by removing `current_revision_id` from both tables entirely — "current revision" is now derived by query (`ORDER BY revision_number DESC LIMIT 1`), which was already documented as the preferred alternative in the original draft (Sec 4). This is genuinely simpler, not just a workaround.
- **Added a trigger rejecting an Attempt transition to `PASS` unless at least one executed Step Result exists at that moment** (State Machine I-10, Sec 11.1). A first version of this trigger only checked that *a* Step Result row existed, not that any step was actually *executed* — a `NOT_EXECUTED` step row would have incorrectly satisfied it. Corrected to check `result_state IN ('PASS','FAIL','BLOCKED','STOPPED','INTERRUPTED')` explicitly.
- **Added a trigger rejecting an Attempt transition into pre-execution `BLOCKED`** unless a linked `MAPPING_EXCEPTION` failure exists and zero Step Results exist, at that moment (State Machine Sec 5.2, Sec 11.1).
- **Corrected the scope claimed for both triggers.** They validate the invariant *at the moment of the Attempt write* — not continuously against later, unrelated deletions of supporting `failures`/`step_results` rows (which don't re-fire an `execution_attempts` trigger). Sec 11.1 now states this explicitly, along with why it's a theoretical rather than practical gap given V1's append-only, single-writer execution engine.
- **Corrected an overclaim.** The original text said this document "implements I-01 through I-12 as literal database constraints." That wasn't accurate — several invariants (I-01 terminal immutability, I-07 one active Attempt, I-11/I-12 Test-Run/Attempt state consistency) are enforced transactionally in the execution engine, not by the schema itself. Section 11.3 now states plainly, invariant by invariant, which mechanism enforces what.
- **Fixed a real INSERT-order bug in Trigger 2.** It was `BEFORE INSERT OR UPDATE`, but a `MAPPING_EXCEPTION` failure row can never exist at the moment an Attempt is first `INSERT`ed (the FK it references doesn't exist yet) — so a direct `INSERT ... status = 'BLOCKED'` could never satisfy the trigger, by construction. Restricted to `BEFORE UPDATE` and added an explicit required write sequence note (Sec 11.1), which matches how Attempts are already created elsewhere in the design (always `NOT_EXECUTED` first, transitioned afterward).
- Documented that `updated_at`'s default only applies at `INSERT`; refreshing it on `UPDATE` is an application-layer responsibility (Sec 4).
- Flagged (not changed, pending confirmation) an assumption on `tcs_revisions.revision_number` scope — currently one evolving TCS series per project, not multiple independently-numbered TCS documents per project (Sec 5).

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
    -- updated_at's DEFAULT only applies at INSERT time; Postgres does not
    -- auto-refresh it on UPDATE. Maintaining it on every update is the
    -- application layer's responsibility (a trigger could do this too, but
    -- isn't necessary for a single column with no correctness implications).
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
                             CHECK (status IN ('ACTIVE','RETIRED'))
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

> **Resolved (was an open question in an earlier draft of this document):** `current_revision_id` is intentionally **not** a column on `tcs_templates`. The original design created a circular foreign key — `tcs_templates` would need to reference `tcs_template_revisions`, which itself references `tcs_templates`, so neither table could be created first with a plain `CREATE TABLE`. Rather than work around this with `ALTER TABLE`/deferred constraints (extra complexity, and not portable to SQLite), the "current" revision is simply derived by query: `SELECT * FROM tcs_template_revisions WHERE template_id = ? ORDER BY revision_number DESC LIMIT 1`. The same fix is applied to `object_mappings`/`mapping_revisions` below. If a later performance need justifies an O(1) lookup, that can be added with an `ALTER TABLE ... ADD CONSTRAINT ... DEFERRABLE` at that time — this is a pure implementation optimization, not a domain change.

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
```

> **Scope assumption, flagged for verification rather than guessed at:** `revision_number` is scoped `UNIQUE (project_id, revision_number)` — i.e., one project has a single evolving TCS series (re-importing the workbook produces revision 2, 3, ...), not multiple independently-numbered TCS "identities" within one project. This matches Domain Model Sec 4.4's description of a TCS Revision as "one imported/normalized version of **a** TCS" (singular, per project) and Sec 4.4's "the source workbook is not silently overwritten" framing, which implies one ongoing workbook per project rather than several. If a project is ever expected to import multiple *distinct* TCS documents independently (not just successive corrected versions of the same one), this table would need an additional `tcs_identity_id` grouping column before implementation — worth a quick confirmation against actual usage before this table is built, since it's cheap to fix now and awkward to fix after data exists.

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

> Same resolution as Sec 4: no `current_revision_id` on `object_mappings` — the current/authoritative revision is derived as `SELECT * FROM mapping_revisions WHERE mapping_id = ? ORDER BY revision_number DESC LIMIT 1`.

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

This is the safety-critical core: implements State Machine v0.4 Sec 5–6, 9. Coverage of I-01 through I-12 is **mixed** — some are enforced here as schema constraints/triggers, others remain the execution engine's responsibility. Section 11.3 gives the complete, honest accounting; don't assume every invariant is a database-level guarantee just because it's discussed in this section.

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
        -- This CHECK only rules out started_at being NULL for status values
        -- other than NOT_EXECUTED/BLOCKED. It does NOT by itself guarantee
        -- that a BLOCKED+NULL row is specifically the pre-execution
        -- MAPPING_EXCEPTION case with zero steps — that stronger guarantee
        -- is enforced by Trigger 2 in Sec 11.1, which this CHECK works
        -- alongside rather than replaces.
    ),

    -- State Machine I-10 (necessary but not sufficient): rules out the
    -- narrow case of PASS with started_at NULL. Does NOT by itself rule out
    -- PASS with zero Step Results despite a non-NULL started_at — that
    -- fuller guarantee is enforced by Trigger 1 in Sec 11.1.
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

### 11.1 Triggers: closing the two gaps a single-table CHECK can't reach

Two State Machine invariants need a child-row lookup, which a single-table `CHECK` constraint cannot express. Both are safety-critical enough to enforce at the database level rather than leave to application discipline alone.

**Trigger 1 — rejects an Attempt transition to `PASS` unless at least one actually-*executed* Step Result exists at the moment of that transition (State Machine I-10).**

```sql
CREATE OR REPLACE FUNCTION enforce_pass_requires_step() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.status = 'PASS' AND NOT EXISTS (
        SELECT 1 FROM step_results
        WHERE attempt_id = NEW.id
          -- Explicit allow-list, not "<> NOT_EXECUTED": a future new
          -- Step Result state (e.g. a hypothetical SKIPPED) should not
          -- silently count as "executed" just by omission from an
          -- exclusion list.
          AND result_state IN ('PASS','FAIL','BLOCKED','STOPPED','INTERRUPTED')
    ) THEN
        RAISE EXCEPTION
            'execution_attempts.id=% cannot be PASS with zero executed Step Results (State Machine I-10)',
            NEW.id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_pass_requires_step
    BEFORE INSERT OR UPDATE ON execution_attempts
    FOR EACH ROW EXECUTE FUNCTION enforce_pass_requires_step();
```

**Trigger 2 — rejects an Attempt transition into pre-execution `BLOCKED` (`started_at IS NULL`) unless the required `MAPPING_EXCEPTION` failure and zero-step conditions are both present at the moment of that transition (State Machine Sec 5.2).**

```sql
CREATE OR REPLACE FUNCTION enforce_pre_execution_blocked_is_mapping_exception() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.status = 'BLOCKED' AND NEW.started_at IS NULL THEN
        IF NOT EXISTS (
            SELECT 1 FROM failures
            WHERE attempt_id = NEW.id
              AND category = 'MAPPING_EXCEPTION'
        ) THEN
            RAISE EXCEPTION
                'execution_attempts.id=% is BLOCKED with started_at NULL but has no linked MAPPING_EXCEPTION failure (State Machine Sec 5.2)',
                NEW.id;
        END IF;
        IF EXISTS (SELECT 1 FROM step_results WHERE attempt_id = NEW.id) THEN
            RAISE EXCEPTION
                'execution_attempts.id=% is BLOCKED with started_at NULL but has Step Results — pre-execution BLOCKED must have zero steps (State Machine I-03)',
                NEW.id;
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_pre_execution_blocked_check
    BEFORE UPDATE ON execution_attempts
    FOR EACH ROW EXECUTE FUNCTION enforce_pre_execution_blocked_is_mapping_exception();
```

**Pre-execution BLOCKED write sequence.** This trigger is deliberately `BEFORE UPDATE` only, not `BEFORE INSERT OR UPDATE`. Because `failures.attempt_id` is a foreign key against `execution_attempts.id`, a `MAPPING_EXCEPTION` failure row cannot possibly exist yet at the moment an Attempt is first `INSERT`ed — the referenced Attempt wouldn't exist. A direct single-statement `INSERT ... status = 'BLOCKED'` could therefore never satisfy this trigger's precondition, by construction, regardless of correctness. This isn't actually a new constraint on the execution engine: Sec 5.1 already establishes that every `execution_attempts` row is created `NOT_EXECUTED` when the Test Run enters `RUNNING`, and only *transitions* to other states afterward. The real write sequence is:

```
1. INSERT execution_attempts row, status = 'NOT_EXECUTED'  (at Test Run prep time)
2. INSERT failures row, category = 'MAPPING_EXCEPTION', attempt_id = <that Attempt>
3. UPDATE execution_attempts SET status = 'BLOCKED' WHERE id = <that Attempt>
```

Step 3 is what the trigger validates. This should be treated as a required implementation note for whoever builds the repository/data-access layer, not just documentation color.

**What these triggers do and don't guarantee.** Both reject an invalid `execution_attempts` write at the moment that write happens — that's a real, enforced guarantee, and it's the one that matters for the normal execution path (an Attempt reaches `PASS`/pre-execution-`BLOCKED` exactly once, at the end of its one execution). What they do **not** do is continuously re-validate the invariant against later, unrelated mutations to `step_results` or `failures` — a subsequent `DELETE` against a supporting `step_results`/`failures` row is a write to a *different* table and does not re-fire these triggers. In the single-writer V1 architecture, the execution engine is also the only code path that ever deletes historical `step_results`/`failures` rows, and it has no reason to do so (Sec 2, Domain Model Design Principle 2: historical data is append-oriented, never deleted in normal operation) — so this is a theoretical gap given V1's actual write patterns, not a practical one. Preserving supporting rows after an Attempt reaches a terminal state is the execution engine's responsibility, the same way I-01/I-06 (terminal immutability) already are (Sec 11.3). On SQLite, the equivalent is `CREATE TRIGGER ... BEFORE INSERT/UPDATE ... WHEN ... BEGIN SELECT RAISE(ABORT, '...') WHERE ...; END;` — same logic, different syntax.

### 11.2 On `FAIL` ↔ `TEST_FAILURE` specifically

`execution_attempts.status = 'FAIL'` should only ever coexist with a linked `failures` row where `category = 'TEST_FAILURE'` (State Machine I-04, Sec 5.3). Unlike the two gaps above, this one is **not** given a trigger here, for a specific reason: the correct write pattern is a single transaction where the execution engine writes the `Step Result`, the `Failure` row, and the `Attempt.status` update together, atomically (State Machine Sec 12) — there is exactly one code path in V1 that ever performs this write (I-07/I-08: one active Attempt, one execution engine). A trigger would be redundant hardening against a failure mode (a second, buggy writer) that V1's single-writer architecture doesn't have. This is deliberately different from Trigger 1/2 above, where the risk was an *incomplete* write (a valid-looking row missing its supporting evidence), not a *conflicting* write — the trigger pattern used there doesn't map as cleanly here. Revisit if/when centralized multi-user deployment (Domain Model Sec 16) introduces more than one writer.

### 11.3 Honest accounting: how each State Machine invariant is actually enforced

An earlier draft of this document claimed I-01 through I-12 were implemented "as literal database constraints." That overstated it. Here is the accurate picture:

| Invariant | Enforcement mechanism |
|---|---|
| I-01 — Terminal immutability | **Application-layer only.** No `CHECK`/trigger currently prevents an `UPDATE` on an already-terminal row. Left to the execution engine's atomic compare-and-swap writes (State Machine Sec 12) recognizing a terminal current state and refusing to transition further. A `BEFORE UPDATE` trigger rejecting any status change away from a terminal value is a reasonable V1.1 hardening step, not required to ship. |
| I-02 — No execution after terminal | Application-layer, same mechanism as I-01. |
| I-03 — No terminal outcome without execution (bounded exception) | **Partially DB-enforced.** `pass_requires_execution` (`CHECK`) + Trigger 1 (Sec 11.1) together cover the `PASS` side. Trigger 2 covers the pre-execution `BLOCKED` exception itself. |
| I-04 — FAIL requires TEST_FAILURE | **Application-layer**, by design — see Sec 11.2. |
| I-05 — BLOCKED excludes product defect | Application-layer (same code path that enforces I-04 also governs this — a step is only ever written as `BLOCKED` by the execution engine's non-`TEST_FAILURE` branch). |
| I-06 — Historical immutability | Same as I-01 — application-layer. |
| I-07 — At most one active Attempt per Test Run | **Application-layer.** Not currently a DB constraint; nothing stops two `execution_attempts` rows under the same `test_run_dataset_id`/Test Run from both being `EXECUTING` at the database level. A partial unique index (`CREATE UNIQUE INDEX ... ON execution_attempts (test_run_id_derived) WHERE status = 'EXECUTING'`, requiring a denormalized `test_run_id` column or a view) is a viable future hardening step but adds schema complexity not taken on in V1. |
| I-08 — One active Test Run per Test PC | Application-layer / OS-level (single installation, no server to arbitrate). |
| I-09 — Snapshot immutability | `test_run_configuration_snapshots` rows are only ever inserted, never updated, by convention — no `CHECK` prevents an `UPDATE` today. Same category as I-01. |
| I-10 — Attempt aggregation algorithm | **DB-enforced at transition time for the "never PASS with zero executed steps" half** (Trigger 1, checking `result_state IN (...)` explicitly, not just row existence). The full step-priority aggregation itself (`INTERRUPTED` > `STOPPED` > `FAIL` > `BLOCKED` > `PASS`) is application-layer — it's the execution engine's job to *compute* the right status, not the database's job to *re-derive* it from steps after the fact. |
| I-11 — No orphan execution state | Application-layer. |
| I-12 — Pause consistency | Application-layer. |

**Why this table matters:** roughly half of these invariants are enforced by the schema itself, and half are enforced by the execution engine's transactional discipline. That's a normal, reasonable split for a single-writer V1 system — but it needs to be stated plainly (as it now is) rather than implied to be stronger than it is. Anyone implementing the execution engine should treat this table as their checklist of *which* invariants they are personally responsible for maintaining, since the database won't catch a violation for them.

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
- Replace the two `plpgsql` triggers in Sec 11.1 with SQLite's `CREATE TRIGGER ... BEFORE INSERT/UPDATE ON ... WHEN <condition> BEGIN SELECT RAISE(ABORT, '<message>') WHERE EXISTS (...); END;` syntax — same rejection logic, different trigger dialect.
- `BOOLEAN` becomes `INTEGER` (0/1) by convention — SQLite has no native boolean type, though it accepts the keyword loosely.
- All `CHECK` constraints used here are supported as-is by SQLite. `RAISE EXCEPTION` becomes `RAISE(ABORT, ...)` inside triggers.

## 15. What This Document Does Not Do

- Does not pick the final database engine (Sec 2).
- Does not specify the exact JSON structure of `parameters`, `definition`, `target_locator`, `target_snapshot`, `environment_snapshot`, or `findings` — those remain a deferred design decision (Domain Model Sec 17 #4).
- Does not resolve dataset credential masking/encryption (Domain Model Sec 17 #11) — `atc_datasets.parameters` and `test_run_datasets.dataset_snapshot` are plain text pending that decision.
- Does not add a trigger for FAIL↔TEST_FAILURE (Sec 11.2) — deliberately left to the single-writer execution engine's transactional discipline, with the reasoning stated for why that's a reasonable V1 line to draw.
- Does not add database-level enforcement for I-01/I-02/I-06/I-07/I-08/I-09/I-11/I-12 — Sec 11.3 states plainly that these remain the execution engine's responsibility in V1.
- Does not include migration/versioning tooling (Domain Model Sec 17 #10) — this is a first-version schema, not a migration script.

## 16. Acceptance Questions for Review

1. Does every constraint here trace back to a specific, already-agreed rule in the Domain Model or State Machine, rather than introducing new behavior?
2. Does `outcome_set_iff_terminal` actually make it impossible to insert a non-terminal run with a computed outcome, or a terminal run without one?
3. Does `dataset_or_placeholder` correctly forbid both the "real dataset with no snapshot" and "placeholder with a snapshot" invalid states?
4. Do the two triggers in Sec 11.1 correctly check for *executed* Step Results / a genuine mapping exception at transition time, and is it clear they don't continuously re-validate against later mutations to supporting rows?
5. Is the FAIL↔TEST_FAILURE decision to rely on the single-writer execution engine (Sec 11.2), rather than a trigger, an acceptable V1 risk?
6. Does the Sec 11.3 table match reality — is there any invariant marked "application-layer" that actually does have schema-level protection somewhere in this document, or vice versa?
7. Does removing `current_revision_id` (Sec 4/7) create any query-performance concern significant enough to reconsider it before implementation, given V1's stated volume targets (Domain Model NFR-003: ~10,000 records/project)?

---

**Status:** SQL Schema (DDL) v0.1 — Approved Baseline. All identified issues resolved: circular FKs removed, both safety-critical triggers checked for correctness (Trigger 1's execution-state check, Trigger 2's INSERT/UPDATE timing), and one scope assumption flagged for a quick confirmation rather than silently guessed at.
**Next artifact:** Adapter Interface Contract.

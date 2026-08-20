# TCS Automation — Domain Model & Database Schema

**Version:** 0.2 — Design Draft
**Status:** Working Draft
**Baseline:** Architecture v0.2 / SRS v1.2

**Changelog since v0.1:**
- Split `Hardware Module` into a generic `Module` entity with an explicit `module_type` (`HARDWARE` / `FIRMWARE` / `SOFTWARE`) so software/firmware-only components have a proper home (previously `Module Version` claimed hardware/software/firmware coverage while its parent entity was scoped to hardware only).
- Clarified `Execution Attempt` linkage for ATCs with zero datasets — a default dataset row is created at Test Run preparation time so the attempt FK always resolves.
- Replaced the ambiguous `Failure.attempt_id or step_result_id` phrasing with two explicit nullable FK columns and a check constraint.
- Corrected the Result State diagram so `BLOCKED` branches directly from `EXECUTING`, parallel to `PASS`/`FAIL`, instead of appearing nested under `FAIL`.
- Added `Execution Attempt.status` enum values.
- Added `Mapping Revision.application_version_id` (optional) to support diagnosing recovery failures across app versions.
- Flagged dataset credential handling as an explicit open item (Sec 17).

## 1. Purpose

This document defines the technology-independent domain model and proposed relational database schema for TCS Automation V1.

The primary goals are:

- Preserve complete Test Run traceability.
- Keep historical records immutable where required.
- Support TCS → ATC → Dataset → Execution relationships.
- Support application and hardware/software version combinations.
- Support mapping revisions and future TCS corrections without corrupting previous runs.
- Support repeat execution without overwriting previous attempts.
- Support project-level backup and restore.
- Keep the model suitable for future centralized/multi-user deployment.

## 2. Design Principles

1. Every persistent business entity has a stable ID.
2. Historical execution data is append-oriented.
3. A Test Run owns an immutable configuration snapshot.
4. Editable configuration never changes an existing Test Run snapshot.
5. Re-running an ATC creates a new execution attempt.
6. Dataset results are independently traceable.
7. Mapping revisions are preserved.
8. Version records are reusable but the exact versions used by a run are frozen in its snapshot.
9. Database writes for Test Run/Attempt/Result state are transactional.
10. Deleting business data is not the normal mechanism for correcting history; corrections create new revisions.
11. Every ATC has an execution linkage path even when it defines no datasets (see 4.20).

## 3. Conceptual Domain Model

```
PROJECT
  │
  ├── TCS TEMPLATE
  │     └── TEMPLATE REVISION
  │            └── TCS REVISION
  │                   └── ATC REVISION
  │                          └── DATASET (zero, one, or many)
  │
  ├── APPLICATION
  │     └── APPLICATION VERSION
  │
  ├── MODULE (hardware / firmware / software)
  │     └── MODULE VERSION
  │
  ├── TARGET
  │
  ├── OBJECT MAPPING
  │     └── MAPPING REVISION
  │
  ├── TEST CONFIGURATION
  │
  └── TEST RUN
         │
         ├── TEST RUN CONFIGURATION SNAPSHOT
         │      ├── APPLICATION VERSION SNAPSHOT
         │      ├── MODULE VERSION SNAPSHOT
         │      └── TARGET/ENVIRONMENT SNAPSHOT
         │
         ├── TEST RUN ATC
         │      └── DATASET SELECTION (default row if ATC has no datasets)
         │
         └── EXECUTION ATTEMPT
                └── STEP RESULT
                       ├── EVIDENCE
                       └── FAILURE / EXCEPTION

AUDIT EVENT
BACKUP RECORD
```

## 4. Entity Definitions

### 4.1 Project

Represents an independent testing project.

Key fields:

- `project_id`
- `project_code`
- `name`
- `description`
- `status`
- `created_by`
- `created_at`
- `updated_at`

A project is the primary ownership boundary for TCS, ATCs, mappings, configurations, runs, evidence and backups.

### 4.2 TCS Template

Defines the expected `.xlsx` structure for a project.

Key fields:

- `template_id`
- `project_id`
- `name`
- `status`
- `current_revision_id`

### 4.3 TCS Template Revision

Immutable revision of a template definition.

Key fields:

- `template_revision_id`
- `template_id`
- `revision_number`
- `definition`
- `created_by`
- `created_at`
- `change_summary`

### 4.4 TCS Revision

Represents one imported/normalized version of a TCS.

Key fields:

- `tcs_revision_id`
- `project_id`
- `template_revision_id`
- `source_file_name`
- `source_file_hash`
- `revision_number`
- `imported_by`
- `imported_at`
- `status`

The source workbook is not silently overwritten. A corrected workbook creates a new revision.

### 4.5 ATC Revision

A versioned, technology-independent test case generated from or associated with a TCS revision.

Key fields:

- `atc_revision_id`
- `tcs_revision_id`
- `atc_key`
- `revision_number`
- `title`
- `objective`
- `preconditions`
- `expected_result`
- `status`
- `created_by`
- `created_at`

### 4.6 Dataset

Represents a parameter/data set associated with an ATC revision.

Key fields:

- `dataset_id`
- `atc_revision_id`
- `dataset_key`
- `name`
- `parameters`
- `revision_number`
- `status`

One ATC may have zero, one or many datasets. See 4.20 for how ATCs with zero datasets are executed.

> **Open item:** `parameters` may contain credential-like values (e.g., a login test's password). See Sec 17 #11 for the deferred decision on masking/encrypting these at rest and in exported reports.

### 4.7 Application

Represents the application under test, such as `SM1.exe`.

Key fields:

- `application_id`
- `project_id`
- `name`
- `executable_name`
- `description`

### 4.8 Application Version

Reusable application version record.

Key fields:

- `application_version_id`
- `application_id`
- `version`
- `build_number`
- `artifact_hash` (optional)
- `notes`

### 4.9 Module

Represents a hardware, firmware, or software module/submodule required by the system under test. Replaces the earlier "Hardware Module" entity so firmware- and software-only components (not tied to a physical card) have a proper home, consistent with SRS Sec 11 / FR-072.

Examples:

- Card-1, Card-2, Card-3, Card-4 (`module_type = HARDWARE`)
- Bootloader (`module_type = FIRMWARE`)
- Diagnostics service (`module_type = SOFTWARE`)

Key fields:

- `module_id`
- `project_id`
- `parent_module_id` (nullable for top-level module)
- `name`
- `module_code`
- `module_type` (`HARDWARE` | `FIRMWARE` | `SOFTWARE`)

This supports the requirement that a Main Module can contain hardware/software/firmware submodules.

### 4.10 Module Version

Reusable version record for a module (hardware, firmware, or software).

Key fields:

- `module_version_id`
- `module_id`
- `version`
- `build_number`
- `artifact_hash` (optional)
- `notes`

### 4.11 Target

Represents the test target/board/device.

Key fields:

- `target_id`
- `project_id`
- `name`
- `target_type`
- `host_identifier`
- `network_identifier` (as appropriate)
- `notes`

Credentials/secrets are not stored in this general entity.

### 4.12 Object Mapping

Logical mapping definition connecting an ATC target reference to an adapter-visible target object.

Key fields:

- `mapping_id`
- `project_id`
- `atc_key`
- `logical_object_key`
- `adapter_type`
- `current_revision_id`

### 4.13 Mapping Revision

Immutable mapping version.

Key fields:

- `mapping_revision_id`
- `mapping_id`
- `revision_number`
- `target_locator`
- `identification_method`
- `confidence_state`
- `application_version_id` (optional — the application version this mapping was validated against, to help diagnose recovery failures when multiple app versions are in play)
- `validation_notes`
- `validated_by`
- `validated_at`

Possible confidence states:

- `STRONG`
- `CANDIDATE`
- `UNRESOLVED`
- `MANUAL_CONFIRMED`

### 4.14 Test Configuration

Editable reusable environment definition.

Key fields:

- `test_configuration_id`
- `project_id`
- `name`
- `description`
- `application_id`
- `application_version_id`
- `target_id`
- `adapter_type`
- `status`

Associated module versions are stored through `test_configuration_modules`.

### 4.15 Test Configuration Module

Associates a Test Configuration with the exact selected reusable module versions.

Fields:

- `test_configuration_id`
- `module_id`
- `module_version_id`
- `role`

### 4.16 Test Run

Represents one named execution round/build verification session.

Key fields:

- `test_run_id`
- `project_id`
- `run_name`
- `run_number`
- `status`
- `created_by`
- `started_at`
- `completed_at`
- `adapter_type`
- `created_at`

### 4.17 Test Run Configuration Snapshot

Immutable copy created when the Test Run is prepared/frozen.

Key fields:

- `snapshot_id`
- `test_run_id`
- `source_test_configuration_id`
- `application_name`
- `application_version`
- `target_snapshot`
- `adapter_type`
- `adapter_version`
- `environment_snapshot`
- `created_at`

The snapshot must be self-contained enough to identify the environment used even if reusable version records later change.

### 4.18 Test Run Module Snapshot

Frozen module/version combination belonging to a Test Run snapshot.

Fields:

- `snapshot_id`
- `module_id`
- `module_name`
- `module_version`
- `role`

### 4.19 Test Run ATC

Associates a selected ATC revision with a Test Run.

Fields:

- `test_run_atc_id`
- `test_run_id`
- `atc_revision_id`
- `sequence_no`
- `selection_status`

### 4.20 Test Run Dataset

Associates a selected dataset with the Test Run ATC.

Fields:

- `test_run_dataset_id`
- `test_run_atc_id`
- `dataset_id` (nullable — see below)
- `is_default_placeholder` (boolean)
- `sequence_no`
- `selection_status`

**Zero-dataset ATCs:** if an ATC Revision defines no datasets, Test Run preparation creates exactly one `Test Run Dataset` row with `dataset_id = NULL` and `is_default_placeholder = true`. This guarantees every `Execution Attempt` always resolves through a `Test Run Dataset`, without requiring a second, optional FK path on `Execution Attempt`.

The selected dataset definition should be snapshotted or content-hashed so later dataset edits cannot alter historical meaning.

### 4.21 Execution Attempt

One actual execution attempt of a selected Test Run ATC/dataset.

Fields:

- `attempt_id`
- `test_run_dataset_id`
- `attempt_number`
- `status`
- `started_at`
- `completed_at`
- `executed_by`
- `execution_mode`
- `failure_summary`

`status` values:

- `NOT_EXECUTED`
- `EXECUTING`
- `PASS`
- `FAIL`
- `BLOCKED`
- `INTERRUPTED`

Each retry creates a new attempt.

**Interruption rule:** if a Test Run transitions to `INTERRUPTED` or `STOPPED` (Sec 9) while an attempt is `EXECUTING`, that attempt's `status` is set to `INTERRUPTED` (not silently left as `EXECUTING` and not auto-converted to `FAIL`). An `INTERRUPTED` attempt requires an explicit resume/re-run/new-run decision, per SRS FR-065.

### 4.22 Step Result

Result of one logical ATC step during one execution attempt.

Fields:

- `step_result_id`
- `attempt_id`
- `step_sequence`
- `action_type`
- `target_reference`
- `expected_result`
- `actual_result`
- `result_state`
- `started_at`
- `completed_at`
- `error_code`
- `error_message`

Result states:

- `PASS`
- `FAIL`
- `BLOCKED`
- `NOT_EXECUTED`

### 4.23 Failure / Exception

Structured record explaining why execution could not continue or produced a failure.

Fields:

- `failure_id`
- `attempt_id` (nullable)
- `step_result_id` (nullable)
- `category`
- `code`
- `message`
- `technical_details`
- `resolved`
- `resolution_notes`

**Constraint:** exactly one of `attempt_id` / `step_result_id` must be set — `attempt_id` for attempt-level failures (e.g., communication loss before any step ran), `step_result_id` for step-level failures. Enforced with a database check constraint, not left as an implicit "or".

Categories:

- `TEST_FAILURE`
- `AUTOMATION_FAILURE`
- `MAPPING_EXCEPTION`
- `COMMUNICATION_FAILURE`
- `CONFIGURATION_FAILURE`
- `ENVIRONMENT_FAILURE`

### 4.24 Evidence

Evidence associated with an execution event.

Fields:

- `evidence_id`
- `project_id`
- `test_run_id`
- `attempt_id`
- `step_result_id` (nullable)
- `evidence_type`
- `file_path`
- `file_hash`
- `captured_at`
- `description`

V1 primarily supports screenshots/attachments; video is out of scope.

### 4.25 Audit Event

Immutable audit record.

Fields:

- `audit_event_id`
- `project_id`
- `user_identity`
- `timestamp`
- `action`
- `entity_type`
- `entity_id`
- `before_value` (optional)
- `after_value` (optional)
- `details`

### 4.26 Backup Record

Records project-level backup/restore operations.

Fields:

- `backup_id`
- `project_id`
- `backup_type`
- `file_path`
- `file_hash`
- `created_by`
- `created_at`
- `restore_status`
- `notes`

## 5. Proposed Relational Tables

```
projects
project_users                         (future multi-user ready; V1 may contain only owner)

tcs_templates
tcs_template_revisions
tcs_revisions
atc_revisions
atc_datasets

applications
application_versions
modules
module_versions

targets

object_mappings
mapping_revisions

test_configurations
test_configuration_modules

test_runs
test_run_configuration_snapshots
test_run_module_snapshots
test_run_atcs
test_run_datasets

execution_attempts
step_results
failures

evidence

audit_events
backup_records
```

## 6. Relationship Rules

### Project ownership

Every project-owned entity must ultimately reference a project either directly or through a parent entity.

### TCS/ATC lineage

```
Template Revision
      ↓
TCS Revision
      ↓
ATC Revision
      ↓
Dataset (optional)
```

An execution record references the exact ATC revision/dataset (or default placeholder) selected for that Test Run.

### Configuration lineage

```
Editable Test Configuration
          ↓
    Run preparation
          ↓
Immutable Test Run Snapshot
          ↓
  Test Run Module Snapshots
```

### Execution lineage

```
Test Run
  ↓
Test Run ATC
  ↓
Test Run Dataset (real or default placeholder)
  ↓
Execution Attempt
  ↓
Step Results
  ↓
Evidence / Failure
```

This makes it possible to answer:
> Which application version, module versions, ATC revision, dataset, mapping and execution attempt produced this result?

## 7. Historical Integrity Rules

### Rule 1 — Never overwrite execution history

A completed execution attempt is immutable except for explicitly auditable metadata such as administrative annotations.

### Rule 2 — Corrections create revisions

If a TCS, ATC, dataset or mapping is corrected, create a new revision.

Existing Test Runs continue to point to the old revision.

### Rule 3 — Configuration changes do not affect old runs

Changing:

```
SM1.exe v2.4.1 → v2.4.2
Card-2 v3.04 → v3.05
```

must not alter any existing Test Run Snapshot.

### Rule 4 — Re-run means new attempt

```
ATC-100 / Dataset-01
  Attempt 1 → FAIL
  Attempt 2 → PASS
```

Both attempts remain stored.

### Rule 5 — Evidence is immutable

Evidence files should be content-hashed. Replacing a file at the same logical evidence location is not permitted.

### Rule 6 — Interruption is preserved, not overwritten

An attempt interrupted mid-execution is marked `INTERRUPTED`, never silently marked `PASS`, `FAIL`, or left ambiguous as `EXECUTING`.

## 8. Result State Model

```
              ┌───────────────┐
              │ NOT_EXECUTED  │
              └───────┬───────┘
                      │ execute
                      ▼
               ┌────────────┐
               │ EXECUTING  │
               └──────┬─────┘
           ┌───────────┼────────────┐
           │            │            │
           ▼            ▼            ▼
        PASS          FAIL       BLOCKED
                                     │
                          (mapping / communication /
                           configuration / automation /
                           environment conditions)

      EXECUTING ──(run interrupted)──▶ INTERRUPTED
```

`BLOCKED` is a distinct outcome from `FAIL`, used when valid verification could not be performed because of mapping, communication, configuration, automation or environment conditions — it is **not** a sub-case of `FAIL`.

`NOT_EXECUTED` is used for selected tests that were skipped/stopped before execution began.

`INTERRUPTED` is used when a step or attempt was actively executing when its Test Run was interrupted or stopped; it requires an explicit resume/re-run decision (see 4.21).

## 9. Test Run State Model

Proposed states:

```
DRAFT
  ↓
READY_FOR_VALIDATION
  ↓
VALIDATION_FAILED ←──────────────┐
  ↓                              │
READY                            │ fix/reconfigure
  ↓                              │
RUNNING ──→ PAUSED ──→ RUNNING   │
  │                               │
  ├──→ COMPLETED                  │
  ├──→ STOPPED                    │
  └──→ INTERRUPTED ───────────────┘
```

An interrupted run is not silently converted into a completed result. The system must preserve the interruption and allow an explicit resume/new run/re-run decision. Per Sec 4.21/8, any attempt that was `EXECUTING` when the Test Run entered `STOPPED`/`INTERRUPTED` is itself marked `INTERRUPTED`.

## 10. Dataset Model

V1 supports parameterized execution.

Example:

```
ATC-020: Verify login

Dataset-01
 username = user1
 password = ****

Dataset-02
 username = user2
 password = ****

Dataset-03
 username = user3
 password = ****
```

Each selected dataset gets its own execution result and can be re-run independently. An ATC with no datasets defined (e.g., a simple non-parameterized check) executes through a single default `Test Run Dataset` placeholder row (Sec 4.20) rather than requiring a separate execution path.

## 11. Mapping and Future TCS Corrections

The architecture intentionally separates:

```
TCS/ATC definition
        │
        ▼
Logical target reference
        │
        ▼
Mapping revision (optionally tagged with the app version it was validated against)
        │
        ▼
Adapter-specific target locator
```

Therefore, a future correction to an ATC does not require manually editing every historical execution record.

A mapping correction creates a new mapping revision. Future runs use the new revision; old runs retain the mapping revision that was actually used.

## 12. Version Tracking Example

For the representative case:

```
Project: SM1 Verification

Application:
  SM1.exe
  Version: 2.4.1

Main Module: Controller (HARDWARE)

Submodules:
  Card-1 → 1.10 (HARDWARE)
  Card-2 → 3.04 (HARDWARE)
  Card-3 → 2.15 (HARDWARE)
  Card-4 → 5.01 (HARDWARE)
  Bootloader → 1.02 (FIRMWARE)

Test Configuration:
  SM1-Config-001

Test Run:
  Round-05

Snapshot:
  SM1.exe = 2.4.1
  Card-1 = 1.10
  Card-2 = 3.04
  Card-3 = 2.15
  Card-4 = 5.01
  Bootloader = 1.02

ATCs:
  ATC-001
  ATC-004
  ATC-009

Results:
  ATC-001 → PASS
  ATC-004 → FAIL
  ATC-009 → BLOCKED
```

If Card-2 is later upgraded to `3.05`, Round-05 remains associated with `3.04`.

## 13. Backup/Restore Boundary

V1 backup is project-level.

A project backup package should contain:

```
manifest
project metadata
templates + revisions
TCS/ATC revisions
datasets
mappings + revisions
applications + versions
modules + versions
test configurations
test run snapshots
test runs + attempts
results + failures
evidence
audit events
```

Restore must validate the package before modifying the local database. The recommended restore process is:

```
Select backup
    ↓
Validate manifest/hash
    ↓
Preview project
    ↓
Check conflicts
    ↓
Confirm restore
    ↓
Transactional import
    ↓
Audit restore
```

## 14. Indexing Strategy — Initial

Indexes should be considered for:

- `project_id` on all project-owned tables.
- TCS/ATC revision lineage.
- `test_run_id` on execution-related tables.
- `attempt_id` on step results/evidence/failures.
- Version lookup by application/module.
- Mapping lookup by logical object key + adapter type.
- Audit lookup by entity/user/time.

Exact indexes will be finalized after expected volume analysis.

## 15. Delete / Archive Strategy

V1 should prefer:

- Archive Project rather than delete.
- Retire TCS/ATC/mapping revisions rather than delete them.
- Retire versions rather than delete versions referenced by history.
- Preserve completed Test Runs and attempts.

Physical deletion should be restricted to controlled maintenance operations and must not break historical references.

## 16. Concurrency

V1 is single-user/single-PC. The database is therefore not designed for simultaneous active Test Runs by multiple users.

The schema nevertheless avoids assumptions that would prevent future multi-user deployment:

- Stable IDs.
- User identity fields.
- Explicit ownership relationships.
- Audit events.
- No reliance on local UI state for business history.

Future centralized deployment will define locking/claiming rules.

## 17. Open Design Decisions

These are intentionally deferred to detailed design:

1. Database engine.
2. Exact SQL types and constraints.
3. ID strategy: UUID/ULID/integer/etc.
4. Exact JSON structure for template definitions and parameter datasets.
5. Whether snapshots store full text, references plus hashes, or both.
6. Evidence physical storage layout.
7. Encryption requirements for backup packages.
8. Exact restore conflict strategy.
9. Retention/cleanup implementation.
10. Database migration/versioning mechanism.
11. Whether credential-like Dataset parameter values (Sec 4.6) should be masked or encrypted at rest and in exported reports.

## 18. Acceptance Questions for Review

Before this schema is baselined, reviewers should verify:

1. Can we identify the exact TCS revision used by a Test Run?
2. Can we identify the exact ATC revision and dataset (or default placeholder) used?
3. Can we identify the exact application version?
4. Can we identify every module (hardware/firmware/software) and submodule version used?
5. Can we identify the mapping revision used, and which application version it was validated against?
6. Can we distinguish application FAIL from infrastructure BLOCKED, and from an INTERRUPTED attempt?
7. Can an ATC be re-run without overwriting the previous attempt?
8. Can one dataset be re-run without changing another dataset's result?
9. Can a corrected TCS/ATC/mapping be introduced without modifying historical runs?
10. Can a project be backed up and restored independently?
11. Can a reviewer understand a run from its exported report and evidence?
12. Can the model evolve to centralized multi-user operation later?
13. Can an ATC with zero datasets still be executed and traced with the same rigor as a parameterized one?

---

**Status:** Domain Model & Database Schema Draft v0.2 — ready for review.
**Next artifact after review:** Detailed SQL schema + Test Run state machine + Adapter interface contract.

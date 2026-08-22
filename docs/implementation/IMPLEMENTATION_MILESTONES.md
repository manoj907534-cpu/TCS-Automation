# TCS Automation — Implementation Milestones & Checkpoints

**Version:** 0.1
**Status:** Implementation Baseline
**Technology Baseline:** Technology Stack v0.1
**Design Baseline:** SRS v1.2 / Architecture v0.2 / Domain Model v0.3 / Test Run State Machine v0.4 / Adapter Interface Contract v0.1 / SQL Schema v0.1

## 1. Purpose

This document defines the implementation sequence, milestone objectives, checkpoints and exit criteria for TCS Automation V1.

The purpose is to provide controlled implementation progress without reopening completed design documents for stylistic refinement.

A milestone is complete only when its checkpoint and exit criteria pass.

## 2. Implementation Principles

1. Implement against the approved design baseline.
2. Complete and verify foundational layers before building dependent layers.
3. Use automated tests wherever practical.
4. Do not bypass the Adapter Interface Contract from Core/Application code.
5. Do not introduce technology changes without a documented impact review.
6. A failed checkpoint blocks progression to the dependent milestone until resolved.
7. Small implementation corrections do not automatically require design-document version changes.

## 3. Milestone Overview

| ID | Milestone | Primary Focus | Indicative Effort |
|---|---|---|---:|
| M0 | Project Setup | Solution, build, CI, baseline dependencies | 1–2 days |
| M1 | Database Foundation | SQLite, DDL, migrations, repositories | 3–4 days |
| M2 | Domain & State Machine | Domain objects and state transitions | 3–4 days |
| M3 | Adapter Framework | Adapter abstractions and Mock Adapter | 3–4 days |
| M4 | Test Execution Engine | Execution orchestration and transactions | 4–6 days |
| M5 | Windows Adapter | Windows UI automation | 4–6 days |
| M6 | Evidence, Logs & Results | Evidence, failures, reporting data | 3–4 days |
| M7 | TCS Import & Configuration | TCS/ATC/dataset import and configuration | 3–5 days |
| M8 | Application UI | Main workflows and operator UI | 4–6 days |
| M9 | Embedded Linux/Qt Adapter | Target-specific automation integration | 4–6 days |
| M10 | End-to-End Integration | Full workflow and regression | 5–7 days |
| M11 | Hardening & V1 Release | Packaging, recovery, documentation | 3–5 days |

> Effort estimates are indicative for one developer with AI-assisted development. They are planning estimates, not commitments.

## 4. M0 — Project Setup

### Objective

Create a clean, buildable .NET solution matching the approved architecture.

### Implementation

- Create solution and project structure.
- Configure .NET 8.
- Create Core, Application, Adapter Abstractions, Data, Evidence, Infrastructure and WPF projects as appropriate.
- Create test projects.
- Configure dependency injection.
- Configure logging baseline.
- Establish GitHub Actions build/test workflow.
- Add initial coding/analyzer configuration if required.

### Checkpoint

- Solution builds successfully from a clean checkout.
- All test projects execute.
- No architecture-breaking project references exist.
- CI build passes.

### Exit Criteria

**M0 PASS:** Clean checkout → restore → build → test succeeds.

## 5. M1 — Database Foundation

### Objective

Implement the approved relational model using SQLite.

### Implementation

- Convert SQL Schema v0.1 into executable SQLite DDL.
- Implement database initialization/migration mechanism.
- Implement connection and transaction handling.
- Implement repositories/data access required by the Core/Application layers.
- Add schema-level tests for primary keys, foreign keys, uniqueness and critical checks/triggers.

### Checkpoint

- Fresh database can be created from zero.
- All expected tables exist.
- Foreign keys are enforced.
- Critical state/outcome constraints behave as designed.
- Transaction rollback works.
- Database survives application restart without corruption.

### Exit Criteria

**M1 PASS:** Database integration tests pass against a fresh SQLite database.

## 6. M2 — Domain & State Machine

### Objective

Implement the domain model and Test Run State Machine v0.4 without UI or real automation dependencies.

### Implementation

- Implement domain entities/value objects required for V1.
- Implement Test Run lifecycle.
- Implement Execution Attempt lifecycle.
- Implement Result State transitions.
- Implement Outcome calculation/freeze rules.
- Implement validation-cycle history.
- Implement run lineage.
- Implement mapping resolution behavior.

### Checkpoint

Verify at minimum:

- Valid Test Run transitions succeed.
- Invalid transitions are rejected.
- Terminal states cannot be incorrectly reopened.
- Only the permitted number of active Attempts can execute.
- PASS requires executed evidence/result as specified.
- Pre-execution BLOCKED follows the mapping-exception rule.
- STOPPED and INTERRUPTED remain distinct.
- Outcome is computed once at terminal transition and is not silently recalculated.

### Exit Criteria

**M2 PASS:** State-machine unit/integration tests cover all mandatory transitions and invariants.

## 7. M3 — Adapter Framework

### Objective

Implement the adapter boundary defined by Adapter Interface Contract v0.1.

### Implementation

- Define adapter interfaces.
- Implement capability discovery.
- Implement lifecycle methods.
- Implement standardized adapter facts/errors.
- Implement Mock Adapter for deterministic testing.
- Implement cancellation/health-check behavior at the contract level.

### Checkpoint

- Core can execute a test using only the adapter abstraction.
- Core has no direct dependency on an automation framework.
- Mock Adapter can simulate success, mismatch, unavailable object, communication loss, timeout and adapter error scenarios.
- Capability checks are enforced.

### Exit Criteria

**M3 PASS:** Complete test execution can be demonstrated using Mock Adapter without any real target.

## 8. M4 — Test Execution Engine

### Objective

Connect domain/state-machine behavior, data access and adapters into the core execution workflow.

### Implementation

- Test Run preparation.
- Configuration snapshot creation.
- Dataset snapshot creation.
- ATC selection.
- Mapping resolution.
- Attempt creation.
- Step execution orchestration.
- Result persistence.
- Failure/exception persistence.
- Pause/stop/cancel handling as applicable.
- Recovery/re-execution flow.

### Checkpoint

A complete mock execution must perform:

```text
Create Project
  → Load TCS/ATC configuration
  → Prepare Test Run
  → Snapshot configuration/dataset
  → Validate
  → Execute Attempt
  → Execute Steps through Adapter
  → Persist Results
  → Persist Evidence/Failure data
  → Compute Outcome
  → Reach Terminal Test Run State
```

### Exit Criteria

**M4 PASS:** Full end-to-end execution works using Mock Adapter with persistent SQLite state.

## 9. M5 — Windows Adapter

### Objective

Implement the first real target adapter using Windows UI automation.

### Implementation

- Application launch/attach.
- Object identification.
- Required actions.
- Required verification capabilities.
- Wait/timeout behavior.
- Health check.
- Reconnect where supported.
- Evidence capture.
- Adapter fact/error mapping.

### Checkpoint

- Real Windows application can be attached/launched.
- Representative actions execute.
- Representative verifications execute.
- Object-not-found and timeout scenarios are reported correctly.
- Core determines the final outcome; adapter does not own PASS/FAIL decisions.

### Exit Criteria

**M5 PASS:** Representative real Windows TCS/ATC scenarios execute successfully through the Windows Adapter.

## 10. M6 — Evidence, Logs & Results

### Objective

Make execution results auditable and diagnosable.

### Implementation

- Evidence Store implementation.
- Screenshot/image capture where applicable.
- Evidence metadata.
- Structured application logs.
- Failure/exception records.
- Result/history views at data level.
- Sensitive-value masking rules.

### Checkpoint

For a failed and successful execution, an engineer can identify:

- Test Run
- ATC
- Attempt
- Step
- Result
- Failure/reason
- Evidence
- Application/target/configuration context

Sensitive dataset values must not be exposed in ordinary logs or reports.

### Exit Criteria

**M6 PASS:** Successful and unsuccessful executions have complete traceable result/evidence records.

## 11. M7 — TCS Import & Configuration

### Objective

Implement the workflow for importing and preparing TCS/ATC/dataset data.

### Implementation

- Excel/TCS parsing using ClosedXML.
- Template/revision handling.
- TCS revision creation.
- ATC revision creation.
- Dataset creation and snapshot preparation.
- Validation errors and import diagnostics.
- Test configuration management.

### Checkpoint

- Valid workbook imports successfully.
- Invalid workbook produces useful validation errors.
- Historical TCS/ATC revisions are not silently overwritten.
- Imported data can be selected for execution.

### Exit Criteria

**M7 PASS:** A representative real TCS workbook can be imported and prepared for execution.

## 12. M8 — Application UI

### Objective

Implement the operator-facing WPF application around already-tested Core/Application workflows.

### Implementation

- Project management.
- TCS/configuration selection.
- Test Run preparation.
- Validation display.
- Execution control.
- Current Attempt/step status.
- Results and history.
- Evidence viewing.
- Error/diagnostic display.

### Checkpoint

A user can perform the primary V1 workflow without direct database interaction.

### Exit Criteria

**M8 PASS:** Primary operator workflow is executable through the UI using Mock Adapter and then Windows Adapter where available.

## 13. M9 — Embedded Linux/Qt Adapter

### Objective

Implement the concrete adapter for the Embedded Linux/Qt target after confirming the target application's available automation/communication mechanism.

### Implementation

- Target connection.
- Application preparation/attach as applicable.
- Object identification.
- Required capabilities.
- Verification.
- Evidence capture.
- Communication health/reconnect behavior.
- Target-specific error/fact translation.

### Checkpoint

- Real target can be connected.
- Representative ATCs execute.
- Communication loss is detected and handled according to the state-machine rules.
- Adapter-specific details remain isolated from Core.

### Exit Criteria

**M9 PASS:** Representative Embedded Linux/Qt tests execute end-to-end through the approved adapter.

## 14. M10 — End-to-End Integration

### Objective

Validate the complete V1 workflow across import, configuration, execution, evidence and reporting.

### Checkpoint

At minimum test:

- Successful execution.
- Functional failure.
- Mapping failure/blocking.
- Object unavailable.
- Timeout.
- Communication loss.
- Stop.
- Interrupted/recovery scenario.
- Re-execution.
- Application restart/recovery.
- Historical result immutability.
- Evidence traceability.

### Exit Criteria

**M10 PASS:** Critical end-to-end regression suite passes on the supported target environments.

## 15. M11 — Hardening & V1 Release

### Objective

Prepare a stable distributable V1.

### Implementation

- Packaging/installer.
- Configuration defaults.
- Database initialization/recovery.
- Backup/restore verification.
- Logging configuration.
- Performance sanity checks.
- Error handling cleanup.
- User/developer documentation.
- Release versioning.

### Checkpoint

A clean machine/environment can install, configure and execute the supported V1 workflow using the documented procedure.

### Exit Criteria

**M11 PASS:** Release candidate passes installation, smoke, regression and recovery checks.

## 16. Release Gates

### Gate 1 — Architecture Compliance

After M3:

- Adapter boundary respected.
- Core remains technology-independent.
- Domain/state-machine behavior matches approved documents.

### Gate 2 — Functional MVP

After M6/M7:

- A complete test can be imported/prepared/executed.
- Results and evidence persist.
- Mock and at least one real adapter path work.

### Gate 3 — V1 Release

After M11:

- Installation succeeds.
- Critical regression suite passes.
- Recovery scenarios pass.
- Documentation matches implementation.

## 17. Checkpoint Status Convention

Each milestone should be recorded as one of:

- **NOT STARTED** — implementation has not begun.
- **IN PROGRESS** — implementation underway.
- **BLOCKED** — progress requires an unresolved dependency/decision.
- **PASS** — all mandatory checkpoint and exit criteria passed.
- **FAILED** — implementation exists but one or more required criteria failed.

A milestone should not be marked PASS based only on code completion.

## 18. Change Control During Implementation

If implementation reveals a genuine contradiction with the approved requirements, architecture, domain model, state machine or adapter contract:

1. Stop the affected implementation path.
2. Record the contradiction.
3. Identify the affected baseline document(s).
4. Make the smallest necessary design correction.
5. Version only the affected document(s).
6. Update this milestone document if the implementation sequence changes.

Do not reopen design documents for stylistic improvements or speculative future requirements.

## 19. Changelog

### v0.1

- Initial implementation milestone plan.
- Added milestone checkpoints and exit criteria.
- Added architecture, MVP and V1 release gates.

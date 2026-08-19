# TCS Automation — System Architecture

**Version:** 0.2 — Architecture Draft  
**Status:** Working Draft  
**Baseline:** SRS v1.2

## 1. Purpose

This document defines the high-level architecture for TCS Automation. The architecture is focused on reducing manual V&V/Test Engineer effort while preserving traceability, repeatability, historical integrity, and engineer control over ambiguous or unsafe automation decisions.

## 2. Architectural Goals

1. Keep the TCS/ATC model independent of Windows, Qt, or other automation technology.
2. Make adapters replaceable and capability-driven.
3. Separate test definition, mapping, configuration, execution, results, evidence, audit and reporting.
4. Never overwrite historical execution attempts.
5. Make repeated regression execution fast: select project → select configuration → select ATCs → validate → run.
6. Keep V1 simple: single Windows PC, local database/storage, one active Test Run.
7. Avoid requiring installation of TCS Automation or an automation agent on embedded Linux targets.
8. Allow future centralized/multi-user deployment without redesigning the core domain model.

## 3. High-Level Architecture

```text
                         ┌─────────────────────────────┐
                         │       Presentation UI        │
                         │ Projects / TCS / ATC / Runs │
                         └──────────────┬──────────────┘
                                        │
                         ┌──────────────▼──────────────┐
                         │       Application Core      │
                         │                              │
                         │ Project Manager              │
                         │ TCS Import & Validation      │
                         │ ATC Manager                  │
                         │ Mapping Manager              │
                         │ Test Configuration Manager   │
                         │ Test Run Manager             │
                         │ Validation Engine            │
                         │ Execution Orchestrator       │
                         │ Result / History Manager     │
                         │ Audit Manager                │
                         │ Report Manager               │
                         └──────────────┬──────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
             ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
             │ Adapter API │    │ Data Layer  │    │ Evidence    │
             │             │    │             │    │ Store       │
             └──────┬──────┘    └─────────────┘    └─────────────┘
                    │
             ┌──────┴───────────┐
             │                  │
      ┌──────▼──────┐    ┌──────▼─────────┐
      │ Windows     │    │ Embedded Qt    │
      │ Adapter     │    │ Adapter        │
      └──────┬──────┘    └──────┬─────────┘
             │                  │
       Windows AUT       Ethernet → Linux/Qt Target
```

## 4. Core Modules

### 4.1 Presentation/UI

Provides screens for project management, TCS import, ATC review, mappings, versions, Test Configuration, Test Run creation, execution monitoring, failures/exceptions, history, reports and backup/restore.

The UI shall not contain adapter-specific automation logic.

### 4.2 Project Manager

Owns project identity and project-level configuration, including the configured TCS template definition and template revisions.

Responsibilities:
- Create/open/archive projects.
- Associate TCS, ATCs, mappings, configurations and Test Runs.
- Maintain project-level template/version metadata.

### 4.3 TCS Import & Validation

Reads `.xlsx`, validates workbook structure and mandatory fields, reports actionable errors, and produces a normalized internal TCS representation.

The importer shall not directly execute tests.

### 4.4 ATC Manager

Transforms/reviews normalized TCS into technology-independent ATCs.

An ATC contains logical steps and may contain one or more datasets/parameter sets.

### 4.5 Mapping Manager

Maintains the association between logical ATC targets and actual application/target objects.

Mapping is independent from ATC definition and has its own revision history.

Mapping states:
- Exact/Strong → eligible for automatic execution according to adapter rules.
- Candidate/Ambiguous → engineer confirmation required.
- Unresolved → mapping/correction required.

### 4.6 Test Configuration Manager

Maintains editable Test Configurations. When a Test Run is created, the selected configuration is copied into an immutable **Test Run Configuration Snapshot** owned by that Test Run.

The snapshot contains:
- Application under test and version.
- Hardware modules/submodules and versions.
- Software/firmware versions.
- Target identity.
- Adapter and adapter revision.
- Relevant environment/configuration values.

Changes to the editable Test Configuration shall never modify an existing Test Run Snapshot.

### 4.7 Test Run Manager

Creates and controls named Test Runs/Rounds.

Responsibilities:
- Select one adapter.
- Select ATCs/datasets.
- Create and freeze the Test Run Configuration Snapshot.
- Validate readiness.
- Start/pause/resume/stop.
- Handle failure continuation policy.
- Start re-execution attempts.
- Preserve historical attempts.

### 4.8 Validation Engine

Performs pre-execution checks before allowing the run to start:
- Workbook/ATC readiness.
- Mapping readiness.
- Adapter capability compatibility.
- Application availability.
- Target availability.
- Ethernet connectivity for embedded runs.
- Configuration/version consistency.

Validation results shall be classified as blocking or non-blocking.

### 4.9 Execution Orchestrator

The central execution engine. It reads technology-independent ATC actions and invokes the selected adapter.

It shall not know how a Windows button is clicked or how a Qt object is accessed. Those responsibilities belong to adapters.

### 4.10 Adapter API

The adapter contract provides logical operations such as:

- `ApplicationLaunch`, `ApplicationAttach`, `Connect`.
- `Click`, `DoubleClick`, `RightClick`.
- `Type`, `Clear`, `KeyPress`.
- `Select`, `Check`, `Uncheck`.
- `Scroll`, `DragDrop`.
- `WaitForObject`, `WaitForState`, `WaitForText`.
- `VerifyExists`, `VerifyVisible`, `VerifyText`, `VerifyValue`, `VerifyState`.
- `CaptureEvidence`.
- `ConnectionHealthCheck`, `TargetReconnect`.

These names form the initial canonical capability vocabulary. Parameters and exact implementation contracts will be finalized during detailed design.

### 4.11 Windows Adapter

V1 primary implementation target. It automates supported Windows applications and may launch or attach to the AUT depending on Test Run mode and adapter capability.

### 4.12 Embedded Qt Adapter

Runs from the Windows Test PC and communicates with Linux/Qt targets over Ethernet using an approved target/application mechanism.

V1 modes:
- Already Running.
- Manual.

Automatic remote application launch is not required unless an approved mechanism is explicitly available.

The adapter shall not assume Ethernet alone provides Qt object inspection.

### 4.13 Result/History Manager

Stores execution attempts, step results, ATC results, dataset results, failures, timestamps and Test Run summaries.

A retry/re-run always creates a new attempt.

### 4.14 Evidence Manager

Stores screenshots and other execution evidence and associates each item with:

`Project → Test Run → Attempt → ATC → Dataset → Step`

Video recording is not part of V1.

### 4.15 Audit Manager

Records significant controlled actions with:
- Windows user identity.
- Timestamp.
- Action.
- Entity.
- Before/after values where applicable.

### 4.16 Report Manager

Generates PDF and `.xlsx` reports. Reports shall show Test Run configuration and results down to ATC/dataset where applicable.

### 4.17 Data Layer

V1 uses a local database on the Windows Test PC. Repository interfaces shall isolate persistence from domain logic so a future server database can be introduced without changing the domain model.

Persistence of Test Run, Attempt and Result records shall be transactional/atomic: a crash during a write shall not leave a partially recorded execution attempt that appears valid.

## 5. Key Domain Model

```text
Project
 ├── TCS Template
 │    └── Template Revisions
 ├── TCS Revisions
 │    └── ATC Revisions
 ├── Object Mappings
 │    └── Mapping Revisions
 ├── Test Configurations (editable)
 └── Test Runs
      ├── Adapter
      ├── Test Run Configuration Snapshot (immutable copy)
      ├── ATC Selection
      │    └── Dataset Selection
      └── Execution Attempts
           ├── Step Results
           ├── Dataset Results
           ├── Evidence
           └── Failure/Exception

Audit Log
Project Backup Records
```

A Test Configuration is editable and reusable. A Test Run owns its own frozen snapshot; the snapshot is not a mutable child shared with the current configuration.

## 6. Critical Relationships

### Application and component versions

A Test Run shall preserve the exact combination used, for example:

```text
SM1.exe = v2.4.1

Hardware / Software configuration:
Card-1 = v1.10
Card-2 = v3.04
Card-3 = v2.15
Card-4 = v5.01

ATCs executed against this combination:
ATC-001, ATC-004, ATC-009...
```

Later changes shall not modify this historical snapshot.

### Re-execution

```text
Run-005
 ├── Attempt-01 → FAIL
 ├── Attempt-02 → FAIL
 └── Attempt-03 → PASS
```

The summary may show the latest/selected outcome, but all attempts remain available.

## 7. Execution Flow

```text
1. Engineer opens Project
2. Selects/creates Test Configuration
3. Uploads/selects TCS .xlsx
4. Import + structural validation
5. TCS → ATC generation/review
6. Mapping validation/recovery
7. Engineer selects Adapter
8. Engineer selects ATCs + datasets
9. System performs pre-run validation
10. Embedded: Ethernet/target connectivity check
11. Engineer resolves blocking exceptions
12. System freezes Test Configuration Snapshot
13. Execution Orchestrator starts
14. Adapter executes each logical action
15. Result/Evidence Manager records outcomes transactionally
16. Failure policy determines Continue / Stop / Engineer Action
17. Test Run completes or is interrupted
18. Engineer can re-run failed/selected ATCs
19. Reports and history are available
```

## 8. Failure and Exception Strategy

The system shall distinguish at least:

- Test failure: application behavior did not meet expected result.
- Automation failure: adapter could not perform the required action.
- Mapping exception: target object could not be uniquely identified.
- Communication failure: target connection was lost/unavailable.
- Configuration failure: required version/configuration is inconsistent or unavailable.
- Environment failure: required external condition is unavailable.

### Failure-to-result mapping

| Condition | Result State | Rationale |
|---|---|---|
| Expected behavior achieved | PASS | Application behavior satisfies expected result. |
| Application behavior violates expected result | FAIL | Valid test execution produced a negative product result. |
| Mapping ambiguous/unresolved | BLOCKED | Test could not be safely executed/identified. |
| Communication unavailable/lost | BLOCKED | Cannot attribute failure to application behavior. |
| Adapter/automation cannot perform required action | BLOCKED | Execution infrastructure failed before valid verification. |
| Required configuration/version invalid or unavailable | BLOCKED | Test environment is invalid. |
| Engineer stops before execution | NOT EXECUTED | No valid execution occurred. |
| Test intentionally skipped | NOT EXECUTED | Execution was not attempted. |

This distinction prevents communication, mapping, automation and environment problems from being incorrectly reported as application FAIL results.

## 9. Adapter Capability Model

The following is the initial canonical vocabulary shared by the Adapter API and capability model:

```text
ApplicationLaunch
ApplicationAttach
Connect
Click
DoubleClick
RightClick
Type
Clear
KeyPress
Select
Check
Uncheck
Scroll
DragDrop
WaitForObject
WaitForState
WaitForText
VerifyExists
VerifyVisible
VerifyText
VerifyValue
VerifyState
CaptureEvidence
ConnectionHealthCheck
TargetReconnect
```

The actual capability list, parameters and support matrix will be finalized during detailed design.

## 10. Storage Strategy

V1:
- Local relational database for structured data.
- Local evidence directory for screenshots/evidence.
- Project-level backup package/destination according to configured backup policy.

The exact database engine and file layout are architecture implementation decisions.

## 11. Reviewer Access Model

Because V1 is single-user/local-storage:

- The Test Engineer has live access to the local project database/history.
- A Project/Engineering Reviewer receives exported PDF/Excel reports and associated evidence through the project's normal document-sharing mechanism.
- V1 does not provide shared live database access to another PC.
- Future centralized deployment may provide shared history and reviewer login/access.

## 12. Security Boundary

V1 uses Windows OS identity and local filesystem permissions. Fine-grained application authorization and centralized identity management are future enhancements.

The architecture shall avoid storing unnecessary credentials for embedded targets. Target authentication/connection details, if required by a specific adapter, shall be handled through the approved project/environment mechanism.

Backup destinations shall be treated as a separate security boundary: if backups are copied to removable media or a network location, the protection of the Windows Test PC's local ACLs shall not be assumed to protect those copies. The V1 backup design shall therefore document destination permissions/protection before baseline.

## 13. Backup and Recovery

V1 backup/restore shall be **project-level**. A backup package shall be independently restorable without requiring restoration of unrelated projects on the same Test PC.

Backup scope shall include:
- Project metadata.
- TCS template and template revisions.
- TCS/ATC definitions and revisions.
- Mappings and mapping revisions.
- Editable Test Configurations.
- Test Run Configuration Snapshots.
- Test Runs and attempts.
- Results.
- Audit records.
- Evidence.

Restore shall preserve entity relationships, stable identifiers and historical execution identity.

Backup and restore operations shall themselves be audited.

## 14. Technology Decisions — To Be Finalized

The SRS intentionally does not mandate the implementation language/framework/database. Detailed design shall evaluate:
- Windows UI technology.
- Excel `.xlsx` library.
- Windows UI automation technology.
- Qt/Qt-QML target communication/inspection mechanism.
- Local database engine.
- PDF/XLSX reporting library.
- Logging framework.
- Installer/update mechanism.

The selection shall favor open-source/free components where practical and shall avoid unnecessary paid dependencies.

## 15. Architecture Principles for Future Development

1. No adapter-specific code in the domain model.
2. No UI-specific logic in execution orchestration.
3. No direct database access from UI screens.
4. Every persistent entity receives a stable identifier.
5. Historical records are append-oriented/immutable wherever required.
6. Adapter capabilities are discoverable and use the canonical vocabulary defined in Section 9.
7. Failures are classified before reporting and mapped to a defined result state.
8. Manual intervention is explicit and auditable.
9. New adapters should implement the existing adapter contract rather than alter core execution logic.
10. Test Run/Attempt/Result persistence shall be atomic/transactional.
11. A Test Run owns its immutable configuration snapshot; editable Test Configurations shall never mutate historical snapshots.
12. V1 simplicity must not prevent future centralized deployment.

## 16. SRS Traceability

The following table provides high-level architecture coverage. Detailed verification traceability will be maintained during design/test planning.

| Architecture Area | Primary SRS Coverage |
|---|---|
| Presentation/UI | FR-001–003, FR-021, FR-050–054, FR-080–084, FR-100–106, FR-150–153 |
| Project Manager | FR-001–003, FR-013, FR-160–162 |
| TCS Import & Validation | FR-010–015 |
| ATC Manager | FR-020–026 |
| Mapping Manager | FR-040–048 |
| Test Configuration Manager | FR-070–074 |
| Test Run Manager | FR-050–054, FR-080–084, FR-100–106 |
| Validation Engine | FR-090–095 |
| Execution Orchestrator | FR-030–032, FR-100–106, FR-110–112 |
| Adapter API | FR-030–032, FR-050–054 |
| Windows Adapter | FR-060–063 |
| Embedded Qt Adapter | FR-064–065, FR-130–133 |
| Result/History Manager | FR-100–112 |
| Evidence Manager | FR-120–125 |
| Audit Manager | FR-140–142 |
| Report Manager | FR-150–153 |
| Data Layer | FR-160–162, NFR-002, NFR-007, NFR-011–014 |
| Security Boundary | NFR-009–010 |
| Backup/Recovery | FR-160–162, NFR-012 |
| Cross-cutting architecture | NFR-001–008, NFR-013–015 |

## 17. Next Design Artifacts

The next architecture/design work shall define, in order:

1. Detailed module interfaces.
2. Domain/entity model and database schema.
3. ATC JSON/internal representation.
4. Adapter interface contract.
5. Windows adapter capability design.
6. Embedded Qt adapter communication boundary.
7. Test Run state machine.
8. Mapping/recovery algorithm.
9. Failure/exception state model.
10. UI navigation and screen responsibilities.
11. Logging/audit schema.
12. Evidence directory/data model.
13. Backup/restore design.
14. Technology stack selection.

---

**Status:** Architecture Draft v0.2 — ready for focused review before detailed domain/database design.

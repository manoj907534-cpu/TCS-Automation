# TCS Automation — Software Requirements Specification (SRS)

**Document ID:** TCS-AUT-SRS  
**Revision:** 1.2  
**Status:** Requirements Baseline Candidate  
**Project:** TCS Automation  
**Date:** 2026-08-19

---

## 1. Purpose

TCS Automation shall reduce repetitive manual V&V test execution effort by reusing existing Excel-based Test Case Specifications (TCS), converting supported test instructions into structured Automated Test Cases (ATCs), executing them through technology-specific adapters, and preserving results, evidence, configuration and history.

The system shall keep the test definition independent of application technology wherever technically feasible.

## 2. Objectives

1. Reduce repetitive manual test execution.
2. Reuse existing `.xlsx` TCS information.
3. Generate/review structured ATCs without forcing engineers to write proprietary scripts.
4. Reuse and recover object mappings to reduce repeated manual corrections.
5. Support Test Runs/Rounds, re-execution and historical results.
6. Record application, hardware module/submodule and software/firmware versions used for every run.
7. Support parameterized/data-driven execution.
8. Provide evidence, audit history and PDF/Excel reporting.
9. Support Windows automation first and provide an embedded Linux/Qt adapter architecture.
10. Avoid requiring TCS Automation installation or an automation agent on embedded targets in V1.

## 3. Problem Statement

Test Engineers spend significant time manually executing repetitive TCS. Change-impact testing can also cause previous-round issues to be missed when only changed functionality is exercised. Existing TCS, execution information and defect records may be maintained in Excel, Mantis and other project records.

TCS Automation shall automate repeatable work while keeping the engineer in control of ambiguous mappings, exceptions, recovery and V&V decisions.

Mantis remains an external defect-management system. **V1 shall not integrate with or replace Mantis**; a defect reference may be recorded against a result if required.

## 4. Scope

### 4.1 In Scope

- Project/test asset management.
- `.xlsx` TCS import and validation.
- Project-configured TCS template definition.
- TCS-to-ATC relationship and ATC revision history.
- Technology-independent action/verification vocabulary.
- Data-driven/parameterized ATCs.
- Object identification, mapping, mapping revision and mapping recovery.
- Adapter selection and capability validation.
- Windows application automation.
- Embedded Linux/Qt/Qt-QML automation through an applicable adapter.
- Test Configuration and application/module/submodule version tracking.
- Test Runs/Rounds.
- Pre-execution validation including Ethernet connectivity for applicable embedded runs.
- Execution, failure handling, retry/re-execution and recovery.
- Screenshots and applicable evidence.
- Audit trail.
- PDF and `.xlsx` reports.
- Local backup/restore.

### 4.2 Out of Scope for V1

- ASRS description creation/maintenance.
- Direct Mantis integration.
- Video recording.
- TCS Automation installation on embedded targets.
- Mandatory target-side automation agent.
- Mandatory modification of the AUT solely for automation.
- Embedded automatic remote application launch unless an approved target mechanism already exists and is explicitly supported.
- Multi-user simultaneous execution.
- Centralized/server deployment.

## 5. Users and V1 Deployment

### 5.1 V&V / Test Engineer

The primary V1 user shall import TCS, generate/review ATCs, resolve mappings/exceptions, create Test Runs, execute/re-execute tests, review evidence and export reports.

### 5.2 Project / Engineering Reviewer

V1 is **single-user, single-PC** with local database/history/evidence storage. Reviewers therefore do not have live/shared access to the engineer's local database.

Reviewer access in V1 shall be through exported PDF/Excel reports and explicitly shared evidence files. Live/shared history is a future centralized-deployment capability.

### 5.3 Deployment Topology

```text
Windows Test PC
 ├─ TCS Automation UI
 ├─ Automation Core
 ├─ Adapters
 ├─ Local Database / History
 └─ Evidence Storage
          |
          +---- Windows AUT
          |
          +---- Ethernet ---- Embedded Linux / Qt Target
```

V1 shall support one active Test Run per Test PC.

## 6. TCS Import and Template Requirements

**FR-001 — Import:** The system shall import Microsoft Excel `.xlsx` workbooks.

**FR-002 — Validation:** The system shall validate the workbook against the project's configured TCS template, including required worksheets, mandatory columns, mandatory values, duplicate identifiers and invalid row structures, and shall report actionable errors.

**FR-003 — Template:** Each project shall reference a TCS import template defining the expected worksheet/column structure and mandatory fields. The exact template editing/version-control workflow shall be finalized during architecture.

**FR-004 — Compatibility:** V1 shall accept `.xlsx`; other spreadsheet formats are out of scope.

**FR-005 — Capacity:** V1 shall target at least 10,000 TCS source rows per workbook.

**FR-006 — Traceability:** Imported Test Case IDs, requirement references, titles, descriptions, scenarios, inputs and expected/observed results shall be preserved where present.

## 7. ATC and Data-Driven Requirements

**FR-010:** The system shall interpret supported test instructions and generate structured ATC steps.

**FR-011:** Engineers shall be able to review and correct generated ATCs before execution.

**FR-012:** The source TCS-to-ATC relationship shall be preserved.

**FR-013:** ATC revisions shall be maintained.

**FR-014:** An ATC shall support one or more parameter/data sets.

**FR-015:** Each ATC/data-set execution shall have an independently identifiable result and shall not overwrite another dataset result.

## 8. Standard Action Model

The core shall use a technology-independent action/verification model. V1 logical operations may include:

- Launch/close/restart where supported.
- Connect/disconnect.
- Click/double-click/right-click/hover.
- Drag/drop and scroll.
- Type/clear.
- Key press/combination.
- Select/check/uncheck.
- Wait/wait-for-object/wait-for-state/wait-for-text.
- Verify exists/visible/enabled/text/value/selected/checked/state.

Actual capability shall be declared by the selected adapter.

## 9. Object Identification and Mapping

**FR-020:** The system shall attempt automatic object identification before requesting manual intervention.

**FR-021:** Adapters shall support multiple identification strategies appropriate to their technology.

**FR-022:** Identification shall be classified as:

1. **Automatic** — a unique exact/strong match satisfies deterministic adapter validation rules.
2. **Engineer Confirmation Required** — multiple viable candidates or a recovered/non-exact mapping requires engineer approval.
3. **Unresolved** — no viable match exists and mapping/correction is required.

**FR-023:** The system shall not silently execute an object interaction when multiple viable targets remain.

**FR-024:** Approved mappings shall be reusable.

**FR-025:** Object mappings shall be maintained separately from ATC definitions.

**FR-026:** Mapping revisions shall be maintained independently from TCS/ATC revisions.

**FR-027:** The system shall attempt mapping recovery when an existing mapping becomes invalid after an application change.

**FR-028:** Recovered non-exact/strong mappings shall require engineer approval before use.

## 10. Adapters and Execution Modes

**FR-030:** The engineer shall select one adapter for a Test Run.

**FR-031:** A Test Run shall use one adapter only.

**FR-032:** Adapters shall expose capabilities and configuration requirements.

**FR-033:** The system shall validate ATC capability compatibility before execution.

### Windows

V1 shall support:

- **Automatic:** adapter may launch the AUT where supported and permitted by Windows.
- **Already Running:** connect to an existing AUT instance where supported.
- **Manual:** engineer prepares the AUT and continues explicitly.

### Embedded Linux / Qt

V1 shall support:

- **Already Running**
- **Manual**

Embedded Automatic remote application launch is **not required in V1**. It may be added later only when an approved remote-launch mechanism is explicitly configured and supported by the target adapter.

## 11. Test Configuration and Version Traceability

Every Test Run shall preserve a Test Configuration snapshot containing, where applicable:

- AUT/application identity and version.
- Hardware module/submodule identity and version.
- Software/firmware identity and version.
- Target/board identity.
- Adapter identity/revision/capabilities.
- Relevant environment/configuration values.

The system shall explicitly record which module/submodule versions were used to test the application version under test.

Historical configuration snapshots shall not change when later configurations change.

## 12. Test Runs / Rounds

**FR-050:** The system shall support named Test Runs/Rounds.

**FR-051:** Engineers shall select ATCs and applicable datasets for a run.

**FR-052:** A run shall record user, adapter, application version, module/submodule versions, software/firmware versions, configuration, selected ATCs/datasets and timestamps.

**FR-053:** Historical runs shall be viewable on the Test PC.

**FR-054:** Completed runs shall be archivable without loss of history.

## 13. Pre-Execution Validation

Before execution the system shall validate:

- Test Configuration completeness.
- ATC readiness.
- Object mappings.
- Adapter availability/capabilities.
- AUT/target availability where applicable.
- Required embedded Ethernet/target connectivity.

Blocking validation failures shall prevent execution until resolved or explicitly handled by defined rules.

## 14. Execution, Failure and Recovery

**FR-060:** Support Start, Pause, Resume and Stop where supported by the adapter/state.

**FR-061:** The engineer shall choose whether execution continues or stops after an ATC failure.

**FR-062:** Support individual, selected-ATC and failed-ATC re-execution.

**FR-063:** Re-execution shall create a new attempt and shall never overwrite previous attempts.

**FR-064:** Interrupted runs shall support resume same run, create new run, or cancel/exit where technically applicable.

**FR-065:** Resuming an interrupted run shall warn if execution-critical configuration has changed.

**FR-066:** Result states shall include PASS, FAIL, BLOCKED and NOT EXECUTED.

Communication loss on an embedded target shall be recorded separately from an application failure where technically observable. Retry/reconnect/pause/stop options shall be provided according to adapter capability.

## 15. Evidence and Audit

**FR-070:** Step-level action, expected result and actual result shall be recorded where available.

**FR-071:** Errors/exceptions shall be recorded where available.

**FR-072:** Applicable failures shall capture a screenshot at or near the failure point. Video is out of scope for V1.

**FR-073:** Evidence shall be associated with the execution attempt, ATC, dataset where applicable and Test Run.

**FR-074:** The system shall maintain an audit trail for significant user/system actions.

**FR-075:** Audit records shall capture user identity, timestamp, action, affected entity and relevant before/after values where applicable.

**FR-076:** Audit identity shall be recorded for Test Run creation/execution, mapping approval/recovery, ATC correction, exception resolution and other configured controlled actions.

## 16. Embedded Linux / Qt / Qt-QML Adapter

The embedded adapter shall be capability-based. Exact mechanisms shall depend on interfaces available on the target and AUT.

V1 logical capabilities shall include, where supported:

| Group | Operations |
|---|---|
| Connection | Connect to running target/AUT, verify connection, disconnect, reconnect |
| Interaction | Click/tap, double-click where meaningful, type/input, clear, key press, select, check/uncheck, scroll |
| Synchronization | Wait, wait-for-object, wait-for-state, wait-for-text/value |
| Verification | Exists, visible, text/value, enabled/state, selected/checked |
| Navigation | Menu/tab/navigation where exposed by the AUT |
| Evidence | Applicable screenshot/image capture |

Constraints:

- Ethernet is the primary planned PC-to-target connection.
- Ethernet/target connectivity shall be checked before applicable embedded runs.
- TCS Automation remains on the Windows Test PC.
- No embedded automation agent is required by the core V1 design.
- Ethernet alone shall not be assumed to provide Qt object inspection.
- The adapter shall support i.MX and other Linux-based boards without processor-specific coupling in the core.

## 17. Reporting

**FR-080:** The system shall provide in-application execution reporting.

**FR-081:** The system shall export PDF reports.

**FR-082:** The system shall export detailed `.xlsx` reports.

**FR-083:** Reports shall include project/run identity, user, application version, module/submodule versions, software/firmware versions, adapter/configuration, selected TCS/ATCs, dataset/parameter-set identity, **per-ATC and per-dataset results**, attempt/re-execution history, expected/actual results, failure/evidence references and summary statistics.

**FR-084:** Because V1 storage is local, Project/Engineering Reviewers shall access results through exported PDF/Excel reports and explicitly shared evidence files. Live/shared history is out of scope for V1.

## 18. Backup and Restore

**FR-090:** The system shall provide a mechanism to back up project data, configuration, Test Run history, audit records and evidence.

**FR-091:** The system shall provide a mechanism to restore from a valid backup.

**FR-092:** Backup and restore operations shall be logged.

## 19. Non-Functional Requirements

| ID | Category | Requirement |
|---|---|---|
| NFR-001 | Usability | Minimize manual interaction and present actionable exceptions. |
| NFR-002 | Reliability | Preserve execution history and never silently overwrite previous attempts. |
| NFR-003 | Performance/Scale | V1 architecture shall target at least 10,000 TCS/ATC records per project and 100 historical Test Runs per project. Detailed response-time targets shall be established during architecture/performance qualification. |
| NFR-004 | Maintainability | Core execution shall be separated from adapter-specific implementation. |
| NFR-005 | Extensibility | New adapters/capabilities shall be addable without redesigning the complete application. |
| NFR-006 | Logging | Important application, adapter, execution, communication, backup/restore and error events shall be timestamped/logged. |
| NFR-007 | Data Integrity | Results, evidence and configuration shall remain associated with their originating execution context. |
| NFR-008 | Recoverability | Interrupted execution and communication loss shall be recoverable according to adapter capability. |
| NFR-009 | User Identity | V1 shall use the Windows logged-in user identity for attribution of controlled actions and Test Runs. A separate application password database is not required for V1. |
| NFR-010 | Access Control | V1 shall rely on Windows OS access controls for application data/evidence. Fine-grained application roles are out of scope for V1. |
| NFR-011 | Storage | Historical evidence shall be retained unless explicitly archived/deleted under defined rules. |
| NFR-012 | Backup | Project data, configuration, history, audit records and evidence shall be included in the backup scope. |
| NFR-013 | Concurrency | V1 shall support one active Test Run per Windows Test PC and shall not support simultaneous execution by multiple users on the same installation. |
| NFR-014 | Deployment | V1 shall operate as a single-user desktop application with local database/history and evidence storage on the Windows Test PC. |
| NFR-015 | Portability | The core shall avoid unnecessary dependency on a specific embedded processor or board. |

## 20. Data and History

The following shall be separately identifiable and history-aware:

- TCS and TCS revisions.
- ATCs and ATC revisions.
- Object mappings and mapping revisions.
- Application versions.
- Hardware module/submodule versions.
- Software/firmware versions.
- Test Configurations.
- Test Runs/Rounds.
- Datasets/parameter sets.
- Execution attempts.
- Evidence.
- Audit records.

Re-execution shall never overwrite previous attempts. Each Test Run shall preserve its configuration snapshot.

## 21. Assumptions and Constraints

- TCS input is primarily Excel `.xlsx`.
- Project TCS templates define required worksheet/column structure.
- ASRS descriptions and ASRS-to-TCS traceability remain external to V1.
- Mantis remains external to V1.
- Windows automation is the first implementation target.
- Qt/Qt-QML applications run on Linux-based embedded targets.
- Ethernet is the primary PC-to-target connection.
- V1 is single-user/single-PC and local-storage based.
- Implementation technologies, automation libraries, database and UI framework are architecture decisions.

## 22. Future Enhancements

- Additional Windows UI technologies/custom controls.
- Advanced Qt/QML inspection and target adapters.
- Embedded automatic remote launch when an approved target mechanism exists.
- Additional embedded communication mechanisms.
- Advanced intelligent mapping/recovery.
- Multi-user/client-server deployment and centralized database.
- Role-based application authorization.
- Advanced Test Run comparison/analytics.
- Additional reports/formats/platforms.

## 23. High-Level Acceptance Criteria

1. A valid TCS `.xlsx` can be imported and validation errors are clearly reported.
2. Supported TCS instructions can be transformed into structured ATCs.
3. ATCs can contain multiple datasets and each dataset has an independent result.
4. Engineers can review/correct ATCs and mappings.
5. Object identification distinguishes Automatic, Engineer Confirmation Required and Unresolved cases.
6. Supported Windows objects can be automated through the Windows adapter.
7. A Test Run records the selected adapter, application version, module/submodule versions and other Test Configuration data.
8. Pre-execution validation identifies blocking configuration/mapping/adapter/AUT/target issues.
9. Embedded runs validate Ethernet/target communication before starting.
10. Results and evidence are preserved.
11. Failed/selected ATCs can be re-run without destroying previous attempts.
12. Interrupted runs can be recovered according to defined behavior.
13. Audit records identify users and significant controlled actions.
14. Backup/restore preserves project/test history and evidence.
15. PDF and Excel reports contain per-dataset results.
16. The technology-independent ATC model remains reusable for future adapters.

## 24. Glossary

| Term | Meaning |
|---|---|
| ASRS | Application Software Requirements Specification, as used by the project. ASRS descriptions/release remain under the existing project/developer process. |
| TCS | Test Case Specification / Test Case data used by the V&V process; official organizational expansion shall be confirmed. |
| ATC | Automated Test Case used by TCS Automation for executable test steps. |
| V&V | Verification and Validation. |
| Test Run / Round | A defined execution instance containing configuration, adapter, selected ATCs/datasets and results. |
| Test Configuration | Application, hardware/software, modules/submodules, versions and environment information used for a Test Run. |
| Adapter | Technology/environment-specific component that translates technology-independent ATC actions into executable operations. |
| Mapping | Association between an ATC target and the actual application/target object used for execution. |
| AUT | Application Under Test. |
| CRUD | Create, Read, Update and Delete operations. |
| Mantis | External defect/report tracking system; no V1 integration. |
| Dataset / Parameter Set | Input values used to execute the same ATC repeatedly with different data. |

## 25. Architecture-Relevant Decisions

| Decision | V1 Baseline |
|---|---|
| Deployment | Single-user Windows desktop application |
| Storage | Local database/history and evidence on Test PC |
| Concurrency | One active Test Run per Test PC |
| Reviewer access | Exported PDF/Excel and shared evidence only |
| Embedded launch | Already Running/Manual only; no required target agent |
| Embedded communication | Ethernet as primary planned connection |
| Identity | Windows logged-in user identity |
| TCS input | `.xlsx` only |
| Template | Project-configured; editing/version workflow to be finalized in architecture |
| Video | Out of scope |
| Mantis | No V1 integration |
| ASRS | External process; no ASRS description management |
| ATC | Technology-independent, adapter-executable, revisioned |
| Mapping | Separate versioned mapping layer with deterministic states |
| Version traceability | Application + module/submodule + hardware/software versions in Test Configuration |
| Backup | Required; mechanism selected during architecture |

## 26. Open Items for Architecture / Detailed Design

1. Confirm exact mandatory columns and worksheet names in the existing TCS `.xlsx` template.
2. Confirm official organizational terminology for TCS and ASRS.
3. Finalize V1 action/verification vocabulary.
4. Define project template editing/version-control workflow.
5. Identify available remote interfaces on representative Linux/Qt target boards.
6. Confirm Windows-integrated identity is acceptable.
7. Confirm backup destination, frequency and retention.
8. Confirm representative project volumes against baseline capacity.
9. Confirm format for storing Mantis defect references without integration.
10. Confirm project-specific security, network and IT restrictions.

## 27. Approval

| Role | Name | Status | Date |
|---|---|---|---|
| Prepared By | | Draft | |
| V&V / Test Engineering Review | | Pending | |
| Software / Technical Review | | Pending | |
| Project Approval | | Pending | |

---

**Revision 1.2 incorporates the detailed review of the previous SRS draft and is the requirements baseline candidate for architecture work.**

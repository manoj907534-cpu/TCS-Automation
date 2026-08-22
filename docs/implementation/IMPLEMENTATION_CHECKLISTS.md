# TCS Automation — Implementation Checklists

**Version:** 0.1  
**Status:** Implementation Baseline  
**Related:** IMPLEMENTATION_MILESTONES.md v0.1 / TECHNOLOGY_STACK.md v0.1

## 1. Purpose

This document provides module-wise checklists to be completed before committing implementation changes.

The checklist is a **human verification gate**. Copilot may implement code and tests, but a milestone must not be considered complete merely because Copilot reports success.

Use this process for every milestone:

```text
Copilot task
    ↓
Implementation + tests
    ↓
Build + automated tests
    ↓
Copilot self-check
    ↓
Developer checklist
    ↓
Git diff review
    ↓
Commit
    ↓
Push
    ↓
Next milestone
```

## 2. How to Use with Copilot

### 2.1 Start each milestone

Give Copilot a short task prompt such as:

```text
Implement Mx from docs/implementation/IMPLEMENTATION_MILESTONES.md.

Follow .github/copilot-instructions.md.
Read only the documents relevant to Mx.
Implement the smallest complete solution.
Write/update tests alongside the implementation.
Build and run the relevant tests.
Do not proceed to the next milestone.
```

Replace `Mx` with M0, M1, etc.

### 2.2 Ask Copilot for a self-check

Before reviewing the Git diff, use:

```text
Self-check the current Mx implementation against:
- docs/implementation/IMPLEMENTATION_MILESTONES.md
- applicable architecture/design documents
- .github/copilot-instructions.md

Do not modify files.

Return:
1. PASS/FAIL
2. Missing requirements
3. Failed tests
4. Architecture concerns
5. Unnecessary changes
6. Unapproved dependencies
```

### 2.3 Developer review

Do not commit until the applicable checklist below is reviewed manually.

Copilot's PASS is not a substitute for this review.

---

# 3. Common Checklist — Every Commit

### Scope

- [ ] Change belongs to the current milestone/task.
- [ ] No unrelated feature was implemented.
- [ ] No speculative future functionality was added.
- [ ] Existing working behavior was not unnecessarily rewritten.

### Architecture

- [ ] Approved architecture is respected.
- [ ] Core/domain remains technology-independent.
- [ ] No unauthorized dependency direction was introduced.
- [ ] Adapter boundaries are respected.
- [ ] Database access remains behind the Data Layer.
- [ ] Evidence access remains behind the Evidence Store abstraction.

### Code Quality

- [ ] Existing code was reused where appropriate.
- [ ] No unnecessary abstraction was introduced.
- [ ] Naming is consistent with the project.
- [ ] No dead/debug code remains.
- [ ] No commented-out implementation remains without a clear reason.
- [ ] No TODOs were added for functionality required by the current task.

### Tests

- [ ] Tests were added/updated with functional changes.
- [ ] Tests cover normal behavior.
- [ ] Tests cover important failure/edge behavior.
- [ ] Relevant tests pass.
- [ ] Existing regression tests pass.
- [ ] No tests were deleted or weakened just to make the build pass.

### Security

- [ ] No passwords/secrets/tokens were added.
- [ ] No sensitive values are written to logs.
- [ ] Configuration does not contain hardcoded credentials.
- [ ] Evidence does not unintentionally expose sensitive information.

### Dependencies

- [ ] No unapproved package/framework was added.
- [ ] Package versions are compatible with TECHNOLOGY_STACK.md.
- [ ] Dependency addition is actually required.

### Build

- [ ] Clean build succeeds.
- [ ] Relevant test projects build successfully.
- [ ] No new compiler warnings of concern were introduced.

### Git Diff

- [ ] Every changed file has been reviewed.
- [ ] No accidental generated files are included.
- [ ] No IDE/user-specific files are included.
- [ ] No unrelated documentation changes are included.
- [ ] Commit contains one logical task/checkpoint.

### Commit

- [ ] Checklist completed.
- [ ] Commit message follows the milestone convention.
- [ ] Commit is made only after review.
- [ ] Push to GitHub occurs only after local verification.

---

# 4. M0 — Project Setup Checklist

### Structure

- [ ] .NET 8 solution exists.
- [ ] Required projects exist according to the architecture.
- [ ] Test projects exist.
- [ ] Project names are clear and consistent.
- [ ] Project references follow the approved dependency direction.

### Tooling

- [ ] Dependency injection baseline configured.
- [ ] Configuration baseline configured.
- [ ] Logging baseline configured.
- [ ] xUnit configured.
- [ ] NSubstitute configured where required.
- [ ] GitHub Actions build/test workflow configured.

### Architecture

- [ ] Core does not reference WPF.
- [ ] Core/Application does not reference FlaUI.
- [ ] Adapter projects remain separate from Core.
- [ ] Data/Evidence responsibilities remain separated.

### Verification

- [ ] Clean checkout restores successfully.
- [ ] Solution builds successfully.
- [ ] All tests execute.
- [ ] CI build passes.
- [ ] No M1 functionality has been implemented prematurely.

**M0 Commit Gate:** All applicable common checks + all M0 checks PASS.

---

# 5. M1 — Database Foundation Checklist

### Schema

- [ ] SQLite-compatible executable schema exists.
- [ ] All required tables are present.
- [ ] Primary keys are correct.
- [ ] Foreign keys are correct.
- [ ] Required UNIQUE constraints exist.
- [ ] Required CHECK constraints exist.
- [ ] Foreign-key enforcement is enabled.
- [ ] Required indexes exist.

### Data Access

- [ ] Dapper is used as approved.
- [ ] Database connections are managed correctly.
- [ ] Transactions are explicit where required.
- [ ] Repository responsibilities are clear.
- [ ] Business rules are not unnecessarily embedded in repositories.

### Tests

- [ ] Fresh database creation tested.
- [ ] Foreign-key behavior tested.
- [ ] Uniqueness tested.
- [ ] Critical constraints tested.
- [ ] Transaction commit tested.
- [ ] Transaction rollback tested.
- [ ] Repository behavior tested.
- [ ] Restart/reopen behavior tested where applicable.

### Data Integrity

- [ ] Historical execution data cannot be silently overwritten.
- [ ] Required relationships cannot be orphaned.
- [ ] Database failure behavior is handled appropriately.

**M1 Commit Gate:** Database integration tests pass against a fresh SQLite database.

---

# 6. M2 — Domain & State Machine Checklist

### Domain

- [ ] Required entities/value objects implemented.
- [ ] Domain invariants are enforced.
- [ ] Domain logic is independent of WPF/database/automation frameworks.

### State Machine

- [ ] Every mandatory valid transition is implemented.
- [ ] Every mandatory invalid transition is rejected.
- [ ] Terminal states are protected.
- [ ] Attempt execution rule is enforced.
- [ ] STOPPED and INTERRUPTED remain distinct.
- [ ] BLOCKED mapping behavior follows the approved rule.
- [ ] Outcome calculation/freeze behavior follows v0.4.
- [ ] Re-execution behavior follows the approved state machine.
- [ ] Validation history behavior is preserved.
- [ ] Run lineage behavior is preserved.

### Tests

- [ ] Transition tests exist.
- [ ] Invalid transition tests exist.
- [ ] Terminal-state tests exist.
- [ ] Outcome tests exist.
- [ ] Edge-case tests exist.
- [ ] Tests do not depend on UI or real automation.

**M2 Commit Gate:** All mandatory state-machine tests pass.

---

# 7. M3 — Adapter Framework Checklist

### Contract

- [ ] Adapter interfaces match Adapter Interface Contract v0.1.
- [ ] Capability discovery is represented correctly.
- [ ] Lifecycle operations are represented correctly.
- [ ] Standardized facts/errors are implemented.
- [ ] Cancellation behavior is represented.
- [ ] Health-check behavior is represented.

### Mock Adapter

- [ ] Success scenario implemented.
- [ ] Mismatch scenario implemented.
- [ ] Object unavailable scenario implemented.
- [ ] Timeout scenario implemented.
- [ ] Communication loss scenario implemented.
- [ ] Adapter error scenario implemented.
- [ ] Environment error scenario implemented where applicable.

### Architecture

- [ ] Core uses adapter abstractions only.
- [ ] Core does not reference FlaUI.
- [ ] Adapter does not determine business PASS/FAIL/BLOCKED.
- [ ] Adapter-specific details remain isolated.

**M3 Commit Gate:** Complete deterministic Mock Adapter execution works.

---

# 8. M4 — Test Execution Engine Checklist

### Workflow

- [ ] Test Run preparation implemented.
- [ ] Configuration snapshot implemented.
- [ ] Dataset snapshot implemented.
- [ ] ATC selection implemented.
- [ ] Mapping resolution implemented.
- [ ] Attempt creation implemented.
- [ ] Step execution implemented.
- [ ] Result persistence implemented.
- [ ] Failure persistence implemented.
- [ ] Evidence association implemented.
- [ ] Terminal outcome behavior implemented.

### State/Transaction Integrity

- [ ] Execution follows the approved state machine.
- [ ] Required operations are transactional.
- [ ] Partial failure does not corrupt state.
- [ ] Adapter errors are translated correctly.
- [ ] Cancellation/stop behavior is correct.

### Tests

- [ ] Successful execution.
- [ ] Functional failure.
- [ ] Mapping failure/blocking.
- [ ] Adapter error.
- [ ] Timeout.
- [ ] Communication loss.
- [ ] Stop.
- [ ] Interruption.
- [ ] Re-execution.
- [ ] Terminal outcome.

**M4 Commit Gate:** Complete Mock Adapter + SQLite execution passes end-to-end.

---

# 9. M5 — Windows Adapter Checklist

### Automation

- [ ] FlaUI is isolated inside Windows Adapter.
- [ ] Application launch/attach works.
- [ ] Object identification works.
- [ ] Required actions work.
- [ ] Required verification works.
- [ ] Wait/timeout behavior works.
- [ ] Health check works.
- [ ] Evidence capture works where required.

### Error Handling

- [ ] Object-not-found behavior verified.
- [ ] Timeout behavior verified.
- [ ] Application failure behavior verified.
- [ ] Communication/environment failure behavior verified where applicable.
- [ ] Errors are translated to the adapter contract.

### Architecture

- [ ] Core/Application contains no FlaUI dependency.
- [ ] Windows-specific code remains inside the adapter.
- [ ] Adapter does not determine business outcome.

### Tests

- [ ] Unit tests for adapter logic.
- [ ] Mock-based tests where real UI is unavailable.
- [ ] Representative real UI smoke test.

**M5 Commit Gate:** Representative real Windows scenario passes through the adapter.

---

# 10. M6 — Evidence, Logs & Results Checklist

### Evidence

- [ ] Evidence Store abstraction is respected.
- [ ] Evidence files are stored in the intended location.
- [ ] Evidence metadata is persisted.
- [ ] Evidence links to Test Run/Attempt/Step correctly.
- [ ] Evidence failures are handled safely.

### Logging

- [ ] Structured logging is implemented.
- [ ] Useful diagnostic context is included.
- [ ] Credentials/secrets are never logged.
- [ ] Sensitive dataset values are masked/omitted.
- [ ] Logging failures do not corrupt execution state.

### Traceability

- [ ] Successful execution is traceable.
- [ ] Failed execution is traceable.
- [ ] Evidence can be traced to the exact step/result.
- [ ] Failure reason is retained.

**M6 Commit Gate:** Successful and failed executions have complete traceable records.

---

# 11. M7 — TCS Import & Configuration Checklist

### Import

- [ ] ClosedXML is used as approved.
- [ ] Valid workbook imports correctly.
- [ ] Invalid workbook is rejected with useful validation information.
- [ ] Required fields are validated.
- [ ] Revision information is preserved.
- [ ] Historical revisions are not silently overwritten.

### Configuration

- [ ] TCS revision creation works.
- [ ] ATC revision creation works.
- [ ] Dataset creation works.
- [ ] Configuration selection works.
- [ ] Imported data can be prepared for execution.

### Tests

- [ ] Valid workbook test.
- [ ] Missing-field test.
- [ ] Invalid-value test.
- [ ] Duplicate/revision test.
- [ ] Representative real workbook test.

**M7 Commit Gate:** Representative TCS workbook can be imported and prepared for execution.

---

# 12. M8 — WPF Application UI Checklist

### Architecture

- [ ] WPF remains in presentation layer.
- [ ] MVVM pattern is followed.
- [ ] Business logic is not duplicated in ViewModels/code-behind.
- [ ] Application services are reused.
- [ ] State-machine logic is not duplicated in UI.

### UI Workflow

- [ ] Application starts correctly.
- [ ] Project selection works.
- [ ] TCS/configuration selection works.
- [ ] Test Run preparation works.
- [ ] Validation errors are visible.
- [ ] Execution status is visible.
- [ ] Current step/status is visible.
- [ ] Stop/cancel controls behave correctly.
- [ ] Results are accessible.
- [ ] Evidence is accessible.
- [ ] Errors are understandable to the operator.

### Usability

- [ ] No obvious blocking UI issues.
- [ ] Long-running operations do not freeze the UI.
- [ ] Error messages are actionable.
- [ ] Controls reflect current state correctly.

### Tests

- [ ] ViewModel tests where applicable.
- [ ] Application-service tests remain passing.
- [ ] Primary UI smoke workflow tested manually.

**M8 Commit Gate:** Primary operator workflow works without direct database interaction.

---

# 13. M9 — Embedded Linux/Qt Adapter Checklist

### Before Implementation

- [ ] Actual target communication/automation mechanism is confirmed.
- [ ] Required capabilities are available on the target.
- [ ] No mechanism has been invented without evidence.
- [ ] Any target limitation is documented.

### Adapter

- [ ] Target connection works.
- [ ] Application preparation/attach works where applicable.
- [ ] Object identification works.
- [ ] Required actions work.
- [ ] Required verification works.
- [ ] Timeout behavior works.
- [ ] Communication-loss behavior works.
- [ ] Reconnect/recovery behavior works where supported.
- [ ] Evidence capture works where required.
- [ ] Target-specific errors are translated into the adapter contract.

### Architecture

- [ ] Qt/Linux-specific code remains inside the adapter.
- [ ] Core does not reference target-specific APIs.
- [ ] State-machine rules are not duplicated in the adapter.

### Tests

- [ ] Adapter unit tests.
- [ ] Communication tests.
- [ ] Failure/timeout tests.
- [ ] Representative real-target test.

**M9 Commit Gate:** Representative Embedded Linux/Qt scenario passes end-to-end.

---

# 14. M10 — End-to-End Integration Checklist

- [ ] Full import → configuration → execution workflow passes.
- [ ] Successful execution passes.
- [ ] Functional failure passes expected outcome handling.
- [ ] Mapping failure/blocking passes.
- [ ] Object unavailable passes expected handling.
- [ ] Timeout passes expected handling.
- [ ] Communication loss passes expected handling.
- [ ] Stop behavior passes.
- [ ] Interrupted/recovery behavior passes.
- [ ] Re-execution passes.
- [ ] Application restart/recovery passes.
- [ ] Historical result immutability verified.
- [ ] Evidence traceability verified.
- [ ] Critical regression suite passes.
- [ ] No unresolved critical/high defects remain for V1 scope.

**M10 Commit Gate:** Critical end-to-end regression suite passes on supported targets.

---

# 15. M11 — Hardening & V1 Release Checklist

### Build/Packaging

- [ ] Release build succeeds.
- [ ] Installer/package works.
- [ ] Clean-machine installation tested.
- [ ] Required runtime/dependencies are included or documented.
- [ ] Configuration defaults are correct.

### Database/Recovery

- [ ] Fresh database initialization works.
- [ ] Backup works.
- [ ] Restore works.
- [ ] Application restart recovery verified.
- [ ] Corrupt/invalid database behavior is understood and handled.

### Testing

- [ ] Full unit test suite passes.
- [ ] Full integration test suite passes.
- [ ] Smoke tests pass.
- [ ] Recovery tests pass.
- [ ] Supported Windows environment tested.
- [ ] Supported Embedded Linux/Qt target tested where applicable.

### Quality

- [ ] No critical/high release-blocking defects remain.
- [ ] Logging is production-appropriate.
- [ ] Sensitive information is protected.
- [ ] Evidence is traceable.
- [ ] Performance is acceptable for V1 scope.
- [ ] No debug/test-only behavior is enabled accidentally.

### Documentation

- [ ] User/developer documentation matches implementation.
- [ ] Technology Stack is still accurate.
- [ ] Milestone status is updated.
- [ ] Release version is recorded.
- [ ] Known limitations are documented.

**M11 Release Gate:** V1 release candidate passes installation, smoke, regression and recovery checks.

---

# 16. Copilot Prompt — Module Completion Check

Use this before committing a completed module:

```text
Run a completion check for Mx.

Read:
- docs/implementation/IMPLEMENTATION_MILESTONES.md
- docs/implementation/IMPLEMENTATION_CHECKLISTS.md
- applicable architecture/design documents
- .github/copilot-instructions.md

Do not modify files.

Check:
1. milestone requirements
2. architecture compliance
3. automated tests
4. build status
5. error handling
6. security
7. dependency compliance
8. unnecessary/unrelated changes

Return a concise report:

STATUS: PASS or FAIL

Missing:
- ...

Warnings:
- ...

Tests:
- ...

Build:
- ...

Do not claim PASS if any mandatory criterion is not verified.
```

# 17. Copilot Prompt — Pre-Commit Review

```text
Perform a pre-commit review of the current changes.

Do not modify files.

Check the Common Checklist in:
 docs/implementation/IMPLEMENTATION_CHECKLISTS.md

Also check the current milestone checklist.

Review the Git diff for:
- unrelated changes
- architecture violations
- unapproved dependencies
- secrets/debug code
- missing tests
- weakened tests
- incorrect error handling
- documentation/version changes that are not justified

Return:
COMMIT READY: YES/NO

Blocking issues:
- ...

Warnings:
- ...

Tests verified:
- ...
```

# 18. Commit Decision Rule

Use the following rule for every milestone:

```text
Mandatory checklist item failed
        ↓
      NO COMMIT
        ↓
Fix → test → re-check

All mandatory items pass
        ↓
Review Git diff
        ↓
Commit
        ↓
Push
```

A Copilot-generated implementation is **not automatically accepted** because it builds or because Copilot says it is complete.

The developer owns the final commit decision.

# 19. Checklist Status

Use:

- `[ ]` Not checked
- `[x]` Verified/pass
- `[!]` Warning — review required
- `[BLOCKED]` Cannot verify because a dependency/decision is missing

Do not mark a requirement `[x]` without actual verification.

# 20. Changelog

### v0.1

- Added common pre-commit checklist.
- Added M0–M11 module-specific checklists.
- Added Copilot self-check and pre-commit prompts.
- Added commit decision rules and verification conventions.

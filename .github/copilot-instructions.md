# TCS Automation — Copilot Development Instructions

## 1. Project

TCS Automation V1 is a local desktop test automation system with pluggable target adapters for Windows and Embedded Linux/Qt targets.

## 2. Authoritative Documents

Use the repository documents as the source of truth. Read only the documents relevant to the current task:

- `docs/requirements/SRS.md`
- `docs/architecture/SYSTEM_ARCHITECTURE.md`
- `docs/architecture/DOMAIN_MODEL_AND_DATABASE_SCHEMA.md`
- `docs/architecture/SQL_SCHEMA.md`
- `docs/architecture/TEST_RUN_STATE_MACHINE.md`
- `docs/architecture/ADAPTER_INTERFACE_CONTRACT.md`
- `docs/implementation/TECHNOLOGY_STACK.md`
- `docs/implementation/IMPLEMENTATION_MILESTONES.md`

Do not invent requirements that are not present in these documents.

## 3. Technology Baseline

Use the approved technology baseline:

- C# / .NET 8
- WPF
- MVVM
- SQLite
- Dapper
- FlaUI / Windows UI Automation
- Serilog
- System.Text.Json
- ClosedXML
- xUnit
- NSubstitute
- Microsoft.Extensions.DependencyInjection
- Microsoft.Extensions.Configuration

Do not introduce another framework or infrastructure technology unless explicitly requested or required by a demonstrated implementation constraint.

## 4. Architecture Rules

1. Core/domain code must remain technology-independent.
2. Core must not depend on WPF.
3. Core/Application must not depend directly on FlaUI.
4. Database access must remain behind the Data Layer.
5. Evidence storage must remain behind the Evidence Store abstraction.
6. Target-specific automation must remain inside adapters.
7. Core/Application determines PASS/FAIL/BLOCKED outcomes.
8. Adapters report facts and standardized errors; they do not own business outcomes.
9. Follow Test Run State Machine v0.4.
10. Preserve immutable historical execution data.
11. Do not bypass the Adapter Interface Contract.
12. Do not duplicate state-machine business rules in UI code.

## 5. Implementation Rules

- Implement only the requested milestone/task.
- Prefer the smallest change that satisfies the requirement.
- Reuse existing classes/interfaces before creating new ones.
- Avoid unnecessary abstractions and speculative future functionality.
- Keep classes and methods focused.
- Follow existing naming and project conventions.
- Add or update tests with functional changes.
- Do not remove or weaken tests to make the build pass.
- Do not modify approved design documents merely for implementation convenience.
- Do not make unrelated refactoring while implementing a task.

## 6. Before Coding

1. Identify the relevant existing files.
2. Identify the applicable requirement/design rule.
3. Check whether the requested behavior already exists.
4. State a short implementation plan when the change is non-trivial.
5. Implement the smallest complete change.

For simple, unambiguous changes, do not spend tokens on a long explanation before coding.

## 7. Testing and Validation

After implementation:

- Build the affected projects.
- Run the relevant unit/integration tests.
- Add regression tests for bug fixes.
- Report build/test failures accurately.
- Do not claim success without actually running the applicable checks.

For milestone work, follow the checkpoint and exit criteria in `IMPLEMENTATION_MILESTONES.md`.

## 8. Design Conflict Rule

If implementation reveals a genuine contradiction between requirements, architecture, domain model, state machine, adapter contract, SQL schema, or technology baseline:

**STOP. Do not silently redesign.**

Report:

```text
CONFLICT:
DOCUMENT:
SECTION:
CURRENT IMPLEMENTATION:
PROBLEM:
MINIMAL RESOLUTION OPTIONS:
```

Only continue after the conflict is resolved.

## 9. Security and Evidence

- Never place credentials, passwords, tokens, or other secrets in source code.
- Do not expose sensitive dataset values in ordinary logs or evidence metadata.
- Preserve evidence traceability to Test Run, Attempt and Step.

## 10. AI Efficiency Rules

To keep AI usage efficient:

- Read only relevant files for the current task.
- Do not reproduce entire repository documents in responses.
- Do not explain obvious generated code unless requested.
- Prefer focused changes over large rewrites.
- Report concise results: changed files, tests, build result, and issues.
- Do not repeatedly re-review unrelated modules.
- Work milestone-by-milestone and checkpoint-by-checkpoint.

## 11. Git/Change Discipline

Keep commits logically small and related to one task or checkpoint. Do not combine unrelated changes.

Recommended commit style:

- `M0: Create solution structure`
- `M1: Add SQLite initialization`
- `M1: Add database tests`
- `M2: Implement Test Run state transitions`

## 12. Completion Rule

A task is complete only when:

1. The requested behavior is implemented.
2. Relevant tests pass.
3. The build passes for the affected solution/projects.
4. No known architecture or design conflict remains.
5. The implementation satisfies the applicable milestone checkpoint.

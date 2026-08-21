# TCS Automation — Test Run State Machine (Detailed)

**Version:** 0.4 — Design Draft
**Status:** Working Draft — candidate baseline for Domain Model v0.3 / SQL DDL
**Baseline:** SRS v1.2 / Architecture v0.2 / Domain Model & Database Schema v0.2

**Changelog since v0.3:**
- Fixed `PAUSED → INTERRUPTED`: this row previously implied an in-flight Attempt is marked `INTERRUPTED`, which directly contradicted I-12 ("a `PAUSED` Test Run has zero `EXECUTING` Attempts"). Corrected: no Attempt is touched by this transition (Sec 3.1).
- Rewrote I-10 to explicitly cover the pre-execution `BLOCKED` case (an Attempt with zero executed steps), instead of a pure step-derived algorithm that had no answer for it (Sec 11).
- Added an explicit safety invariant: an Attempt with zero executed steps can never aggregate to `PASS` — the only valid zero-step terminal state is pre-execution `BLOCKED` (Sec 11, folded into I-10).
- Replaced Outcome's precedence prose with an explicit, ordered algorithm (Sec 4).
- Added a rule for how a Stop request resolves against a pending Pause request: Stop always wins (Sec 3.2).
- Tightened Interruption Recovery's re-run scope from "did not reach terminal PASS/FAIL/BLOCKED" to explicitly "`NOT_EXECUTED` or `INTERRUPTED` only" — `STOPPED` Attempts are not automatically swept into recovery (Sec 8).
- Added an explicit rule for where a `CANDIDATE → MANUAL_CONFIRMED` mapping confirmation is recorded: against the run/execution context, not the canonical mapping record (Sec 5.2). Flagged as a Domain Model decision too.
- Added an explicit sentence clarifying that unstarted Step Results become terminal `NOT_EXECUTED` the moment their parent Attempt terminates as `STOPPED`/`INTERRUPTED` (Sec 5.1).
- Reworded I-03 for precision ("no terminal execution outcome without execution") and reworded the `RUNNING` state description (Sec 3, Sec 11).

## 1. Purpose

This document is the authoritative behavioral specification for how a **Test Run**, its **Execution Attempts**, and their **Step Results** move between states. It exists to remove ambiguity before coding starts, because the single biggest risk in this project is a state-handling bug that causes a genuine test failure to be reported as PASS, or an infrastructure problem to be silently reported as a test FAIL (SRS guiding principles, Sec 4).

This document does not redefine entities (see Domain Model v0.2) — it defines the **rules of motion** between the states that document already introduced.

## 2. The Three State Levels

```
TEST RUN                       (one execution round)
   └── EXECUTION ATTEMPT       (one attempt of one ATC/dataset within that run)
          └── STEP RESULT      (one logical step within that attempt)
```

State changes flow **downward** — a Step Result can never unilaterally change its parent Attempt's state, and an Attempt can never unilaterally change its parent Test Run's state, except by reporting information upward that the parent then acts on.

**Serial execution rule:** within a single Test Run, **at most** one Execution Attempt is `EXECUTING` at any moment. The Test Run can be `RUNNING` while zero Attempts are `EXECUTING` — during mapping confirmation (Sec 5.2), between one Attempt finishing and the next one's turn arriving, or at a pause boundary. `Test Run.status == RUNNING` must never be read as implying `Attempt.status == EXECUTING`.

## 3. Test Run States

| State | Meaning |
|---|---|
| `DRAFT` | Being assembled — ATCs, datasets, and configuration are still editable. |
| `READY_FOR_VALIDATION` | Engineer has requested validation of the assembled Test Run. |
| `VALIDATION_FAILED` | Pre-execution validation found a blocking problem (see Sec 7). |
| `READY` | Validation passed. The configuration snapshot has not yet been frozen. |
| `RUNNING` | The Test Run is in its execution phase. An Attempt may currently be `EXECUTING`, or the execution engine may be performing execution-phase orchestration (mapping resolution, transitioning between Attempts) with no Attempt actively executing. |
| `PAUSED` | Execution is temporarily suspended by the engineer; no Attempt is `EXECUTING` (I-12). |
| `COMPLETED` | Every selected Attempt reached `PASS`, `FAIL`, or `BLOCKED` — the run ended normally, with every selected item processed. |
| `STOPPED` | The engineer deliberately ended the run, OR the run ended with at least one selected Attempt left `NOT_EXECUTED`. |
| `INTERRUPTED` | Execution stopped **unexpectedly** (crash, power loss, unrecoverable adapter/target disconnect) rather than by deliberate engineer action. Terminal — see Sec 8. |

### 3.1 Test Run Transition Table

| From | To | Trigger | Guard / Precondition | Side Effects |
|---|---|---|---|---|
| — | `DRAFT` | Engineer creates a new Test Run | — | `test_run` row created |
| `DRAFT` | `READY_FOR_VALIDATION` | Engineer requests validation | At least one ATC selected | — |
| `READY_FOR_VALIDATION` | `VALIDATION_FAILED` | Validation completes with a blocking issue | See Sec 7 | Validation cycle record created |
| `READY_FOR_VALIDATION` | `READY` | Validation completes with no blocking issue | — | Validation cycle record created |
| `VALIDATION_FAILED` | `READY_FOR_VALIDATION` | Engineer fixes the issue and re-requests validation | — | — |
| `READY` | `DRAFT` | Engineer edits anything affecting execution validity | See Sec 3.3 | Prior validation cycle marked stale, not deleted |
| `READY` | `RUNNING` | Engineer starts execution | No edits since the last passing validation | Configuration snapshot frozen (Domain Model Sec 4.17); `Test Run ATC`/`Test Run Dataset` rows created |
| `RUNNING` | `PAUSED` | Engineer pauses | Pause takes effect only at the next step boundary, and only if no Stop is pending (Sec 3.2) | — |
| `PAUSED` | `RUNNING` | Engineer resumes | — | — |
| `RUNNING` | `COMPLETED` | Every selected Attempt reaches `PASS`/`FAIL`/`BLOCKED` | No selected Attempt is `NOT_EXECUTED` | `completed_at` set; Outcome computed (Sec 4) |
| `RUNNING` | `STOPPED` | Engineer deliberately stops (always takes precedence over a pending Pause, Sec 3.2), OR the run otherwise ends with ≥1 Attempt still `NOT_EXECUTED` | — | Currently-`EXECUTING` Attempt (if any) → Attempt-level `STOPPED`; its unstarted Step Results become terminal `NOT_EXECUTED`; other unstarted Attempts stay `NOT_EXECUTED`; `completed_at` set; Outcome computed |
| `PAUSED` | `STOPPED` | Engineer deliberately stops a paused run | No Attempt is `EXECUTING` (I-12) | Remaining unstarted Attempts stay `NOT_EXECUTED`; `completed_at` set; Outcome computed |
| `RUNNING` | `INTERRUPTED` | Unexpected failure the system cannot recover from | — | In-flight Attempt (if any) → Attempt-level `INTERRUPTED`; its unstarted Step Results become terminal `NOT_EXECUTED`; other unstarted Attempts stay `NOT_EXECUTED`; `completed_at` set; Outcome computed |
| `PAUSED` | `INTERRUPTED` | Unexpected failure while paused (e.g., application crash) | No Attempt is `EXECUTING` (I-12) | **No Attempt is marked `INTERRUPTED`** — by definition nothing was in-flight; existing terminal Attempts are untouched; unstarted Attempts stay `NOT_EXECUTED`; `completed_at` set; Outcome computed |

`COMPLETED`, `STOPPED`, and `INTERRUPTED` are all terminal for the Test Run record itself. Recovery from `INTERRUPTED` always creates a **new** Test Run (Sec 8); it never mutates the interrupted one.

### 3.2 Pause and Stop timing/race rules

A pause request is accepted immediately but does not take visible effect until the currently executing step reaches its own terminal state. This avoids a half-completed UI action being left undefined:

```
RUNNING
  │
  │ pause requested  → pause_requested_at recorded
  │ (Test Run remains RUNNING; UI shows "Pausing…")
  ▼
current step finishes (PASS/FAIL/BLOCKED)
  │
  │ pause_effective_at recorded
  ▼
PAUSED
```

**Stop always takes precedence over a pending Pause.** If the engineer clicks Stop while a Pause is still only "requested" (not yet effective), the pending pause is discarded and the run proceeds directly to `STOPPED` via the normal stop rules — it never becomes `PAUSED` first. This prevents an engineer who changed their mind from Pause to Stop from ending up in an unintended `PAUSED` state.

### 3.3 READY staleness rule

Validation certifies a *specific* configuration. If any of that changes while the Test Run sits in `READY`, the previously-passed validation no longer certifies what would actually execute. Therefore any such edit forces `READY → DRAFT`, and the engineer must re-run validation before `RUNNING` is reachable again. The stale validation cycle record is kept (not deleted) for audit purposes but marked superseded.

## 4. Test Run Outcome (separate from State)

**State** answers "where is this run in its lifecycle." **Outcome** answers "what did the run find." They are tracked separately because a `COMPLETED` (or `STOPPED`) run can have found a defect, an infrastructure problem, or nothing wrong.

| Outcome | Meaning |
|---|---|
| `PASS` | Every selected Attempt is `PASS`. |
| `FAIL` | At least one selected Attempt is `FAIL`. |
| `BLOCKED` | No Attempt is `FAIL`, but at least one is `BLOCKED`. |
| `PARTIAL` | The run ended before every selected Attempt was processed, and neither `FAIL` nor `BLOCKED` applies. |
| `NOT_EVALUATED` | No Attempt ever reached `EXECUTING` or the pre-execution `BLOCKED` gate — the run never left the preparation states. |

**Outcome algorithm** (evaluated in order; first match wins):

```
1. If any Attempt = FAIL                                    → FAIL
2. Else if any Attempt = BLOCKED                             → BLOCKED
3. Else if Test Run ∈ {STOPPED, INTERRUPTED} and any Attempt
   is NOT_EXECUTED, STOPPED, or INTERRUPTED                  → PARTIAL
4. Else if every selected Attempt = PASS                     → PASS
5. Otherwise (no Attempt ever executed)                      → NOT_EVALUATED
```

**Worked examples:**

```
ATC1=PASS,  ATC2=STOPPED                         → Outcome: PARTIAL
ATC1=FAIL,  ATC2=STOPPED                         → Outcome: FAIL
ATC1=BLOCKED, ATC2=STOPPED                       → Outcome: BLOCKED
ATC1=PASS,  ATC2=PASS,  ATC3=PASS                → Outcome: PASS
ATC1=FAIL,  ATC2=BLOCKED                         → Outcome: FAIL
```

`FAIL` and `BLOCKED` both take precedence over `PARTIAL`: if there is already concrete evidence of a defect or a blocking infrastructure problem, that must not be hidden just because the run was later stopped or interrupted.

Outcome is computed once, when the Test Run reaches a terminal State, and never recalculated afterward (I-06).

**Note on abandoned Drafts:** a Test Run left in `DRAFT` and never progressed has no Outcome — it is not treated as an executable or historical run. No `CANCELLED` state is introduced; nothing in the SRS requires tracking cancellation as distinct from an unfinished draft.

## 5. Execution Attempt States

| State | Meaning |
|---|---|
| `NOT_EXECUTED` | Selected for this run but never started. |
| `EXECUTING` | Actively running steps. |
| `PASS` | All steps passed. |
| `FAIL` | At least one step genuinely failed the expected result — a real product defect, with no step `BLOCKED`. |
| `BLOCKED` | Valid verification could not be completed — either before execution began (unresolved mapping) or during it (mapping/communication/configuration/automation/environment failure) — not a product defect. |
| `STOPPED` | Was `EXECUTING` when the engineer deliberately stopped the parent Test Run. |
| `INTERRUPTED` | Was `EXECUTING` when the parent Test Run was unexpectedly interrupted. |

### 5.1 Execution Attempt Transition Table

| From | To | Trigger | Guard | Side Effects |
|---|---|---|---|---|
| — | `NOT_EXECUTED` | Created when Test Run enters `RUNNING` | — | `execution_attempt` row created for each selected ATC × dataset (or default placeholder, Domain Model Sec 4.20); `attempt_number` assigned (Domain Model Sec 4.21) |
| `NOT_EXECUTED` | `EXECUTING` | This Attempt's turn arrives and its mapping resolves to `STRONG`, `MANUAL_CONFIRMED`, or an engineer-confirmed `CANDIDATE` (Sec 5.2) | — | `started_at` set |
| `NOT_EXECUTED` | `BLOCKED` | This Attempt's turn arrives and its mapping is `UNRESOLVED`, or `CANDIDATE` and the engineer rejects confirmation (Sec 5.2) | Failure category `MAPPING_EXCEPTION` | `completed_at` set; `started_at` is **not** set — this Attempt never entered `EXECUTING` (bounded exception, see I-03/I-10) |
| `EXECUTING` | `PASS` | Last step completes `PASS`, no prior step `FAIL`/`BLOCKED` | — | `completed_at` set |
| `EXECUTING` | `FAIL` | A step's actual result genuinely did not match expected | Failure category is `TEST_FAILURE` (Sec 5.3) | Remaining steps → `NOT_EXECUTED`; `completed_at` set |
| `EXECUTING` | `BLOCKED` | A step could not be validly verified for a non-product reason *after execution had already started* | Failure category is `AUTOMATION_FAILURE` / `MAPPING_EXCEPTION` / `COMMUNICATION_FAILURE` / `CONFIGURATION_FAILURE` / `ENVIRONMENT_FAILURE` | Remaining steps → `NOT_EXECUTED`; `completed_at` set |
| `EXECUTING` | `STOPPED` | Engineer deliberately stops the parent Test Run while this Attempt is mid-step | — | In-flight step → `STOPPED`; remaining steps → terminal `NOT_EXECUTED`; `completed_at` set |
| `EXECUTING` | `INTERRUPTED` | Parent Test Run is unexpectedly interrupted while this Attempt is mid-step | — | In-flight step → `INTERRUPTED`; remaining steps → terminal `NOT_EXECUTED`; `completed_at` set |
| `NOT_EXECUTED` | `NOT_EXECUTED` (terminal for this run) | Parent Test Run reaches `COMPLETED`/`STOPPED`/`INTERRUPTED` before this Attempt's turn | — | No `started_at` is ever set |

**Terminal cascade rule:** whenever an Attempt terminates as `STOPPED` or `INTERRUPTED`, every Step Result that had not yet started remains `NOT_EXECUTED` and becomes terminal at that same moment — it will never later execute, because its parent Attempt has terminated (I-01).

Note the two distinct routes to `BLOCKED`: a **pre-execution** route (`NOT_EXECUTED → BLOCKED`, mapping never resolved, zero steps ever ran) and a **mid-execution** route (`EXECUTING → BLOCKED`, something went wrong after a valid start). Both are `BLOCKED`, not `FAIL`, and both are equally visible in reporting.

### 5.2 Mapping resolution gate

Pre-execution validation (Sec 7) checks mapping *existence*, not final confirmation — a `CANDIDATE` mapping is a non-blocking validation finding. An Attempt only enters `EXECUTING` once its mapping is actually resolved to something usable.

```
NOT_EXECUTED
     │  (this Attempt's turn arrives)
     ▼
mapping confidence state?
     ├── STRONG             → EXECUTING
     ├── MANUAL_CONFIRMED    → EXECUTING
     ├── CANDIDATE           → engineer confirmation prompt
     │                           ├── Confirm → EXECUTING
     │                           └── Reject  → BLOCKED directly
     │                                          (category: MAPPING_EXCEPTION,
     │                                           never entered EXECUTING)
     └── UNRESOLVED           → BLOCKED directly
                                 (category: MAPPING_EXCEPTION,
                                  never entered EXECUTING)
```

**Where the confirmation is recorded:** confirming a `CANDIDATE` mapping during a run does **not** implicitly modify the canonical `Object Mapping`/`Mapping Revision` record (Domain Model Sec 4.12/4.13). It is recorded against the run/execution context only — conceptually, an `execution_attempt.mapping_resolution` (or a dedicated `test_run_mapping_resolution` record with `mapping_id`, `test_run_id`, `resolved_confidence`, `resolved_by`, `resolved_at`). A future run against the same mapping sees it as `CANDIDATE` again and requires its own confirmation, unless a separate, explicit workflow promotes the canonical mapping to `MANUAL_CONFIRMED`. This preserves historical immutability and avoids one engineer's in-run judgment call silently changing the mapping for everyone. *(This is a Domain Model v0.3 decision as much as a state-machine one — flagged in Sec 13.)*

The confirmation prompt is not a formal Attempt state — it is a sub-step of Attempt turn-taking, during which the Test Run remains `RUNNING` with zero Attempts `EXECUTING`.

### 5.3 The FAIL vs. BLOCKED decision — this is the core safety rule

An Attempt is only ever marked `FAIL` when the *product under test* produced a result that genuinely did not match the expected result, and the tooling has reasonable confidence the comparison itself was valid. Every other reason a step could not be validly completed must produce `BLOCKED`, never `FAIL`.

**Rule of thumb for implementers:** if you are not certain the comparison was valid, the answer is `BLOCKED`, not `FAIL`.

## 6. Step Result States

| State | Meaning |
|---|---|
| `NOT_EXECUTED` | Not yet reached, or the Attempt ended before this step ran. Becomes terminal once the parent Attempt reaches a terminal state (I-01). |
| `PASS` | Actual result matched expected result. |
| `FAIL` | Actual result did not match expected result (genuine product behavior). |
| `BLOCKED` | Could not be validly verified (Sec 5.3 categories). |
| `STOPPED` | Was actively executing when the engineer deliberately stopped the run. |
| `INTERRUPTED` | Was actively executing when the run was unexpectedly interrupted. |

The first step in an Attempt to reach a terminal, non-`PASS` state determines that Attempt's terminal state; every step after it is set to `NOT_EXECUTED`.

## 7. Pre-Execution Validation (READY_FOR_VALIDATION → READY / VALIDATION_FAILED)

Validation is **repeatable** — a Test Run may cycle through `READY_FOR_VALIDATION ↔ VALIDATION_FAILED` multiple times, and may return to `DRAFT` per Sec 3.3. Each validation attempt is recorded:

- `validation_id`, `test_run_id`, `validated_at`, `validator_version`, `result` (`PASSED`/`FAILED`), `findings`

**Blocking findings (→ `VALIDATION_FAILED`):**
- Selected ATC references a mapping that is `UNRESOLVED` with no fallback.
- Selected Test Configuration references an application/module version that no longer exists.
- Required target/adapter is unreachable at validation time.
- TCS/ATC revision referenced by the selection has since been retired without a replacement.

**Non-blocking findings (→ `READY`, surfaced as warnings):**
- A mapping is `CANDIDATE` (resolved at execution time per Sec 5.2, not at validation time).
- A module version is present but has no recorded compatibility notes.

The exhaustive rule list is finalized during detailed design.

## 8. Interruption Recovery — What Happens Next

An `INTERRUPTED` Test Run is a terminal record — it never transitions again, and it is never edited (I-06). On the next launch, the tool must:

1. Detect the `INTERRUPTED` Test Run and surface it explicitly.
2. Present the engineer with an explicit choice, each of which creates a **new** Test Run in `DRAFT`, linked back to the original via `parent_test_run_id`:
   - **Start a new Test Run** using the same selection.
   - **Re-run only the Attempts whose final state is `NOT_EXECUTED` or `INTERRUPTED`** — not `STOPPED`. A `STOPPED` Attempt reflects a deliberate decision, not an unexpected gap, so it is not automatically swept into interruption recovery. (If the engineer wants to re-run a deliberately stopped Attempt too, that is a separate, explicit re-run action, not part of this recovery flow.)
   - **Close out the interrupted run as-is** — no new Test Run is created; the `INTERRUPTED` record stands, with Outcome `PARTIAL`.
3. The original `INTERRUPTED` Test Run and its Attempts/Step Results are never edited or deleted (I-06).

`parent_test_run_id` is a generic run-to-run lineage link, not conceptually restricted to interruption recovery — the same field can represent a deliberate re-execution of a `COMPLETED` run, or multiple recovery attempts branching from one interrupted run.

> **Domain Model follow-up required:** `parent_test_run_id` does not yet exist on the `Test Run` entity (Domain Model v0.2, Sec 4.16). Must be added before SQL DDL.

## 9. Timeout Handling

A step that does not respond within its timeout threshold is **not** automatically marked `FAIL`. `FAIL` is permitted only when the verification operation itself was successfully performed and the observed behavior can be validly compared against the expected result. A connected-but-malfunctioning target that never actually produces a comparable result resolves to `BLOCKED`, not `FAIL`.

```
Step timeout reached
        │
        ▼
Was the verification operation itself successfully
performed, with a result that can be validly compared?
        │
   ┌────┴────┐
  Yes         No
   │           │
   ▼           ▼
Evaluate as   BLOCKED
FAIL only if  (category depends on cause)
the mismatch
is genuine
```

Exact timeout threshold values are deferred to detailed design (Sec 13 #3).

## 10. Worked Example

```
Test Run: Round-07
  ATC-001 / Dataset-01  → EXECUTING → step1 PASS → step2 PASS → step3 PASS → PASS
  ATC-004 / Dataset-01  → EXECUTING → step1 PASS → step2 FAIL → Attempt = FAIL
  ATC-009 / Dataset-01  → EXECUTING → step1 PASS → step2: target disconnected
                           mid-step → BLOCKED (COMMUNICATION_FAILURE) →
                           Attempt = BLOCKED, step3 = NOT_EXECUTED
  ATC-012 / Dataset-01  → still NOT_EXECUTED when engineer clicks Stop
                           → Test Run = STOPPED

Test Run Round-07 final:
  State:   STOPPED   (ATC-012 never started)
  Outcome: FAIL      (FAIL takes precedence, Sec 4)

  ATC-001 → PASS
  ATC-004 → FAIL          (real defect — reported to dev team)
  ATC-009 → BLOCKED       (infra issue — not a defect)
  ATC-012 → NOT_EXECUTED  (never ran)
```

## 11. State Machine Invariants

- **I-01 — Terminal immutability.** Once a Step Result, Execution Attempt, or Test Run reaches its terminal state, it cannot transition again. For Step Results, `NOT_EXECUTED` becomes terminal once its parent Attempt itself reaches a terminal state.
- **I-02 — No execution after terminal state.** An Attempt in a terminal state can never execute another Step Result.
- **I-03 — No terminal execution outcome without execution, with one bounded exception.** An Attempt cannot become `PASS` or `FAIL` without first entering `EXECUTING`. An Attempt can become a pre-execution `BLOCKED` *only* through the mapping-resolution gate (Sec 5.2) — never through any other path.
- **I-04 — FAIL requires valid verification.** `FAIL` requires failure category `TEST_FAILURE` and a successfully performed, validly comparable verification operation.
- **I-05 — BLOCKED excludes product defect.** `BLOCKED` must never be used for a verified product mismatch.
- **I-06 — Historical immutability.** Completed/terminated Test Run, Attempt, and Step Result records are never modified after reaching a terminal state.
- **I-07 — At most one active Attempt per Test Run.** Attempts never execute concurrently; the Test Run may be `RUNNING` with zero Attempts `EXECUTING`.
- **I-08 — One active Test Run per Test PC.** Per SRS NFR-013.
- **I-09 — Snapshot immutability.** After `READY → RUNNING`, the execution snapshot cannot change for the life of that Test Run.
- **I-10 — Attempt aggregation algorithm.** An Attempt's terminal state is determined as follows:
  - **If the Attempt entered `EXECUTING`** (has at least one Step Result), its terminal state is derived from its steps in this priority order: `INTERRUPTED` if any step is `INTERRUPTED`; else `STOPPED` if any step is `STOPPED`; else `FAIL` if any step is `FAIL`; else `BLOCKED` if any step is `BLOCKED`; else `PASS` if every executed step is `PASS`.
  - **If the Attempt never entered `EXECUTING`** (zero Step Results), its only valid terminal state is `BLOCKED` via the pre-execution mapping-resolution gate (Sec 5.2), with `failure_category = MAPPING_EXCEPTION`. An Attempt with zero executed steps can **never** aggregate to `PASS` — there is no code path that produces this, and no implementation should introduce one.
- **I-11 — No orphan execution state.** An `EXECUTING` Attempt must belong to a `RUNNING` Test Run. An in-progress Step Result must belong to an `EXECUTING` Attempt.
- **I-12 — Pause consistency.** A `PAUSED` Test Run must have zero Attempts in `EXECUTING`. Consequently, a transition out of `PAUSED` (to `RUNNING`, `STOPPED`, or `INTERRUPTED`) never touches any Attempt's execution state directly — see the `PAUSED → INTERRUPTED` row in Sec 3.1.

## 12. Atomic Transition Requirement

Every state transition at every level must be implemented as an atomic, current-state-checked write — a compare-and-swap: "update to new state only if the record is still in the expected prior state." If the check fails, the write is rejected and the caller re-reads the current state.

**A rejected transition must never attempt a compensating overwrite of the newly observed state.** If an execution thread's `EXECUTING → PASS` write loses the race against a UI thread's `EXECUTING → STOPPED` write, the execution thread must accept `STOPPED` as final — never retry, force, or "correct" the record back toward `PASS`.

This is required even in the single-user, single-PC V1 scope, because the UI thread and the execution/adapter thread are still two independent writers to the same record. Exact implementation is a detailed-design decision; the atomicity requirement itself is not.

## 13. Open Design Decisions

1. Exact set of blocking vs. non-blocking pre-execution validation rules (Sec 7).
2. **Domain Model addition required:** `parent_test_run_id` on the `Test Run` entity (Sec 8).
3. Timeout threshold values (Sec 9).
4. Whether `PAUSED` has a maximum duration before the tool prompts the engineer.
5. Exact mechanism for the atomic transition requirement (Sec 12).
6. **Domain Model decision required:** exact shape of the run-scoped mapping confirmation record (Sec 5.2) — `execution_attempt.mapping_resolution` field vs. a dedicated `test_run_mapping_resolution` table.

## 14. Acceptance Questions for Review

1. Does every state have a defined trigger to enter it and at least one defined trigger to leave it (except terminal states)?
2. Is it always possible to tell, from stored data alone, whether an Attempt never started, was deliberately stopped, or was unexpectedly interrupted?
3. Does the FAIL vs. BLOCKED rule leave any ambiguous case?
4. Can a `PAUSED` run ever leave a step in a half-completed, unrecorded state, or produce an `INTERRUPTED` Attempt while paused?
5. Does the recovery flow ever allow an `INTERRUPTED` run to be silently treated as complete, mutated, or have a deliberately `STOPPED` Attempt swept into automatic recovery?
6. Does I-10 produce an unambiguous answer for every possible Attempt, including one that never executed?
7. Is a Stop request's precedence over a pending Pause request explicit and unambiguous?
8. Is it clear that confirming a `CANDIDATE` mapping during one run does not silently change the mapping for all future runs?

---

**Status:** Test Run State Machine Draft v0.4 — candidate baseline.
**Next artifact after review:** Domain Model v0.3 (add `parent_test_run_id`, validation history table, Test Run `outcome` field, run-scoped mapping resolution record), then Detailed SQL schema (DDL), then Adapter interface contract.

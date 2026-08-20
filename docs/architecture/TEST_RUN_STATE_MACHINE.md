# TCS Automation — Test Run State Machine (Detailed)

**Version:** 0.1 — Design Draft
**Status:** Working Draft
**Baseline:** SRS v1.2 / Architecture v0.2 / Domain Model & Database Schema v0.2

## 1. Purpose

This document is the authoritative behavioral specification for how a **Test Run**, its **Execution Attempts**, and their **Step Results** move between states. It exists to remove ambiguity before coding starts, because the single biggest risk in this project is a state-handling bug that causes a genuine test failure to be reported as PASS, or an infrastructure problem to be silently reported as a test FAIL (SRS guiding principles, Sec 4).

This document does not redefine entities (see Domain Model v0.2) — it defines the **rules of motion** between the states that document already introduced.

## 2. The Three State Levels

TCS Automation tracks state at three nested levels, and a change at one level can force a change at the levels below it:

```
TEST RUN                       (one execution round)
   └── EXECUTION ATTEMPT       (one attempt of one ATC/dataset within that run)
          └── STEP RESULT      (one logical step within that attempt)
```

A Test Run can contain many Execution Attempts (one per selected ATC × dataset). An Execution Attempt can contain many Step Results. State changes generally flow **downward** (a Test Run being stopped forces its in-flight attempts to a terminal state) — a Step Result can never unilaterally change its parent Attempt's state, and an Attempt can never unilaterally change its parent Test Run's state, except by reporting information upward (e.g., "I failed") that the parent then acts on.

## 3. Test Run States

| State | Meaning |
|---|---|
| `DRAFT` | Being assembled — ATCs, datasets, and configuration are still editable. |
| `READY_FOR_VALIDATION` | Engineer has requested validation of the assembled Test Run. |
| `VALIDATION_FAILED` | Pre-execution validation found a blocking problem (see Sec 6). |
| `READY` | Validation passed. The configuration snapshot has not yet been frozen. |
| `RUNNING` | Actively executing Attempts. |
| `PAUSED` | Execution is temporarily suspended by the engineer; no Attempt is actively executing. |
| `COMPLETED` | All selected Attempts reached a terminal state; the run ended normally. |
| `STOPPED` | The engineer deliberately ended the run before all Attempts reached a terminal state. |
| `INTERRUPTED` | Execution stopped **unexpectedly** (crash, power loss, adapter/target disconnect the system could not recover from) rather than by deliberate engineer action. |

### 3.1 Test Run Transition Table

| From | To | Trigger | Guard / Precondition | Side Effects |
|---|---|---|---|---|
| — | `DRAFT` | Engineer creates a new Test Run | — | `test_run` row created |
| `DRAFT` | `READY_FOR_VALIDATION` | Engineer requests validation | At least one ATC selected | — |
| `READY_FOR_VALIDATION` | `VALIDATION_FAILED` | Pre-execution validation completes with a blocking issue | See Sec 6 for blocking vs. non-blocking findings | Validation report recorded |
| `READY_FOR_VALIDATION` | `READY` | Pre-execution validation completes with no blocking issue | — | Validation report recorded |
| `VALIDATION_FAILED` | `READY_FOR_VALIDATION` | Engineer fixes the issue and re-requests validation | — | — |
| `READY` | `RUNNING` | Engineer starts execution | Configuration snapshot is frozen at this transition (Domain Model Sec 4.17) | Snapshot rows created; `Test Run ATC`/`Test Run Dataset` rows created |
| `RUNNING` | `PAUSED` | Engineer pauses | No Attempt may be left mid-step when the pause takes effect — see Sec 3.2 | — |
| `PAUSED` | `RUNNING` | Engineer resumes | — | — |
| `RUNNING` | `COMPLETED` | Last selected Attempt reaches a terminal state | Every selected Attempt is `PASS`/`FAIL`/`BLOCKED`/`NOT_EXECUTED` | `completed_at` set |
| `RUNNING` | `STOPPED` | Engineer deliberately stops the run | — | In-flight Attempt(s) forced to terminal state per Sec 4.4; `completed_at` set |
| `PAUSED` | `STOPPED` | Engineer deliberately stops a paused run | — | Remaining unstarted Attempts marked `NOT_EXECUTED`; `completed_at` set |
| `RUNNING` | `INTERRUPTED` | Unexpected failure the system cannot recover from (crash, power loss, unrecoverable disconnect) | — | In-flight Attempt(s) forced to `INTERRUPTED` per Sec 4.4; `completed_at` set |
| `PAUSED` | `INTERRUPTED` | Unexpected failure while paused (e.g., application crash) | — | Same as above |
| `INTERRUPTED` | `READY` | Engineer explicitly chooses to resume/re-prepare | Requires explicit engineer decision — never automatic (SRS FR-065) | A **new** Test Run or a continuation is created per Sec 7; never silently resumes into the old run as if nothing happened |

### 3.2 Why "Pause" cannot interrupt a step mid-action

A pause request is accepted immediately but does not take visible effect until the **currently executing step** reaches its own terminal state (PASS/FAIL/BLOCKED). This avoids a half-completed UI action (e.g., a button click sent but the resulting screen not yet verified) being left in an undefined state. The Test Run stays `RUNNING` for the brief interval between "pause requested" and "pause effective"; the UI should indicate "Pausing…" during this window.

## 4. Execution Attempt States

| State | Meaning |
|---|---|
| `NOT_EXECUTED` | Selected for this run but has not started (including: run ended/stopped before this Attempt's turn came up). |
| `EXECUTING` | Actively running steps. |
| `PASS` | All steps passed. |
| `FAIL` | At least one step genuinely failed the expected result (a real product defect or unexpected behavior), and no step was BLOCKED. |
| `BLOCKED` | Valid verification could not be completed due to mapping, communication, configuration, automation, or environment conditions (Domain Model Sec 4.23 failure categories) — **not** a product defect. |
| `INTERRUPTED` | Was `EXECUTING` when its parent Test Run was unexpectedly interrupted or deliberately stopped. |

### 4.1 Execution Attempt Transition Table

| From | To | Trigger | Guard | Side Effects |
|---|---|---|---|---|
| — | `NOT_EXECUTED` | Created when Test Run enters `RUNNING` (snapshot/selection frozen) | — | `execution_attempt` row created for each selected ATC × dataset (or default placeholder, Domain Model Sec 4.20) |
| `NOT_EXECUTED` | `EXECUTING` | Adapter begins the first step of this Attempt | Object mapping for this ATC must be resolved (`STRONG`/`MANUAL_CONFIRMED`) or the Attempt goes straight to `BLOCKED` (see 4.2) | `started_at` set |
| `EXECUTING` | `PASS` | Last step completes with `PASS` and no prior step in this Attempt was `FAIL`/`BLOCKED` | — | `completed_at` set |
| `EXECUTING` | `FAIL` | A step's actual result did not match its expected result (genuine mismatch, not an infrastructure problem) | The failing category recorded on the `Failure` record must be `TEST_FAILURE` — see 4.2 | Remaining steps in this Attempt marked `NOT_EXECUTED`; `completed_at` set |
| `EXECUTING` | `BLOCKED` | A step could not be validly executed/verified for a non-product reason | Failure category is one of `AUTOMATION_FAILURE`, `MAPPING_EXCEPTION`, `COMMUNICATION_FAILURE`, `CONFIGURATION_FAILURE`, `ENVIRONMENT_FAILURE` | Remaining steps marked `NOT_EXECUTED`; `completed_at` set |
| `EXECUTING` | `INTERRUPTED` | Parent Test Run transitions to `STOPPED` or `INTERRUPTED` while this Attempt is mid-step | — | The in-flight step is marked `INTERRUPTED` too (Sec 5); `completed_at` set |
| `NOT_EXECUTED` | `NOT_EXECUTED` (terminal, unchanged) | Parent Test Run reaches `COMPLETED`/`STOPPED` before this Attempt's turn | — | No `started_at` is ever set |

### 4.2 The FAIL vs. BLOCKED decision — this is the core safety rule

An Attempt is only ever marked `FAIL` when the *product under test* produced a result that genuinely did not match the expected result, and the tooling has reasonable confidence the comparison itself was valid (correct object was checked, adapter/target communication was healthy throughout, mapping was `STRONG`/`MANUAL_CONFIRMED`, not `CANDIDATE`/`UNRESOLVED`).

Every other reason a step could not be validly completed — the target disconnected, the mapped object could no longer be found, the adapter itself errored, the environment/configuration was wrong — must produce `BLOCKED`, never `FAIL`. This directly implements the SRS guiding principle that automation must never silently misreport an infrastructure problem as a product defect.

**Rule of thumb for implementers:** if you are not certain the comparison was valid, the answer is `BLOCKED`, not `FAIL`. `FAIL` requires positive confidence, not just the absence of a recognized error.

## 5. Step Result States

| State | Meaning |
|---|---|
| `NOT_EXECUTED` | Not yet reached, or the Attempt ended before this step ran. |
| `PASS` | Actual result matched expected result. |
| `FAIL` | Actual result did not match expected result (genuine product behavior). |
| `BLOCKED` | Could not be validly verified (see Sec 4.2 categories). |
| `INTERRUPTED` | Was actively executing when the parent Test Run was stopped/interrupted. |

Step Result states cascade upward exactly once: the first step in an Attempt to reach `FAIL`, `BLOCKED`, or `INTERRUPTED` determines that Attempt's terminal state (Sec 4.1), and every step after it in that Attempt is set to `NOT_EXECUTED` — steps are never executed after their Attempt has already reached a terminal outcome.

## 6. Pre-Execution Validation (READY_FOR_VALIDATION → READY / VALIDATION_FAILED)

Validation runs once, before any Attempt starts, and checks things that would make every subsequent Attempt meaningless if wrong. This is intentionally separate from per-Attempt execution failures.

**Blocking findings (→ `VALIDATION_FAILED`):**
- Selected ATC references a mapping that is `UNRESOLVED` with no fallback.
- Selected Test Configuration references an application/module version that no longer exists.
- Required target/adapter is unreachable at validation time.
- TCS/ATC revision referenced by the selection has since been retired without a replacement.

**Non-blocking findings (→ `READY`, but surfaced as warnings):**
- A mapping is `CANDIDATE` (will require in-run confirmation rather than blocking the whole run).
- A module version is present but has no recorded compatibility notes.

Full validation rule set is finalized during detailed design; this section defines the *category* split (blocking vs. warning), not the exhaustive rule list.

## 7. Interruption Recovery — What Happens Next

When a Test Run reaches `INTERRUPTED`, the system must never auto-resume as if nothing happened (SRS FR-065). On the next launch (or when the engineer next opens the project), the tool must:

1. Detect the `INTERRUPTED` Test Run and surface it explicitly — it does not disappear into history silently.
2. Present the engineer with an explicit choice:
   - **Start a new Test Run** using the same selection (previous Attempts remain in the interrupted run's history, untouched).
   - **Re-run only the Attempts that did not reach a terminal PASS/FAIL/BLOCKED** (i.e., those still `NOT_EXECUTED` or `INTERRUPTED`) as a new Test Run, cross-referenced to the original.
   - **Close out the interrupted run as-is**, accepting the partial results and marking it permanently `INTERRUPTED` in history.
3. Whichever choice is made, the original `INTERRUPTED` Test Run record and its Attempts/Step Results are never edited or deleted (Historical Integrity Rule 1, Domain Model Sec 7) — only new records are created.

The exact UI flow for this decision is a detailed-design item; the rule that it must always be an explicit engineer decision, never automatic, is not.

## 8. Worked Example

```
Test Run: Round-07
  ATC-001 / Dataset-01  → RUNNING → step 1 PASS → step 2 PASS → step 3 PASS → PASS
  ATC-004 / Dataset-01  → RUNNING → step 1 PASS → step 2 FAIL (expected label
                           "Ready", actual "Error") → PASS/PASS/FAIL → Attempt = FAIL
  ATC-009 / Dataset-01  → RUNNING → step 1 PASS → step 2: target disconnected
                           mid-step → step 2 = BLOCKED (COMMUNICATION_FAILURE) →
                           Attempt = BLOCKED, step 3 = NOT_EXECUTED
  ATC-012 / Dataset-01  → still NOT_EXECUTED when engineer clicks Stop
                           → Test Run = STOPPED → ATC-012 stays NOT_EXECUTED
                             (it never started, so it is not INTERRUPTED)

Test Run Round-07 final status: STOPPED
  ATC-001 → PASS
  ATC-004 → FAIL          (real defect — reported to dev team)
  ATC-009 → BLOCKED       (infra issue — not a defect, target reconnect needed)
  ATC-012 → NOT_EXECUTED  (never ran)
```

This example shows why `NOT_EXECUTED` (never started) and `INTERRUPTED` (was executing, got cut off) must remain distinct: ATC-012 needs no special recovery flow, but an Attempt that was mid-step when Round-07 stopped would need the Sec 7 recovery flow.

## 9. Open Design Decisions

Deferred to detailed design / implementation:

1. Exact set of blocking vs. non-blocking pre-execution validation rules (Sec 6 gives categories, not the full list).
2. Whether "re-run only incomplete Attempts" (Sec 7) creates a new Test Run that references the original, or a special "continuation" run type — the domain model doesn't yet have a `parent_test_run_id` concept.
3. Timeout thresholds that turn a slow/unresponsive step into a `COMMUNICATION_FAILURE`/`BLOCKED` outcome rather than waiting indefinitely.
4. Whether `PAUSED` has a maximum duration before the tool prompts the engineer (e.g., "this run has been paused for 2 hours — resume or stop?").

## 10. Acceptance Questions for Review

1. Does every Test Run state have a defined trigger to enter it and at least one defined trigger to leave it (except terminal states)?
2. Is it always possible to tell, from stored data alone, whether an Attempt never started (`NOT_EXECUTED`) versus started and got cut off (`INTERRUPTED`)?
3. Does the FAIL vs. BLOCKED rule (Sec 4.2) leave any ambiguous case where an implementer would have to guess?
4. Can a `PAUSED` run ever leave a step in a half-completed, unrecorded state?
5. Does the recovery flow (Sec 7) ever allow an `INTERRUPTED` run to be silently treated as complete?

---

**Status:** Test Run State Machine Draft v0.1 — ready for review.
**Next artifact after review:** Detailed SQL schema (DDL), then Adapter interface contract.

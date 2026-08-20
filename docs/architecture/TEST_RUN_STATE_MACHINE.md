# TCS Automation — Test Run State Machine (Detailed)

**Version:** 0.3 — Design Draft
**Status:** Working Draft — candidate baseline
**Baseline:** SRS v1.2 / Architecture v0.2 / Domain Model & Database Schema v0.2

**Changelog since v0.2:**
- Changed the serial-execution rule from "exactly one Attempt EXECUTING" to "**at most** one Attempt EXECUTING" (Sec 2), and added an explicit note that `RUNNING` does not guarantee an Attempt is currently executing (Sec 2, 5.2).
- Removed the artificial `EXECUTING → BLOCKED` detour for `UNRESOLVED` mappings and rejected `CANDIDATE` confirmations — these now go directly `NOT_EXECUTED → BLOCKED` without ever entering `EXECUTING` (Sec 5.1, 5.2). Added this as an explicit, bounded exception to invariant I-03 rather than leaving a silent contradiction.
- Simplified the `PARTIAL` Outcome definition to avoid the "terminal vs. non-terminal" framing (Sec 4).
- Added explicit Outcome precedence worked examples (Sec 4).
- Tightened FAIL's evidentiary bar in the timeout rule: from "communication was healthy" to "the verification operation itself was successfully performed and the observed behavior can be validly compared" (Sec 9).
- Corrected I-01 to account for `NOT_EXECUTED` becoming terminal for a Step Result once its parent Attempt terminates (Sec 11).
- Expanded I-10 into a full, unambiguous aggregation algorithm covering `STOPPED`/`INTERRUPTED` (Sec 11).
- Added I-11 (no orphan execution state) and I-12 (pause consistency) (Sec 11).
- Added an explicit "rejected transitions must not compensate/overwrite" sentence to the atomic transition requirement (Sec 12).
- Broadened the description of `parent_test_run_id` so it isn't conceptually restricted to interruption recovery — it's a generic run-to-run lineage link (Sec 8).
- Added a note that abandoned/deleted `DRAFT` runs must not be treated as executable Test Runs by the UI or database (Sec 4) — without introducing a new `CANCELLED` state, since nothing in the SRS requires one.

## 1. Purpose

This document is the authoritative behavioral specification for how a **Test Run**, its **Execution Attempts**, and their **Step Results** move between states. It exists to remove ambiguity before coding starts, because the single biggest risk in this project is a state-handling bug that causes a genuine test failure to be reported as PASS, or an infrastructure problem to be silently reported as a test FAIL (SRS guiding principles, Sec 4).

This document does not redefine entities (see Domain Model v0.2) — it defines the **rules of motion** between the states that document already introduced.

## 2. The Three State Levels

```
TEST RUN                       (one execution round)
   └── EXECUTION ATTEMPT       (one attempt of one ATC/dataset within that run)
          └── STEP RESULT      (one logical step within that attempt)
```

A Test Run can contain many Execution Attempts (one per selected ATC × dataset). An Execution Attempt can contain many Step Results. State changes flow **downward** — a Step Result can never unilaterally change its parent Attempt's state, and an Attempt can never unilaterally change its parent Test Run's state, except by reporting information upward that the parent then acts on.

**Serial execution rule:** within a single Test Run, **at most** one Execution Attempt is `EXECUTING` at any moment — Attempts execute one at a time, never in parallel. This is a ceiling, not a guarantee of continuous activity: the Test Run can be `RUNNING` while zero Attempts are `EXECUTING` — for example, during mapping confirmation (Sec 5.2), between one Attempt finishing and the next one's turn arriving, or at a pause boundary. `Test Run.status == RUNNING` must never be read by implementers as implying `Attempt.status == EXECUTING`.

## 3. Test Run States

| State | Meaning |
|---|---|
| `DRAFT` | Being assembled — ATCs, datasets, and configuration are still editable. |
| `READY_FOR_VALIDATION` | Engineer has requested validation of the assembled Test Run. |
| `VALIDATION_FAILED` | Pre-execution validation found a blocking problem (see Sec 7). |
| `READY` | Validation passed. The configuration snapshot has not yet been frozen. |
| `RUNNING` | In its execution phase. Does not guarantee an Attempt is currently `EXECUTING` (Sec 2). |
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
| `RUNNING` | `PAUSED` | Engineer pauses | Pause takes effect only at the next step boundary (Sec 3.4) | — |
| `PAUSED` | `RUNNING` | Engineer resumes | — | — |
| `RUNNING` | `COMPLETED` | Every selected Attempt reaches `PASS`/`FAIL`/`BLOCKED` | No selected Attempt is `NOT_EXECUTED` | `completed_at` set; Outcome computed (Sec 4) |
| `RUNNING` | `STOPPED` | Engineer deliberately stops, OR the run otherwise ends with ≥1 Attempt still `NOT_EXECUTED` | — | Currently-`EXECUTING` Attempt (if any) → Attempt-level `STOPPED`; remaining unstarted Attempts stay `NOT_EXECUTED`; `completed_at` set; Outcome computed |
| `PAUSED` | `STOPPED` | Engineer deliberately stops a paused run | — | Remaining unstarted Attempts stay `NOT_EXECUTED`; `completed_at` set; Outcome computed |
| `RUNNING` | `INTERRUPTED` | Unexpected failure the system cannot recover from | — | In-flight Attempt → Attempt-level `INTERRUPTED` (Sec 5.1); `completed_at` set; Outcome computed |
| `PAUSED` | `INTERRUPTED` | Unexpected failure while paused (e.g., application crash) | — | Same as above |

`COMPLETED`, `STOPPED`, and `INTERRUPTED` are all terminal for the Test Run record itself. Recovery from `INTERRUPTED` always creates a **new** Test Run (Sec 8); it never mutates the interrupted one.

### 3.2 Why "Pause" cannot interrupt a step mid-action

A pause request is accepted immediately but does not take visible effect until the currently executing step reaches its own terminal state (PASS/FAIL/BLOCKED). This avoids a half-completed UI action being left in an undefined state.

### 3.3 READY staleness rule

Validation certifies a *specific* configuration. If any of that changes while the Test Run sits in `READY`, the previously-passed validation no longer certifies what would actually execute. Therefore any such edit forces `READY → DRAFT`, and the engineer must re-run validation before `RUNNING` is reachable again. The stale validation cycle record is kept (not deleted) for audit purposes but marked superseded.

### 3.4 Pause timing detail

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

Both timestamps are recorded so the "pausing" interval is auditable without introducing a separate lifecycle state.

## 4. Test Run Outcome (separate from State)

**State** answers "where is this run in its lifecycle." **Outcome** answers "what did the run find." They are tracked separately because a `COMPLETED` (or `STOPPED`) run can have found a defect, an infrastructure problem, or nothing wrong — collapsing these into one field was the ambiguity in v0.1.

| Outcome | Meaning |
|---|---|
| `PASS` | Every selected Attempt is `PASS`. |
| `FAIL` | At least one selected Attempt is `FAIL`. |
| `BLOCKED` | No Attempt is `FAIL`, but at least one is `BLOCKED`. |
| `PARTIAL` | The run ended (`STOPPED`/`INTERRUPTED`) before every selected Attempt was processed, and neither `FAIL` nor `BLOCKED` applies. |
| `NOT_EVALUATED` | No Attempt ever reached `EXECUTING` — the run never left `DRAFT`/`READY_FOR_VALIDATION`/`VALIDATION_FAILED`/`READY`. |

**Precedence rule** (highest priority first): `FAIL` > `BLOCKED` > `PARTIAL` > `PASS`. A single genuine defect always takes precedence, even if other Attempts were merely `BLOCKED` or unstarted — a real defect must never be hidden behind an infrastructure-issue or partial-run outcome.

**Worked examples:**

```
ATC1=PASS,  ATC2=STOPPED                         → Outcome: PARTIAL
ATC1=FAIL,  ATC2=STOPPED                         → Outcome: FAIL
ATC1=BLOCKED, ATC2=STOPPED                       → Outcome: BLOCKED
ATC1=PASS,  ATC2=PASS,  ATC3=PASS                → Outcome: PASS
ATC1=FAIL,  ATC2=BLOCKED                         → Outcome: FAIL
```

The `BLOCKED`-over-`PARTIAL` case is deliberate: if there is already concrete evidence of a blocking infrastructure problem, that must not be hidden just because the run was later stopped.

Outcome is computed once, when the Test Run reaches a terminal State, and never recalculated afterward (I-06).

**Note on abandoned Drafts:** a Test Run left in `DRAFT` and never progressed is not itself a special state — no `CANCELLED` state is introduced, since nothing in the SRS requires tracking cancellation as distinct from an unfinished draft. The UI and database must simply ensure an abandoned `DRAFT` is never surfaced as if it were an executable or historical run with an Outcome — it has none, because it never left the preparation states.

## 5. Execution Attempt States

| State | Meaning |
|---|---|
| `NOT_EXECUTED` | Selected for this run but never started. |
| `EXECUTING` | Actively running steps. |
| `PASS` | All steps passed. |
| `FAIL` | At least one step genuinely failed the expected result — a real product defect, with no step `BLOCKED`. |
| `BLOCKED` | Valid verification could not be completed due to mapping, communication, configuration, automation, or environment conditions — not a product defect. |
| `STOPPED` | Was `EXECUTING` when the engineer deliberately stopped the parent Test Run. |
| `INTERRUPTED` | Was `EXECUTING` when the parent Test Run was unexpectedly interrupted. |

### 5.1 Execution Attempt Transition Table

| From | To | Trigger | Guard | Side Effects |
|---|---|---|---|---|
| — | `NOT_EXECUTED` | Created when Test Run enters `RUNNING` | — | `execution_attempt` row created for each selected ATC × dataset (or default placeholder, Domain Model Sec 4.20); `attempt_number` assigned (Domain Model Sec 4.21) |
| `NOT_EXECUTED` | `EXECUTING` | This Attempt's turn arrives and its mapping resolves to `STRONG`, `MANUAL_CONFIRMED`, or an engineer-confirmed `CANDIDATE` (Sec 5.2) | — | `started_at` set |
| `NOT_EXECUTED` | `BLOCKED` | This Attempt's turn arrives and its mapping is `UNRESOLVED`, or `CANDIDATE` and the engineer rejects confirmation (Sec 5.2) | Failure category `MAPPING_EXCEPTION` | `completed_at` set; `started_at` is **not** set — this Attempt never entered `EXECUTING` (bounded exception to I-03, see Sec 11) |
| `EXECUTING` | `PASS` | Last step completes `PASS`, no prior step `FAIL`/`BLOCKED` | — | `completed_at` set |
| `EXECUTING` | `FAIL` | A step's actual result genuinely did not match expected | Failure category is `TEST_FAILURE` (Sec 5.3) | Remaining steps → `NOT_EXECUTED`; `completed_at` set |
| `EXECUTING` | `BLOCKED` | A step could not be validly verified for a non-product reason *after execution had already started* (e.g., mapped object disappears mid-run, target disconnects mid-step) | Failure category is `AUTOMATION_FAILURE` / `MAPPING_EXCEPTION` / `COMMUNICATION_FAILURE` / `CONFIGURATION_FAILURE` / `ENVIRONMENT_FAILURE` | Remaining steps → `NOT_EXECUTED`; `completed_at` set |
| `EXECUTING` | `STOPPED` | Engineer deliberately stops the parent Test Run while this Attempt is mid-step | — | In-flight step → `STOPPED`; remaining steps → `NOT_EXECUTED`; `completed_at` set |
| `EXECUTING` | `INTERRUPTED` | Parent Test Run is unexpectedly interrupted while this Attempt is mid-step | — | In-flight step → `INTERRUPTED`; remaining steps → `NOT_EXECUTED`; `completed_at` set |
| `NOT_EXECUTED` | `NOT_EXECUTED` (unchanged, terminal for this run) | Parent Test Run reaches `COMPLETED`/`STOPPED`/`INTERRUPTED` before this Attempt's turn | — | No `started_at` is ever set |

Note the two distinct routes to `BLOCKED`: a **pre-execution** route (`NOT_EXECUTED → BLOCKED`, mapping never resolved) and a **mid-execution** route (`EXECUTING → BLOCKED`, something went wrong after a valid start). Both are `BLOCKED`, not `FAIL`, and both are equally visible in reporting — the distinction only matters for diagnosing *when* the problem was detected, not for the safety classification itself.

### 5.2 Mapping resolution gate (resolves the CANDIDATE ambiguity)

Pre-execution validation (Sec 7) checks mapping *existence*, not final confirmation — a `CANDIDATE` mapping is a non-blocking validation finding because it doesn't prevent the run from starting. But an Attempt only enters `EXECUTING` once its mapping is actually resolved to something usable. `EXECUTING` means the system has genuinely started running steps — it is never used to represent "we are still deciding whether we're allowed to execute."

```
NOT_EXECUTED
     │  (this Attempt's turn arrives)
     ▼
mapping confidence state?
     ├── STRONG             → EXECUTING
     ├── MANUAL_CONFIRMED    → EXECUTING
     ├── CANDIDATE           → engineer confirmation prompt
     │                           ├── Confirm → mapping becomes
     │                           │             MANUAL_CONFIRMED → EXECUTING
     │                           └── Reject  → BLOCKED directly
     │                                          (category: MAPPING_EXCEPTION,
     │                                           never entered EXECUTING)
     └── UNRESOLVED           → BLOCKED directly
                                 (category: MAPPING_EXCEPTION,
                                  never entered EXECUTING)
```

The confirmation prompt is not a formal Attempt state — it is a sub-step of Attempt turn-taking. During it, the Test Run remains `RUNNING` with zero Attempts `EXECUTING` (see Sec 2's note on what `RUNNING` does and does not guarantee).

### 5.3 The FAIL vs. BLOCKED decision — this is the core safety rule

An Attempt is only ever marked `FAIL` when the *product under test* produced a result that genuinely did not match the expected result, and the tooling has reasonable confidence the comparison itself was valid (correct object was checked, adapter/target communication was healthy throughout, mapping was `STRONG`/`MANUAL_CONFIRMED`).

Every other reason a step could not be validly completed must produce `BLOCKED`, never `FAIL`.

**Rule of thumb for implementers:** if you are not certain the comparison was valid, the answer is `BLOCKED`, not `FAIL`. `FAIL` requires positive confidence, not just the absence of a recognized error.

## 6. Step Result States

| State | Meaning |
|---|---|
| `NOT_EXECUTED` | Not yet reached, or the Attempt ended before this step ran. Becomes terminal once the parent Attempt terminates (see I-01). |
| `PASS` | Actual result matched expected result. |
| `FAIL` | Actual result did not match expected result (genuine product behavior). |
| `BLOCKED` | Could not be validly verified (Sec 5.3 categories). |
| `STOPPED` | Was actively executing when the engineer deliberately stopped the run. |
| `INTERRUPTED` | Was actively executing when the run was unexpectedly interrupted. |

The first step in an Attempt to reach a terminal, non-`PASS` state determines that Attempt's terminal state; every step after it is set to `NOT_EXECUTED`.

## 7. Pre-Execution Validation (READY_FOR_VALIDATION → READY / VALIDATION_FAILED)

Validation is **repeatable** — a Test Run may cycle through `READY_FOR_VALIDATION ↔ VALIDATION_FAILED` multiple times while being prepared, and may return to `DRAFT` per Sec 3.3 and require another cycle. Each validation attempt is recorded, not just the latest one:

- `validation_id`
- `test_run_id`
- `validated_at`
- `validator_version` (which ruleset/build performed the check)
- `result` (`PASSED` / `FAILED`)
- `findings` (structured list of blocking/non-blocking items)

**Blocking findings (→ `VALIDATION_FAILED`):**
- Selected ATC references a mapping that is `UNRESOLVED` with no fallback.
- Selected Test Configuration references an application/module version that no longer exists.
- Required target/adapter is unreachable at validation time.
- TCS/ATC revision referenced by the selection has since been retired without a replacement.

**Non-blocking findings (→ `READY`, surfaced as warnings):**
- A mapping is `CANDIDATE` (resolved at execution time per Sec 5.2, not at validation time).
- A module version is present but has no recorded compatibility notes.

The exhaustive rule list is finalized during detailed design; this section defines the category split and the record-keeping requirement, not the full rule set.

## 8. Interruption Recovery — What Happens Next

An `INTERRUPTED` Test Run is a terminal record — it never transitions again, and it is never edited (I-06). There is no `INTERRUPTED → READY` transition. On the next launch (or when the engineer next opens the project), the tool must:

1. Detect the `INTERRUPTED` Test Run and surface it explicitly.
2. Present the engineer with an explicit choice, each of which creates a **new** Test Run in `DRAFT`, linked back to the original via `parent_test_run_id`:
   - **Start a new Test Run** using the same selection.
   - **Re-run only the Attempts that did not reach a terminal `PASS`/`FAIL`/`BLOCKED`** as a new Test Run.
   - **Close out the interrupted run as-is** — no new Test Run is created; the `INTERRUPTED` record simply stands, with Outcome `PARTIAL`.
3. The original `INTERRUPTED` Test Run and its Attempts/Step Results are never edited or deleted (I-06) — only new records are created.

`parent_test_run_id` is intentionally **not** conceptually restricted to interruption recovery — it's a generic run-to-run lineage link. The same field can later represent a deliberate re-execution of a `COMPLETED` run, or multiple recovery attempts branching from one interrupted run (`Original Run → Recovery Run 1`, `Original Run → Recovery Run 2`, etc.).

> **Domain Model follow-up required:** `parent_test_run_id` does not yet exist as a field on the `Test Run` entity (Domain Model v0.2, Sec 4.16). This must be added before the SQL DDL is written.

## 9. Timeout Handling

A step that does not respond within its timeout threshold is **not** automatically marked `FAIL`. `FAIL` is permitted only when the verification operation itself was successfully performed — the object was found, the comparison was actually made — and the observed product behavior can be validly compared against the expected result. Communication being nominally "healthy" is necessary but not sufficient on its own; a connected-but-malfunctioning target that never actually produces a comparable result must still resolve to `BLOCKED`, not `FAIL`.

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
FAIL only if  (category depends on cause —
the mismatch  COMMUNICATION_FAILURE if the
is genuine    channel dropped, ENVIRONMENT_FAILURE
              /AUTOMATION_FAILURE otherwise)
```

Exact timeout threshold values are deferred to detailed design (Sec 13 #3), but the branching rule above — never default a timeout straight to `FAIL`, and never treat "connected" as equivalent to "verifiable" — is not deferred, since it follows directly from Sec 5.3.

## 10. Worked Example

```
Test Run: Round-07
  ATC-001 / Dataset-01  → EXECUTING → step1 PASS → step2 PASS → step3 PASS → PASS
  ATC-004 / Dataset-01  → EXECUTING → step1 PASS → step2 FAIL (expected "Ready",
                           actual "Error") → Attempt = FAIL
  ATC-009 / Dataset-01  → EXECUTING → step1 PASS → step2: target disconnected
                           mid-step → step2 = BLOCKED (COMMUNICATION_FAILURE) →
                           Attempt = BLOCKED, step3 = NOT_EXECUTED
  ATC-012 / Dataset-01  → still NOT_EXECUTED when engineer clicks Stop
                           → Test Run = STOPPED

Test Run Round-07 final:
  State:   STOPPED   (ATC-012 never started)
  Outcome: FAIL      (FAIL takes precedence over BLOCKED/PARTIAL, Sec 4)

  ATC-001 → PASS
  ATC-004 → FAIL          (real defect — reported to dev team)
  ATC-009 → BLOCKED       (infra issue — target reconnect needed, not a defect)
  ATC-012 → NOT_EXECUTED  (never ran)
```

## 11. State Machine Invariants

- **I-01 — Terminal immutability.** Once a Step Result, Execution Attempt, or Test Run reaches its terminal state for the current execution context, it cannot transition again. For Step Results, `NOT_EXECUTED` becomes terminal once its parent Attempt itself reaches a terminal state.
- **I-02 — No execution after terminal state.** An Attempt in a terminal state can never execute another Step Result.
- **I-03 — No result without execution, with one bounded exception.** An Attempt cannot become `PASS`/`FAIL` without having been `EXECUTING`. The sole exception is the pre-execution `BLOCKED` route (Sec 5.1/5.2), where a mapping never resolves and the Attempt is `BLOCKED` without ever entering `EXECUTING` — this is the only case where a terminal Attempt state is reached with `started_at` unset.
- **I-04 — FAIL requires valid verification.** `FAIL` requires failure category `TEST_FAILURE` and a successfully performed, validly comparable verification operation (Sec 9).
- **I-05 — BLOCKED excludes product defect.** `BLOCKED` must never be used for a verified product mismatch.
- **I-06 — Historical immutability.** Completed/terminated Test Run, Attempt, and Step Result records are never modified after reaching a terminal state; corrections happen via new revisions/new runs (Domain Model Sec 7).
- **I-07 — At most one active Attempt per Test Run.** Attempts within one Test Run never execute concurrently (Sec 2); the Test Run may be `RUNNING` with zero Attempts `EXECUTING`.
- **I-08 — One active Test Run per Test PC.** Per SRS NFR-013.
- **I-09 — Snapshot immutability.** After `READY → RUNNING`, the execution snapshot cannot change for the life of that Test Run.
- **I-10 — Parent-child consistency (full aggregation algorithm).** An Attempt's terminal state is determined by its steps as follows, in priority order:
  - `INTERRUPTED` if any step is `INTERRUPTED`.
  - else `STOPPED` if any step is `STOPPED`.
  - else `FAIL` if any step is `FAIL`.
  - else `BLOCKED` if any step is `BLOCKED`.
  - else `PASS` if every executed step is `PASS`.
- **I-11 — No orphan execution state.** An `EXECUTING` Attempt must belong to a `RUNNING` Test Run. An `EXECUTING` (in-progress) Step Result must belong to an `EXECUTING` Attempt.
- **I-12 — Pause consistency.** A `PAUSED` Test Run must have zero Attempts in `EXECUTING`.

## 12. Atomic Transition Requirement

Every state transition at every level (Test Run, Attempt, Step Result) must be implemented as an atomic, current-state-checked write — conceptually a compare-and-swap: "update to new state only if the record is still in the expected prior state." If the check fails (state already changed by another path — e.g., an engineer's Stop click racing against a step's own completion), the write is rejected and the caller re-reads the current state rather than blindly overwriting it.

**A rejected transition must never attempt a compensating overwrite of the newly observed state.** For example, if an execution thread's `EXECUTING → PASS` write loses the race against a UI thread's `EXECUTING → STOPPED` write, the execution thread must accept the `STOPPED` outcome as final — it must not retry, force, or "correct" the record back toward `PASS`. Whichever transition wins the compare-and-swap is authoritative.

This is required even in the single-user, single-PC V1 scope, because the UI thread and the execution/adapter thread are still two independent writers to the same record. Exact implementation (DB-level optimistic locking, in-process mutex, or both) is a detailed-design decision; the atomicity requirement itself is not.

## 13. Open Design Decisions

Deferred to detailed design / implementation:

1. Exact set of blocking vs. non-blocking pre-execution validation rules (Sec 7 gives categories, not the full list).
2. **Domain Model addition required:** `parent_test_run_id` on the `Test Run` entity (Sec 8).
3. Timeout threshold values (Sec 9 gives the branching rule, not the numbers).
4. Whether `PAUSED` has a maximum duration before the tool prompts the engineer.
5. Exact mechanism for the atomic transition requirement (Sec 12) — DB-level, in-process, or both.

## 14. Acceptance Questions for Review

1. Does every state have a defined trigger to enter it and at least one defined trigger to leave it (except terminal states)?
2. Is it always possible to tell, from stored data alone, whether an Attempt never started, was deliberately stopped, or was unexpectedly interrupted?
3. Does the FAIL vs. BLOCKED rule leave any ambiguous case where an implementer would have to guess?
4. Can a `PAUSED` run ever leave a step in a half-completed, unrecorded state?
5. Does the recovery flow ever allow an `INTERRUPTED` run to be silently treated as complete, or to be mutated after the fact?
6. Does the State vs. Outcome separation give reports enough information to distinguish "found a defect" from "couldn't finish testing"?
7. Is the `CANDIDATE` mapping behavior consistent between validation-time and execution-time handling, and does it correctly avoid ever marking an unresolved mapping as `EXECUTING`?
8. Does I-10's aggregation algorithm produce an unambiguous answer for every possible combination of step outcomes within an Attempt?

---

**Status:** Test Run State Machine Draft v0.3 — candidate baseline for Domain Model v0.3 and SQL DDL.
**Next artifact after review:** Domain Model v0.3 (add `parent_test_run_id`, `validation` history table, Test Run `outcome` field), then Detailed SQL schema (DDL), then Adapter interface contract.

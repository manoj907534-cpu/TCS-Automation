# TCS Automation — Adapter Interface Contract

**Version:** 0.1 — Design Draft
**Status:** Approved Baseline
**Baseline:** Architecture v0.2 / Domain Model & Database Schema v0.3 / Test Run State Machine v0.4 / SQL Schema (DDL) v0.1

## 1. Purpose and Scope

This document defines the interface that **every** adapter (Windows, Embedded Linux/Qt, and any future target type) must implement, so the execution engine can drive any of them identically without knowing which one it's talking to. It is the seam between two very differently-shaped worlds — a desktop application's UI tree and an embedded board's hardware/firmware — and the core engine's job is to never need to know the difference.

This document does not redefine states or entities (see State Machine v0.4, Domain Model v0.3) — it defines **what the execution engine calls, what the adapter must hand back, and — critically — where the responsibility boundary sits between "the adapter reports facts" and "the core engine decides outcomes."**

## 2. The Central Design Rule: Adapters Report Facts, the Core Engine Decides Outcomes

This is the single most important rule in this document, and every interface method below is shaped by it.

**An adapter never decides `PASS`, `FAIL`, or `BLOCKED`.** Those are State Machine concepts (Sec 5.1, 5.3) owned entirely by the core execution engine. An adapter's job is narrower and more mechanical: attempt the requested action, observe what actually happened, and report a structured, honest account of what occurred — including when it doesn't know, when it couldn't tell, or when something outside the product under test went wrong.

This split exists because of the project's core safety principle (State Machine Sec 5.3): a genuine `FAIL` requires positive confidence that the comparison itself was valid. An adapter is the only component with first-hand knowledge of *how* an action actually happened — whether the object was found, whether the comparison completed, whether communication held — so it must report that raw information faithfully. But the *policy decision* of "given these facts, is this a FAIL or a BLOCKED" belongs to the core engine, applying the rule in State Machine Sec 5.3/Sec 9 consistently across both adapter types. If adapters were allowed to self-report `FAIL`/`BLOCKED` directly, two different adapter implementations could silently apply two different safety standards — which is exactly the risk this whole document exists to prevent.

## 3. Adapter Types in Scope

| `adapter_type` (SQL Schema Sec 9) | Target | Object model |
|---|---|---|
| `WINDOWS` | Windows desktop application under test | UI controls (buttons, fields, labels, etc.) |
| `EMBEDDED_LINUX_QT` | Embedded Linux/Qt board (e.g., Card-1..4, Controller) | QML/Qt widgets, plus hardware module state where applicable |

Both implement the identical interface defined below. Adapter-specific behavior lives entirely inside the adapter implementation and in the `target_locator`/`identification_method` payloads (Domain Model Sec 4.13) — never in a different method signature per adapter type. If a future adapter type is added (e.g., a web adapter), it implements this same interface; nothing here is Windows- or Qt-specific by design.

## 4. Adapter Lifecycle

Implements SRS FR-060–065 (Already Running / Manual / Automatic preparation modes) and Domain Model Sec 4.14 (Test Configuration's `adapter_type`).

```
Connect(target_connection_info)
        │
        ▼
GetCapabilities()  ──────────► used at pre-execution validation (State Machine Sec 7)
        │
        ▼
PrepareApplication(mode, application_info)   ── mode-dependent, see below
        │
        ▼
[ execution phase — IdentifyObject / ExecuteStep / CaptureEvidence, repeated ]
        │
        ▼
Disconnect()
```

### 4.1 `Connect(connectionInfo) → ConnectResult`

Establishes the adapter's link to the target (a Windows process handle, or a network/serial connection to an embedded board). Must be idempotent — calling `Connect` on an already-connected adapter returns the existing connection's status rather than erroring.

```
ConnectResult {
    connected: boolean
    adapter_version: string        // → Test Run Configuration Snapshot.adapter_version
    failure: FailureInfo | null    // populated iff connected = false
}
```

### 4.2 `GetCapabilities() → CapabilitySet`

Declares what this adapter instance can actually do right now — used by pre-execution validation (State Machine Sec 7: "required target/adapter is unreachable" is one blocking finding this call helps answer) and by the Windows/Embedded action-vocabulary tables (SRS Sec 9/10).

```
CapabilitySet {
    supported_action_types: [ActionType]      // e.g. CLICK, VERIFY_TEXT, SET_VALUE, WAIT, ...
    supports_screenshot_evidence: boolean
    supports_health_check: boolean
    adapter_type: "WINDOWS" | "EMBEDDED_LINUX_QT"
}
```

### 4.3 `PrepareApplication(mode, applicationInfo) → PrepareResult`

`mode` is one of `ALREADY_RUNNING`, `MANUAL`, `AUTOMATIC` (SRS FR-060–065). Behavior per mode:

- **`ALREADY_RUNNING`** — adapter verifies the target application/process is already up and reachable; never launches or modifies it. This is the only mode valid for `EMBEDDED_LINUX_QT` in V1 (State Machine's embedded-target constraint, carried from SRS Sec 5.2/10).
- **`MANUAL`** — adapter waits for the engineer to confirm the application is ready (a UI-level prompt orchestrated by the core engine, not the adapter itself); the adapter's role is limited to verifying reachability once signaled.
- **`AUTOMATIC`** — Windows-only in V1: adapter is permitted to launch the target application itself, given `applicationInfo` (executable path, arguments).

```
PrepareResult {
    ready: boolean
    failure: FailureInfo | null    // populated iff ready = false
}
```

### 4.4 `Disconnect() → void`

Releases the adapter's connection. Must be safe to call even if the connection was never fully established (no-op, not an error).

## 5. Object Identification

Implements the mapping-resolution gate (State Machine Sec 5.2) and Domain Model Sec 4.12–4.13. This is the step where a `target_locator` (adapter-specific JSON, Domain Model Sec 4.13) is turned into a live, addressable object on the actual target.

### 5.1 `IdentifyObject(targetLocator, identificationMethod) → IdentificationResult`

```
IdentificationResult {
    outcome: "FOUND" | "CANDIDATE" | "NOT_FOUND"
    matched_object_ref: ObjectRef | null   // opaque handle, only if FOUND or CANDIDATE
    candidate_details: string | null       // human-readable, only if CANDIDATE — surfaced
                                            // to the engineer during the confirmation prompt
                                            // (State Machine Sec 5.2)
    failure: FailureInfo | null            // populated iff NOT_FOUND
}
```

**Critical boundary:** the adapter reports `FOUND` / `CANDIDATE` / `NOT_FOUND` as a factual identification outcome — it does **not** independently decide `STRONG` vs. `CANDIDATE` vs. `UNRESOLVED` confidence in the `Mapping Revision` sense (Domain Model Sec 4.13's `confidence_state`). That confidence classification belongs to the **mapping subsystem** (a separate concern from the adapter, responsible for the object-matching algorithm and its confidence scoring — Architecture Sec 9's capability model, deferred to detailed design per SRS Open Item #1). The adapter's `IdentifyObject` call is invoked *by* that mapping subsystem, one layer below the confidence decision, not a replacement for it. `FOUND` here means "the adapter successfully resolved the locator to exactly one object"; the mapping subsystem is what turns repeated `IdentifyObject` results, over time, into a `STRONG`/`CANDIDATE`/`UNRESOLVED` `Mapping Revision`.

`outcome = "NOT_FOUND"` at execution time (as opposed to at validation time) is the mid-execution route to `BLOCKED` with category `MAPPING_EXCEPTION` (State Machine Sec 5.1's `EXECUTING → BLOCKED` row — e.g., an object that was previously mapped has since disappeared from the UI/board state).

## 6. Step Execution

This is where the FAIL/BLOCKED safety boundary (Sec 2) is most concrete. Implements State Machine Sec 5.3, 6, 9 and Domain Model Sec 4.22 (`Step Result`).

### 6.1 `ExecuteStep(actionType, objectRef, parameters, expectedResult, timeoutMs) → StepExecutionResult`

```
StepExecutionResult {
    outcome: StepOutcome            // see enum below — this is the core of the contract
    actual_result: string | null    // populated for outcomes where a comparison was made
    verification_performed: boolean // true iff the adapter actually completed the
                                     // expected-vs-actual comparison (State Machine Sec 9)
    failure: FailureInfo | null     // populated for any non-MATCH/MISMATCH outcome
    duration_ms: integer
}

enum StepOutcome {
    MATCH,                  // verification performed, actual == expected
    MISMATCH,                // verification performed, actual != expected — genuine
                              // product behavior
    OBJECT_UNAVAILABLE,       // object could not be interacted with (disappeared,
                              // became inaccessible mid-step)
    COMMUNICATION_LOST,       // adapter/target link dropped during the action
    ADAPTER_ERROR,            // the adapter itself faulted (unexpected exception,
                              // automation library error) — not a target/product issue
    ENVIRONMENT_ERROR,        // target/environment condition prevented the action
                              // (e.g., board in an unexpected power state)
    TIMED_OUT_UNVERIFIED      // timeout elapsed without the comparison ever completing
                              // (State Machine Sec 9's "no" branch)
}
```

### 6.2 How `StepOutcome` maps to State Machine categories — the engine's job, but the mapping must be exact

The core engine (not the adapter) performs this mapping when writing the `Step Result` and `Failure` rows (SQL Schema Sec 11):

| `StepOutcome` | `step_results.result_state` | `failures.category` |
|---|---|---|
| `MATCH` | `PASS` | *(no Failure row)* |
| `MISMATCH` **and** `verification_performed = true` | `FAIL` | `TEST_FAILURE` |
| `OBJECT_UNAVAILABLE` | `BLOCKED` | `MAPPING_EXCEPTION` |
| `COMMUNICATION_LOST` | `BLOCKED` | `COMMUNICATION_FAILURE` |
| `ADAPTER_ERROR` | `BLOCKED` | `AUTOMATION_FAILURE` |
| `ENVIRONMENT_ERROR` | `BLOCKED` | `ENVIRONMENT_FAILURE` |
| `TIMED_OUT_UNVERIFIED` | `BLOCKED` | `COMMUNICATION_FAILURE` (if the drop was link-level) or `ENVIRONMENT_FAILURE`/`AUTOMATION_FAILURE` (otherwise) — the adapter's `failure.category_hint` (Sec 7) disambiguates this one case |

**This table is the literal implementation of State Machine Sec 5.3's "if you are not certain the comparison was valid, the answer is BLOCKED, not FAIL."** Note there is no `StepOutcome` value that lets an adapter directly assert `FAIL` — the only path to `FAIL` is `MISMATCH` with `verification_performed = true`, meaning the adapter is structurally prevented from ever reporting a failure it isn't certain was a genuine, validly-compared product defect. This is a deliberate interface design choice, not an oversight: it makes the safety rule impossible to bypass by construction, rather than relying on every adapter implementation remembering to apply it correctly.

`verification_performed = false` paired with `MISMATCH` should not occur — if it does, the core engine must treat it as `ADAPTER_ERROR`/`AUTOMATION_FAILURE` (a contract violation by the adapter) rather than trust the `MISMATCH` at face value. This is a defensive rule for the execution engine's implementation, not a state an adapter should ever intentionally produce.

### 6.3 Timeout handling contract

Implements State Machine Sec 9 directly. `timeoutMs` is supplied by the core engine (exact values are a detailed-design/configuration decision — State Machine Sec 13 #3, still open). The adapter's obligation:

- If the verification operation completes (object found, comparison made) before the timeout, return `MATCH`/`MISMATCH` with `verification_performed = true`, regardless of how close to the timeout it was.
- If the timeout elapses with the verification operation never having completed, return `TIMED_OUT_UNVERIFIED` with `verification_performed = false`. **The adapter must never convert a timeout into a synthetic `MISMATCH`** — doing so would silently violate State Machine Sec 9's core rule by manufacturing false confidence.

## 7. Failure Reporting Contract

Implements Domain Model Sec 4.23 (`Failure`). Every non-`MATCH`/`MISMATCH` `StepExecutionResult`, and every `false`-`ready`/`false`-`connected` lifecycle result, carries a `FailureInfo`:

```
FailureInfo {
    category_hint: FailureCategory   // adapter's best classification — see note below
    code: string | null              // adapter/vendor-specific error code, if any
    message: string                  // human-readable, becomes failures.message
    technical_details: string | null // stack trace / raw adapter log excerpt, optional
}
```

**`category_hint` is a hint, not a directive.** For most `StepOutcome` values (Sec 6.2's table), the category is fully determined by the outcome itself, and the adapter's `category_hint` should simply agree. The one genuinely ambiguous case is `TIMED_OUT_UNVERIFIED`, where the adapter is in the best position to know *why* the timeout happened (a dropped connection vs. a genuinely slow-but-connected target) — but the core engine retains final authority to override the hint if it contradicts other evidence (e.g., a health check run immediately after shows the connection is actually fine, suggesting `AUTOMATION_FAILURE` over `COMMUNICATION_FAILURE`). The adapter is never the sole or final authority on categorization — consistent with Sec 2's central rule.

## 8. Evidence Capture

Implements Domain Model Sec 4.24 (`Evidence`). Video is out of scope (SRS Sec 5.2); V1 supports screenshots/attachments only.

### 8.1 `CaptureEvidence(evidenceType) → EvidenceCaptureResult`

```
EvidenceCaptureResult {
    captured: boolean
    file_bytes: binary | null   // raw image/file data; core engine handles storage
                                 // path assignment and hashing (SQL Schema Sec 11:
                                 // evidence.file_path, evidence.file_hash)
    failure: FailureInfo | null // populated iff captured = false
}
```

Evidence capture failing does **not** by itself change the outcome of the step it's attached to — a screenshot failure is logged but does not retroactively turn a `MATCH` into anything else. This is a deliberate scope boundary: evidence is supporting material for a human reviewer, not an input to the FAIL/BLOCKED decision.

## 9. Health Check and Cancellation

### 9.1 `CheckHealth() → HealthStatus`

```
HealthStatus {
    healthy: boolean
    detail: string | null
}
```

Available only if `CapabilitySet.supports_health_check = true` (Sec 4.2). Used by the core engine both proactively (before starting an Attempt) and reactively (to help disambiguate a `TIMED_OUT_UNVERIFIED` category hint, Sec 7).

### 9.2 `RequestCancel() → void`

Implements the Pause/Stop boundary (State Machine Sec 3.2, 3.4): a pause or stop request is only ever allowed to take effect **at a step boundary**, never mid-action. `RequestCancel` is therefore a best-effort, asynchronous signal — it tells the adapter "finish what you're doing and don't start anything new," not "abort immediately." The core engine must not assume a step was interrupted just because `RequestCancel` was called; it still waits for `ExecuteStep`'s normal return (with whatever `StepOutcome` that in-flight action naturally produces) before treating the Attempt as `STOPPED`/`INTERRUPTED` (State Machine Sec 5.1). This keeps step atomicity intact — the adapter, not the core engine, is the only component that knows when an in-flight action has actually reached a safe stopping point.

## 10. Versioning and Compatibility

- `ConnectResult.adapter_version` (Sec 4.1) is captured into `Test Run Configuration Snapshot.adapter_version` (SQL Schema Sec 9) at run-freeze time, so every historical run records exactly which adapter build produced its results.
- This interface contract itself is versioned independently of any individual adapter's internal version — a breaking change to this contract (adding/removing/changing a method signature) requires every existing adapter implementation to be updated together, since the core engine has no per-adapter-type branching logic by design (Sec 3).
- `CapabilitySet.supported_action_types` (Sec 4.2) is the mechanism for two adapters to legitimately differ in what they support without requiring interface changes — a new Qt action type does not require a new method, only an addition to that adapter's declared capability list and the shared `ActionType` enum.

## 11. Open Design Decisions

Deferred to detailed design/implementation — consistent with what earlier documents already flagged as open:

1. Exact `ActionType` enum values (SRS Sec 9/10 give the vocabulary categories; the literal enum needs finalizing against both adapters' real capabilities).
2. Exact `timeoutMs` default values and whether they're global, per-action-type, or per-ATC-configurable (State Machine Sec 13 #3, still open).
3. Whether `IdentifyObject` and `ExecuteStep` are separate calls (as modeled here) or whether some action types collapse them into one adapter call for efficiency — a pure implementation optimization that doesn't change the contract's safety properties either way.
4. The mapping subsystem's confidence-scoring algorithm that consumes `IdentificationResult` (Sec 5.1) — explicitly out of scope for this document, which only defines the adapter-facing boundary below that subsystem.
5. Serialization format for `ObjectRef` (opaque handle) and `target_locator` — JSON structure remains a deferred decision (Domain Model Sec 17 #4), consistent with earlier documents.
6. Concrete transport for `Connect`/embedded communication (SRS mentions Ethernet for embedded targets) — protocol-level detail, doesn't affect this interface.

## 12. Acceptance Questions for Review

1. Is there any `StepOutcome` value, or any combination of fields, that would let an adapter directly assert `FAIL` without the core engine independently verifying `verification_performed = true`? (There should not be — Sec 6.2.)
2. Does the contract give the adapter any way to bypass the mapping-resolution gate (State Machine Sec 5.2) — i.e., can an adapter cause an Attempt to enter `EXECUTING` without the core engine's confidence check happening first?
3. Is `RequestCancel`'s best-effort, step-boundary-respecting semantics (Sec 9.2) sufficient to implement Pause/Stop (State Machine Sec 3.2, 3.4) without ever leaving a step in a half-completed state?
4. Does every `Failure` category referenced in Domain Model Sec 4.23 / SQL Schema Sec 11 have a corresponding, unambiguous path to it from this contract's `StepOutcome`/`FailureInfo` types?
5. Is the boundary between "adapter identifies an object" (Sec 5) and "mapping subsystem assigns confidence" (Domain Model Sec 4.13) clear enough that a future adapter implementer wouldn't accidentally conflate the two?

---

**Status:** Adapter Interface Contract Draft v0.1 — ready for review.
**Next step after review:** this is the last design artifact in the sequence (SRS → Architecture → Domain Model → State Machine → SQL DDL → Adapter Contract). Implementation (execution engine, Windows adapter, Embedded Linux/Qt adapter) can begin against this baseline.

# Zoom-Out Architectural Review (`docs/designs/browser-execution-subsystem.md`)

## Executive Summary

The design addresses real limitations when embedding `jev-ultrafast` as a multi-step tactical sub-goal runner (tab lifecycle persistence, avoiding secondary LLM calls on known `TYPE_TEXT` inputs via an orthogonal `type_text_input` head, masking password fields with lowest-layer `$secret:<id>` dereferencing, and fixing nested scroll/Shadow DOM blind spots). However, the proposal over-engineers several boundaries, introduces unimplemented/speculative abstractions, drops into low-level JavaScript/DOM weeds inside the high-level specification (`1..379`), and fragments a deliberately tiny ~700-line codebase into too many modules and parallel abstractions.

---

## Findings

### 1. Architectural Mismatch & YAGNI: Unimplemented Daemon-Side Network/Navigation Interception
* **Severity**: Blocker
* **Affected Scope**: `Overview` (lines 16–17), `Technical Plan §1, §2, §4` (lines 106–111, 156–157, 199–210), vs. `Detailed Implementation Plan` (lines 382–531)
* **Simplicity Rationale**:
  - The high-level architecture claims Tier 3 modifies the `browser-harness` daemon to disable `TAB_MARKER_JS`, listen to `Page.frameRequestedNavigation` and `Target.targetCreated` events, and enforce a "Layer 5 Secondary Network Guard (`Network` / `Fetch`)" blocking subresource requests, file downloads/uploads, and third-party payment/webhook hosts.
  - However, `browser-harness` is an external package dependency (`browser_harness`), not part of this repository, and the `Detailed Implementation Plan` (lines 382–531) touches zero files in `browser-harness`—implementing child-tab discovery via `poll_child_tabs` in [`jev_ultrafast/session.py`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L406), title normalization in [`jev_ultrafast/snapshot.js`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L423), and domain checks in [`jev_ultrafast/browser.py`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L439).
  - Furthermore, as the document itself notes in `Alternative 1` (line 327), intercepting `Fetch` / `Network` events over `browser-harness`'s synchronous IPC ring buffer risks deadlocks and event drops.
  - **Recommendation**: Drop "Layer 5 Secondary Network Guard" and claims of modifying `browser-harness` daemon internals from the architecture. Scope Tier 3 strictly to `jev_ultrafast`'s CDP session/tab management (`browser.py` + `snapshot.js` + `session.py`) using pre-/post-action URL/domain guards and target polling.

### 2. Speculative YAGNI Fields and Unproducible Reason Codes in `SubgoalRequest` / `SubgoalResult`
* **Severity**: Major
* **Affected Scope**: `Technical Plan §6` (lines 241–299) and `Detailed Implementation Plan §1, §8` (lines 384–393, 479)
* **Simplicity Rationale**:
  - `SubgoalRequest` defines `expected_result: dict[str, Any]` (line 251), and `SubgoalResult` defines 20 fields including `extracted_data: dict[str, Any]` (line 292) and `claimed_evidence: dict[str, Any]` (line 296). Neither the TypeSafe policy in [`jev_ultrafast/model.py`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/jev_ultrafast/model.py) nor `SubgoalRunner` in [`jev_ultrafast/agent.py`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/jev_ultrafast/agent.py) has any mechanism to extract arbitrary structured data or consume `expected_result`.
  - Similarly, `ReasonCode` (lines 262–280, 390) defines 17 fine-grained literals, including codes like `AUTH_WALL_DETECTED` and `MISSING_INPUT_VALUE`, even though the model only chooses `BLOCKED` or `NEEDS_APPROVAL` without a sub-reason head. Guessing why the model chose `BLOCKED` requires brittle heuristics.
  - **Recommendation**: Remove dead/speculative fields (`expected_result`, `extracted_data`, `claimed_evidence`) and trim `ReasonCode` to deterministic runtime causes (`GOAL_SATISFIED`, `MODEL_BLOCKED`, `MODEL_ESCALATED_APPROVAL`, `APPROVAL_GATE_TRIGGERED`, `DOMAIN_BOUNDARY_BLOCKED`, `NO_PROGRESS_3X`, `STEP_BUDGET_EXCEEDED`, `TIME_BUDGET_EXCEEDED`, `UNCERTAIN_MUTATION_STATE`, `CDP_PROTOCOL_ERROR`, `STALE_PAGE_EXHAUSTED`), putting fine-grained gate categories (`credentials`, `destructive`, `payment`, `submit_in_read_only`) on `pending_approval` instead.

### 3. Duplicating the Execution Loop Between `SubgoalRunner` and `Agent`
* **Severity**: Major
* **Affected Scope**: `Technical Plan §1` (line 100) and `Detailed Implementation Plan §8` (lines 465–481)
* **Simplicity Rationale**:
  - [`jev_ultrafast/agent.py`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/jev_ultrafast/agent.py) is currently 175 lines and implements a clean single-step state machine (`command("predict")`, `command("act")`, `command("tick")`, `run()`).
  - Section 8 proposes adding a separate `SubgoalRunner` class with its own loop alongside `Agent`, while also extending `Agent.__init__` to accept `inputs`, `constraints`, `session_id`, and `tab_id` (line 480). Maintaining two parallel loops in `agent.py` risks divergence in stale-page handling, `pending_text` caching, pre-act gating, and history logging.
  - **Recommendation**: Extend [`Agent`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/jev_ultrafast/agent.py#L12-L175) directly to support leased tabs, `inputs`, `constraints`, and `needs_approval` state transitions within `command("predict" | "act" | "tick")`, and make `run_subgoal(request)` a thin wrapper around `Agent` rather than introducing a second runner class.

### 4. `custom_js_predicate` in `verify_tab` Violates Non-Goals and Read-Only Guarantees
* **Severity**: Major
* **Affected Scope**: `Technical Plan §6` (lines 303–316) and `Detailed Implementation Plan §9` (lines 482–490)
* **Simplicity Rationale**:
  - [`AGENTS.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/AGENTS.md) and `Non-Goals` (line 45) strictly forbid letting an agent emit selectors or executable JavaScript, and `Technical Plan §2` (line 153) states that in `verifying` state all mutating `Runtime.evaluate` calls are blocked.
  - Yet `verify_tab` (line 312, 487) exposes `custom_js_predicate: str | None = None` to Tier 1 callers. Executing caller-supplied JavaScript strings inside `verifying` state both creates an arbitrary code-execution surface for the calling agent and makes it impossible to guarantee that `verify_tab` is read-only (since arbitrary JS can mutate the DOM or submit forms).
  - Existing verification in [`examples/flights.py`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/examples/flights.py#L18-L39) already demonstrates declarative checks over observed URL, text, and control values/labels.
  - **Recommendation**: Remove `custom_js_predicate` from `verify_tab`. Keep verification purely declarative (`url_matches`, `required_text`, `forbidden_text`, `control_values`) evaluated by code-owned read-only inspection logic.

### 5. Altitude Drops (Weeds) in the Feature Specification (`1..379`)
* **Severity**: Major
* **Affected Scope**: `Technical Plan §3` (lines 190–194) and `Technical Plan §5` (lines 213–220)
* **Simplicity Rationale**:
  - Lines 191–193 embed literal JavaScript snippets (`const safe = e => !['file','hidden'].includes(e.type);`, `const isSensitive = e => e.type === 'password' || ...`) in the high-level architecture section, and lines 214–220 prescribe low-level DOM API calls (`document.querySelectorAll(selector)`, `document.createTreeWalker(document.body, ...)`, `scrollHeight > clientHeight + 20`, `action.get("x", 550)`).
  - These implementation weeds belong exclusively in the `Detailed Implementation Plan` (`380..549`), where they are already repeated verbatim at lines 424–441.
  - **Recommendation**: Elevate `Technical Plan §3` and `§5` (`1..379`) to describe architectural contracts and invariants (what metadata is exposed, how scroll targets and open Shadow roots are represented, and where secrets are resolved), leaving concrete JS expressions and function-level edits to `Detailed Implementation Plan` (`380..549`).

### 6. Over-Fragmentation of a Small Codebase (Module Proliferation)
* **Severity**: Nit
* **Affected Scope**: `Detailed Implementation Plan §1, §2, §3, §9` (lines 384–417, 482–490)
* **Simplicity Rationale**:
  - [`README.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/README.md#L107-L117) emphasizes *"Small enough to read"* as a core property of `jev_ultrafast` (currently 5 core files). Adding four new modules (`contracts.py`, `session.py`, `policy.py`, `verifier.py`) nearly doubles the file count for logic that is tightly coupled (e.g., `verify_tab` is a single function; `contracts.py` is pure dataclasses used by `session.py`/`policy.py`/`agent.py`).
  - **Recommendation**: Consider consolidating `contracts.py` and `verifier.py` into `session.py` or `policy.py` (or keeping new modules to at most two: `session.py` for tab lifecycle/contracts and `policy.py` for safety/verification) to preserve the repository's small-footprint design.

# Stage 9 Implementation Readiness Report (`implementation_readiness_report.md`)

**Target Document**: [`docs/designs/browser-execution-subsystem.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L380-L549) (`DetailedRange: 380..549`, with `1..379` as reference context)
**Reviewers**: `egm-implementation-readiness`, `egm-codebase-details`, `egm-style`

---

## 🔴 Blockers

### 1. Password Inputs Still Dropped by `role(e)` in `snapshot.js` Despite Changing `safe(e)` (`egm-implementation-readiness`, `egm-codebase-details`)
* **Affected Scope**: [Line 424](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L424) and [line 517](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L517) (referencing [`jev_ultrafast/snapshot.js`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/jev_ultrafast/snapshot.js#L28-L59))
* **Finding**:
  - Line 424 instructs changing `safe` in `snapshot.js` from `!['password','file','hidden'].includes(e.type)` to `!['file','hidden'].includes(e.type)` so password fields are emitted in `actions` with `sensitive: true` and `value == "[MASKED]"`.
  - However, in [`jev_ultrafast/snapshot.js` lines 35–42](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/jev_ultrafast/snapshot.js#L35-L42), `role(e)` only maps `['text','email','url','tel'].includes(e.type)` to `'textbox'` and returns `null` for `e.type === 'password'`. Line 59 (`if (!rname || ...) continue;`) then drops any element whose `role(e)` is `null`.
* **Action Required**: Specify in Target File 4 (`snapshot.js`, line 424) that `role(e)` must include `'password'` (`['text','email','url','tel','password'].includes(e.type)`), and that both `'fill'` and `'click'` (`Open ...`) action entries for a sensitive field carry `sensitive: true`, `value_required: !e.value`, and `value: e.value ? '[MASKED]' : ''`.

### 2. Post-`insertText` `Runtime.evaluate` in `browser_operation` Risks Raising `StalePage` After Browser Mutation (`egm-implementation-readiness`, `egm-codebase-details`)
* **Affected Scope**: [Line 441](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L441) (referencing [`jev_ultrafast/browser.py` lines 127–133](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/jev_ultrafast/browser.py#L127-L133) and [lines 169–186](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/jev_ultrafast/browser.py#L169-L186))
* **Finding**:
  - Line 441 instructs `browser.py` to dispatch bubbling `input` and `change` events on the target element *after* `Input.insertText`.
  - In `browser.py`, `evaluate()` converts `exceptionDetails` into `StalePage("Document changed during evaluation")` for `kind == "fill"`. If a post-`Input.insertText` event listener triggers navigation or destroys the execution context, `evaluate()` raises `StalePage` *after* the browser mutation has already occurred. `Agent.command("tick")` catches `StalePage` and retries the step—violating `AGENTS.md` (*"Never retry a browser mutation"*). Furthermore, CDP `Input.insertText` already dispatches a native `input` event, so dispatching another synthetic `input` event double-triggers listeners.
* **Action Required**: Specify in line 441 that after `Input.insertText`, `browser.py` dispatches only a bubbling `change` event inside a guarded post-mutation `Runtime.evaluate` call that never raises `StalePage` on context destruction, and that any unexpected CDP error occurring *after* `mousePressed` or `Input.insertText` raises `RuntimeError` (`UNCERTAIN_MUTATION_STATE`) rather than `StalePage`.

### 3. Missing Field Schemas and Type Contradictions for `PendingApproval`, `ConstraintEvent`, and `VerificationReport` (`egm-implementation-readiness`, `egm-codebase-details`)
* **Affected Scope**: [Lines 294–316](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L294-L316) and [lines 384–393](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L384-L393) (`jev_ultrafast/contracts.py`, `session.py`, `policy.py`, `verifier.py`)
* **Finding**:
  - Line 391 requires defining `PendingApproval`, `ConstraintEvent`, and `VerificationReport` dataclasses in `contracts.py`, but their fields are not defined anywhere in the document, and `SubgoalResult` (lines 294–295) types `constraint_events` and `pending_approval` as `dict[str, Any]` instead of `ConstraintEvent` and `PendingApproval`.
* **Action Required**: Define the explicit `@dataclass` field schemas for `PendingApproval`, `ConstraintEvent`, and `VerificationReport` in Section 6 / Target File 1 (`contracts.py`).

### 4. Missing State Persistence and Execution Semantics for `awaiting_approval` Pause/Resume (`egm-implementation-readiness`, `egm-codebase-details`)
* **Affected Scope**: [Lines 399–404](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L399-L404) and [lines 472–474](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L472-L474) (`jev_ultrafast/session.py`, `jev_ultrafast/agent.py`)
* **Finding**:
  - `Browser.fresh(page, action)` and `Browser.act(action, page, text=...)` require the full `page` dictionary (`page_key`, `guards`, `marker`, `actions`) and the `action` dictionary from when the tab was paused, plus persisted `history`. Storing only `page_fingerprint` and `target_node` in `PendingApproval` is insufficient.
  - Furthermore, `awaiting_approval` can be entered either via `evaluate_pre_act_gate()` (which has a concrete `action`) or via `operation == "NEEDS_APPROVAL"` (which has `action=None`).
* **Action Required**: Specify that `TabRecord` / `PendingApproval` stores the paused `page` snapshot, `action` dict (or `None` when escalated via `NEEDS_APPROVAL`), and `history`, and define the exact resume behavior for both pre-act-gated actions and model-escalated `NEEDS_APPROVAL`.

### 5. Adding `omitted_actions` to `body["state"]["page"]` Without `.get()` Breaks Existing Offline Tests (`egm-codebase-details`)
* **Affected Scope**: [Line 461](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L461) vs. [`jev_ultrafast/model.py` line 110](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/jev_ultrafast/model.py#L110) and [`tests/test_agent.py` lines 15–29](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/tests/test_agent.py#L15-L29)
* **Finding**:
  - The existing test fixture `page()` in `tests/test_agent.py` does not include `"omitted_actions"` in `state`. Indexing `state["omitted_actions"]` directly in `model.py` will raise `KeyError` in existing unit tests.
* **Action Required**: Specify using `state.get("omitted_actions", 0)` in `model.py` (line 461).

---

## 🟡 Majors

### 6. Nested Scroll Container Offset Omitted from `state["scroll"]` and `fingerprint(state)` (`egm-codebase-details`)
* **Affected Scope**: [Line 427](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L427) vs. [`jev_ultrafast/browser.py` lines 115–117](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/jev_ultrafast/browser.py#L115-L117)
* **Finding**:
  - `fingerprint(state)` hashes `("url", "text", "actions", "scroll")`, not `page_key` or `marker`. If a nested container scrolls while window `scrollY` stays `0`, `state["scroll"]` must include the active container's `scrollTop`/`scrollHeight` so `fingerprint(state)` updates (`page_changed=True`) and avoids false `NO_PROGRESS_3X` blocks.
* **Action Required**: Include the active scroll container's `scrollTop` and `scrollHeight` in `state["scroll"]` in `snapshot.js` (line 427).

### 7. Incomplete Shadow DOM Traversal and Hit-Testing Specification in `snapshot.js` and `browser.py` (`egm-implementation-readiness`, `egm-codebase-details`)
* **Affected Scope**: [Line 426](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L426) and [line 438](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L438)
* **Finding**:
  - In `snapshot.js`, `cache.pageKey` (`querySelectorAll('input,textarea,select')`), `name(e)` (`e.getRootNode().getElementById(id)`), and `guard(e)` must also use shadow-aware traversal.
  - In `browser.py`, `e.contains(hit)` fails across shadow boundaries and must walk upward via `cur = cur.parentElement || cur.getRootNode()?.host`.
* **Action Required**: Specify shadow-aware helpers for `pageKey`, `name(e)`, `guard(e)`, and composed-tree ancestor hit-testing in `browser.py`.

### 8. Canonical Consequence Taxonomy, Keyword Rules, and `__GENERATE__` Guard on Sensitive Fields (`egm-implementation-readiness`)
* **Affected Scope**: [Lines 206, 234–236, 390, 414–416, 475–479](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L414-L416)
* **Finding**:
  - Unify the consequence tag names across Section 4 (`consequence_class`), `SubgoalConstraints.require_approval_for`, and `policy.py`, define deterministic keyword/DOM matching rules, and forbid calling `field_text()` (`__GENERATE__`) on `sensitive: true` fields.
* **Action Required**: Align consequence category names and precedence in `policy.py` and specify blocking `field_text()` on `sensitive: true` targets in `agent.py`.

### 9. `browser-harness` External Package Boundary & `BH_TAB_MARKER=0` (`egm-codebase-details`)
* **Affected Scope**: [Lines 401, 423, 434–442](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L401)
* **Finding**:
  - Since `browser-harness==0.1.13` is an installed package in `.venv`, `session.py` / `browser.py` should set `os.environ.setdefault("BH_TAB_MARKER", "0")` before calling `ensure_daemon()` to disable daemon title-marker injection cleanly without modifying `browser_harness/daemon.py`.
* **Action Required**: Specify `os.environ.setdefault("BH_TAB_MARKER", "0")` in `session.py` / `browser.py`.

### 10. Internal Audit Jargon in `snapshot.js` Rationale (`egm-style`)
* **Affected Scope**: [Line 421](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L421)
* **Finding**: Remove `"identified during the Goldfish audit"` and repetitive `"blindness"` phrasing in line 421.
* **Action Required**: Replace with `"Closes five DOM observation gaps in snapshot.js: omitted password fields, unindexed nested scroll containers, unindexed open Shadow DOM trees, title-marker StalePage collisions, and missing structural consequence metadata."`

---

## 🟢 Nits

11. **Optional `secret_resolver` and `constraints` Keys in `browser_operation(request)` (`egm-codebase-details`)**: Use `request.get("secret_resolver")` and `request.get("constraints")` inside `browser_operation` so existing unit tests calling `browser_operation({"operation": ..., "session": "test"})` remain compatible.
12. **Preserve `MAX_STEPS = 60` Default for Legacy `Agent(url, goals)` (`egm-codebase-details`)**: Note at line 480 that `Agent(url, goals)` defaults to `MAX_STEPS = 60` when `constraints=None`.
13. **Style Polish in `DetailedRange` (`egm-style`)**: Replace `"defense-in-depth safety enforcement"` (line 410), standardize `"Backward-Compatible"` / `"backward-compatible"` (lines 467–468, 480), and replace `"must pass 100% offline"` (line 539) with `"must pass without network access or paid API calls"`.

# Goldfish Annotated Review Report

**Status:** resolved
**Target:** `docs/designs/browser-execution-subsystem.md` (`1..549`)

## Prioritized Findings & AI Suggestions

### [Issue B1] [`egm-zoom-out`, `egm-comprehension`, `egm-design`, `egm-codebase-details`] - Severity: High
*   **Affected Scope:** [Lines 16-209](docs/designs/browser-execution-subsystem.md#L16-L209) (`Overview`, `CUJ 3`, `Technical Plan §1, §2, §4`) & [Lines 401-439](docs/designs/browser-execution-subsystem.md#L401-L439) (`Detailed Implementation Plan`)
*   **Discrepancy Description:** `FeatureRange` claims the `browser-harness` daemon is modified to disable `TAB_MARKER_JS`, intercept `Page.frameRequestedNavigation`, and run a Layer 5 `Network` / `Fetch` guard, whereas `browser-harness==0.1.13` is an installed package in `.venv`, `Page.frameRequestedNavigation` is a passive notification event, and `DetailedRange` only modifies files inside `./jev_ultrafast/`.
*   **Rationale:** The specification and implementation plan must agree on the package boundary and use valid, deadlock-free CDP primitives.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Scope Tier 3 enforcement inside `jev_ultrafast` (`session.py`, `policy.py`, `browser.py`, `snapshot.js`) over `browser-harness`'s `cdp()` channel, set `os.environ.setdefault("BH_TAB_MARKER", "0")` before `ensure_daemon()`, replace `Page.frameRequestedNavigation` and Layer 5 daemon `Fetch` claims with pre-choice/pre-act `href`/`form.action` validation plus post-act URL and `Target.getTargets` (`openerId`) verification, and avoid fragile `Page.addScriptToEvaluateOnNewDocument` monkey-patches per Stage 8 Zoom-Out.
*   ***Proposed Edit:***
    *   Location: `Overview`, `CUJ 3`, `Technical Plan §1, §2, §4`, `Detailed Implementation Plan §2, §5`
    *   Change: Update Tier 3 descriptions, diagrams, and policy table to 4 consequence-based layers executed by `jev_ultrafast` over `browser-harness` CDP sessions (`BH_TAB_MARKER=0`, pre-act URL/form checks, post-act URL/child-tab verification with `Page.stopLoading` and disallowed child-tab closure).

### [Issue B2] [`egm-design`, `egm-implementation-readiness`, `egm-codebase-details`] - Severity: High
*   **Affected Scope:** [Lines 189-198](docs/designs/browser-execution-subsystem.md#L189-L198) (`Technical Plan §3`) & [Lines 424-442](docs/designs/browser-execution-subsystem.md#L424-L442) (`Detailed Implementation Plan §4, §5`)
*   **Discrepancy Description:** (1) `role(e)` in `snapshot.js:40` returns `null` for `e.type === 'password'`, dropping password inputs even when `safe(e)` allows them; (2) typing `$secret:<id>` into a non-password input exposes plaintext on the next `observe()` or `verify_tab()` call unless the node is tainted; (3) `Browser.fresh(page, action)` skips the per-node guard check for `action["kind"] == "fill"`; (4) mutating DOM styles to mask screenshots risks invalidating `pageKey()`, and recording `"$secret:<id>"` in `history["text"]` leaks secret key IDs to external models.
*   **Rationale:** Password fields must be observable as `textbox` controls, secret-filled DOM nodes must remain masked on all subsequent reads, target node guards must run before secret injection, and screenshots/history must not leak secret values or IDs.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Including `'password'` in `role(e)`, tracking `secretNodes: new WeakSet()` in `window.__jevFast`, checking per-node guards for `"fill"` in `Browser.fresh()`, recording `"[SECRET]"` in `history["text"]`, and suppressing screenshot capture (`screenshot_b64 = None`) when a filled sensitive/secret-tainted element is visible resolves all four gaps cleanly.
*   ***Proposed Edit:***
    *   Location: `Technical Plan §3`, `Non-Functional Requirements`, `Detailed Implementation Plan §4, §5`
    *   Change: Specify `'password'` in `role(e)`, `window.__jevFast.secretNodes` taint tracking, `action["kind"] in {"click", "select", "fill"}` in `Browser.fresh()`, `"[SECRET]"` in `history["text"]`, and screenshot suppression (`None`) when filled sensitive controls are visible.

### [Issue B3] [`egm-implementation-readiness`, `egm-codebase-details`] - Severity: High
*   **Affected Scope:** [Lines 219-220](docs/designs/browser-execution-subsystem.md#L219-L220) & [Line 441](docs/designs/browser-execution-subsystem.md#L441) (`Detailed Implementation Plan §5`)
*   **Discrepancy Description:** Calling `evaluate()` after `Input.insertText` in `browser.py` raises `StalePage` on `exceptionDetails` when `kind == "fill"`, which `Agent.command("tick")` catches and retries—violating `AGENTS.md` (*"Never retry a browser mutation"*). Also, `Input.insertText` already dispatches a native `input` event.
*   **Rationale:** Post-mutation event dispatch must never raise `StalePage` or double-fire `input` listeners.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Dispatching a bubbling `change` event inside a guarded post-mutation `Runtime.evaluate` call that ignores navigation context destruction—and raising `RuntimeError` (`UNCERTAIN_MUTATION_STATE`) on any unexpected CDP failure after `mousePressed` or `Input.insertText`—preserves the non-retryable mutation invariant.
*   ***Proposed Edit:***
    *   Location: `Technical Plan §5`, `Detailed Implementation Plan §5`
    *   Change: Specify guarded post-`Input.insertText` `change` event dispatch that never raises `StalePage`, and convert any post-input CDP error into `RuntimeError` (`UNCERTAIN_MUTATION_STATE`).

### [Issue B4] [`egm-comprehension`, `egm-design`, `egm-structural`, `egm-implementation-readiness`, `egm-codebase-details`] - Severity: High
*   **Affected Scope:** [Lines 65-75, 151-152, 224-316](docs/designs/browser-execution-subsystem.md#L224-L316) (`CUJ 2`, `Technical Plan §2, §6`) & [Lines 384-405, 470-479](docs/designs/browser-execution-subsystem.md#L384-L405) (`Detailed Implementation Plan §1, §2, §8`)
*   **Discrepancy Description:** `PendingApproval`, `ConstraintEvent`, and `VerificationReport` lack field definitions in Section 6; `SubgoalResult` types `constraint_events` and `pending_approval` as `dict[str, Any]`; `approval_challenge_id` vs. `approval_token` is inconsistent; and `TabRecord` / `PendingApproval` does not specify storing `paused_page` and `history` or handling both concrete gated actions (`action != None`) and model-escalated `NEEDS_APPROVAL` (`action == None` / late-bound credential inputs) on resume.
*   **Rationale:** Implementers need exact dataclass schemas and unambiguous pause/resume state semantics.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Define `PendingApproval`, `ConstraintEvent`, and `VerificationReport` in Section 6, type `SubgoalResult` with them, standardize on `approval_token`, persist `(paused_page, history, action, input_source)` on `TabRecord`, and define both the concrete-action resume branch and the `NEEDS_APPROVAL` / newly-supplied-credential re-choose resume branch. Note on `expected_result` / `extracted_data` / `claimed_evidence`: because the user explicitly requested `expected_result` and `claimed_evidence` in the brainstorm contract, we retain `expected_result` on `SubgoalRequest` and `claimed_evidence` on `SubgoalResult` with a deterministic definition (populated from the terminal page's URL, title, and matched form/control values), while dropping unpopulated `extracted_data`.
*   ***Proposed Edit:***
    *   Location: `CUJ 2`, `Technical Plan §2, §6`, `Detailed Implementation Plan §1, §2, §8`
    *   Change: Add explicit `@dataclass` definitions for `PendingApproval`, `ConstraintEvent`, and `VerificationReport`; unify `approval_token`; persist `paused_page` and `history` on `TabRecord`; and specify both resume branches.

### [Issue B5] [`egm-zoom-out`, `egm-design`] - Severity: High
*   **Affected Scope:** [Lines 303-316](docs/designs/browser-execution-subsystem.md#L303-L316) (`Technical Plan §6`) & [Lines 482-490](docs/designs/browser-execution-subsystem.md#L482-L490) (`Detailed Implementation Plan §9`)
*   **Discrepancy Description:** `verify_tab` exposes `custom_js_predicate: str | None = None`, allowing Tier 1 callers to pass arbitrary JavaScript strings during `verifying` state in violation of `AGENTS.md` (*"Never let the model emit selectors or executable code"*), Non-Goal 6, and the read-only verification guarantee.
*   **Rationale:** Verification must be strictly declarative and code-owned.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Remove `custom_js_predicate` from `verify_tab` in Section 6 and Target File 9 (`verifier.py`), keeping declarative checks (`url_matches`, `title_matches`, `required_text`, `forbidden_text`, `control_values`).
*   ***Proposed Edit:***
    *   Location: `Technical Plan §6`, `Detailed Implementation Plan §9`
    *   Change: Remove `custom_js_predicate` parameter and description from `verify_tab`.

### [Issue B6] [`egm-codebase-details`] - Severity: High
*   **Affected Scope:** [Line 461](docs/designs/browser-execution-subsystem.md#L461) (`Detailed Implementation Plan §7`)
*   **Discrepancy Description:** Existing unit tests in `tests/test_agent.py` pass a `page()` fixture without `"omitted_actions"`. Direct `state["omitted_actions"]` indexing in `model.py` raises `KeyError`.
*   **Rationale:** Must use `state.get("omitted_actions", 0)` to keep existing offline tests passing.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Specify `state.get("omitted_actions", 0)` in `model.py`.
*   ***Proposed Edit:***
    *   Location: `Detailed Implementation Plan §7`
    *   Change: Specify `"omitted_actions": state.get("omitted_actions", 0)` in `model.py`.

### [Issue M1] [`egm-zoom-out`, `egm-implementation-readiness`] - Severity: Medium
*   **Affected Scope:** [Lines 100, 206, 234-236, 262-280, 389-391, 414-416, 465-481](docs/designs/browser-execution-subsystem.md#L465-L481)
*   **Discrepancy Description:** (1) Proposing a separate `SubgoalRunner` class with its own loop duplicates `Agent`'s step loop in `agent.py`; (2) consequence category names differ between Section 4 (`submit`, `financial`, `communication`, `credential`) and `SubgoalConstraints.require_approval_for` (`submit_in_read_only`, `payment`, `send`, `credentials`); (3) calling `field_text()` (`__GENERATE__`) on `sensitive: true` fields is not explicitly blocked; and (4) `ReasonCode` trigger rules need deterministic definitions.
*   **Rationale:** A single unified loop in `Agent` (with `run_subgoal` as a thin wrapper) and a single canonical consequence taxonomy prevent implementation drift.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Unify `Agent` and `run_subgoal` in `agent.py`, standardize consequence categories on `{"credentials", "payment", "destructive", "submit_in_read_only", "external_navigation", "send"}` with explicit DOM/keyword rules and deterministic `ReasonCode` mappings, and forbid calling `field_text()` on `sensitive: true` targets.
*   ***Proposed Edit:***
    *   Location: `Technical Plan §1, §4, §6`, `Detailed Implementation Plan §1, §3, §8`
    *   Change: Make `run_subgoal` a thin entrypoint over `Agent`, unify consequence category names and keyword rules, block `field_text()` on `sensitive: true` fields, and specify deterministic triggers for all `ReasonCode` literals.

### [Issue M2] [`egm-design`, `egm-structural`] - Severity: Medium
*   **Affected Scope:** [Lines 132-158, 400-406](docs/designs/browser-execution-subsystem.md#L132-L158) (`Technical Plan §2`, `Detailed Implementation Plan §2`)
*   **Discrepancy Description:** The `stateDiagram-v2` omits `verifying --> verifying` (when `verify_tab` attaches to a completed sub-goal), `verifying --> leased` (recovery sub-goal), `awaiting_approval --> available` (on stale page during resume), and `close_tab()` from `verifying` / `awaiting_approval`. Also, parent lease release on `follow_new_tabs="auto"` child-tab handoff and pre-mutation lease validation (`assert_active_lease`) in `Browser.act()` are not explicit.
*   **Rationale:** State transitions and lease handoff rules must cover all CUJ paths.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Add the missing transitions to `stateDiagram-v2` and `session.py`, specify releasing the parent tab's mutation lease (`leased -> available`) when switching to a child tab under `follow_new_tabs="auto"`, and check `assert_active_lease` before every mutation in `Browser.act()`.
*   ***Proposed Edit:***
    *   Location: `Technical Plan §2`, `Detailed Implementation Plan §2, §5`
    *   Change: Update `stateDiagram-v2`, child-tab lease handoff rules, and pre-act lease validation.

### [Issue M3] [`egm-design`, `egm-implementation-readiness`, `egm-codebase-details`] - Severity: Medium
*   **Affected Scope:** [Lines 213-218, 426-427, 438](docs/designs/browser-execution-subsystem.md#L426-L438) (`Technical Plan §5`, `Detailed Implementation Plan §4, §5`)
*   **Discrepancy Description:** (1) `fingerprint(state)` hashes `state["scroll"]`, not `page_key` or `marker`, so nested scroll container offsets must be included in `state["scroll"]`; (2) Shadow DOM traversal in `snapshot.js` must also cover `cache.pageKey`, `name(e)` (`e.getRootNode().getElementById`), and `guard(e)` (`e.getRootNode()?.host`); (3) `e.contains(hit)` in `browser.py` returns `false` across Shadow DOM boundaries and must walk upward via `cur = cur.parentElement || cur.getRootNode()?.host`.
*   **Rationale:** Ensures nested scrolling updates `fingerprint(state)` (`page_changed=True`) and open Shadow DOM elements pass freshness and occlusion checks.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Specify including active container scroll offsets in `state["scroll"]` and using shadow-aware helpers in `snapshot.js` and `browser.py`.
*   ***Proposed Edit:***
    *   Location: `Technical Plan §5`, `Detailed Implementation Plan §4, §5`
    *   Change: Add `state["scroll"]` container offset and shadow-aware `pageKey`, `name(e)`, `guard(e)`, and composed-ancestor hit-testing specifications.

### [Issue M4] [`egm-zoom-out`, `egm-structural`, `egm-style`] - Severity: Medium
*   **Affected Scope:** [Lines 3, 11, 20-25, 50, 162, 187, 190-194, 213-220, 326, 332-374, 421](docs/designs/browser-execution-subsystem.md#L1-L425)
*   **Discrepancy Description:** (1) `FeatureRange` (`1..379`) embeds literal JS code snippets (`const safe = ...`) and low-level DOM API calls that belong in `DetailedRange`; (2) `FeatureRange` uses machine-local `file:///c:/Users/Albe/...` URIs instead of portable relative repository paths; (3) lines 3, 50, 326, and 421 contain internal review-process jargon (`"Goldfish"`) or subjective phrasing (`"poorly designed web applications"`).
*   **Rationale:** Keeps `FeatureRange` at architectural altitude, makes all markdown links portable across machines, and maintains objective engineering prose.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Elevate `Technical Plan §3 & §5` to behavioral invariants (keeping the JS details in `Detailed Implementation Plan §4 & §5`), convert all `file:///c:/Users/Albe/...` links to portable relative paths, and apply the style replacements.
*   ***Proposed Edit:***
    *   Location: Throughout `docs/designs/browser-execution-subsystem.md`
    *   Change: Replace machine-local `file://` links with relative paths, move literal JS snippets out of `1..379`, and remove internal review jargon/subjective phrasing.

### [Issue N1-N4] [`egm-comprehension`, `egm-design`, `egm-structural`, `egm-style`, `egm-codebase-details`] - Severity: Low
*   **Affected Scope:** [Lines 15, 35-36, 82, 96-110, 180, 192, 214, 362, 373-374, 410, 424, 434-442, 467-480, 539](docs/designs/browser-execution-subsystem.md)
*   **Discrepancy Description:** Minor comprehension, formatting, and test-compatibility items: (N1) explain `TypeSafe` (`/v1/systemone`) inline, align `one-time-code` in `autocomplete` lists, and update `TYPE_TEXT` operation description when `inputs` are present; (N2) check `scrollHeight > clientHeight + 20 && clientHeight >= 80` before calling `getComputedStyle(el).overflowY`; (N3) use optional `request.get("secret_resolver")` / `request.get("constraints")` in `browser_operation` and keep `MAX_STEPS = 60` default when `Agent(..., constraints=None)` is used; (N4) replace `\n` with `<br/>` in Mermaid flowchart labels, remove extra horizontal rule at line 362, normalize list markers, and standardize `"Backward Compatibility"` / `"backward-compatible"`.
*   **Rationale:** Low-cost improvements that eliminate test friction and polish document readability.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Apply all four sets of improvements directly while updating the document.
*   ***Proposed Edit:***
    *   Location: `Overview`, `Technical Plan`, `Detailed Implementation Plan`
    *   Change: Apply inline `TypeSafe` explanation, `autocomplete` list alignment, `scrollHeight` pre-filter, `browser_operation` optional dict keys, `MAX_STEPS = 60` legacy default, and Markdown/Mermaid formatting polish.

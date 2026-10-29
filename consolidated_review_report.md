# Consolidated EGM Goldfish Review Report (`consolidated_review_report.md`)

**Target Document**: [`docs/designs/browser-execution-subsystem.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md)
**Semantic Split**: `FeatureRange: 1..379`, `DetailedRange: 380..549`
**Stage Reports**:
- Stage 5 (Initial Zoom Out): [`zoom_out_report.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/zoom_out_report.md)
- Stage 6 (Comprehension Test): [`comprehension_report.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/comprehension_report.md)
- Stage 7 (Critic Review — Design, Structural, Style): [`critic_report.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/critic_report.md)
- Stage 8 (Intermediate Zoom Out): [`intermediate_zoom_out_report.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/intermediate_zoom_out_report.md)
- Stage 9 (Implementation Readiness — Tech Lead, Codebase Details, Style): [`implementation_readiness_report.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/implementation_readiness_report.md)

---

## Prioritized Consolidated Action Items

### 🔴 Blockers (Must Fix Before Implementation)

#### B1. Align Tier 3 Boundary with `browser-harness` Package Reality & Remove Invalid CDP Hooks
* **Sources**: Stage 5 `#1`, Stage 6 `#1`, Stage 7 `Blocker 1`, Stage 8 `#1`, Stage 9 `Major 9`
* **Affected Lines**: `16–17`, `79–82`, `106–111`, `155–157`, `203–210`, `327`, `401`, `439`
* **Problem**:
  1. `FeatureRange` (`1..379`) claims that the `browser-harness` daemon is modified to disable `TAB_MARKER_JS`, intercept `Page.frameRequestedNavigation` and `Target.targetCreated`, and enforce a "Layer 5 Secondary Network Guard (`Network` / `Fetch`)". However, `browser-harness==0.1.13` is an external package in `.venv`, and `DetailedRange` (`380..549`) only modifies files inside `./jev_ultrafast/`.
  2. `Page.frameRequestedNavigation` is a passive CDP notification event, not a synchronous blocker, and client-side `Fetch.requestPaused` interception over `browser-harness`'s 500-event ring buffer risks deadlocks (as noted in line 327).
* **Consolidated Fix**:
  - Remove claims of editing `browser_harness/daemon.py`, remove `Page.frameRequestedNavigation`, and drop "Layer 5 Secondary Network Guard" (making the safety engine a 4-layer architecture inside `jev_ultrafast`).
  - Specify that `jev_ultrafast` sets `os.environ.setdefault("BH_TAB_MARKER", "0")` before calling `ensure_daemon()` (plus strips any `'🐴 '` prefix in `snapshot.js`), enforces domain boundaries via pre-choice/pre-act `href`/`form.action` validation in `snapshot.js` + `policy.py` + `browser.py`, and verifies post-action URLs and child targets (`Target.getTargets` with `openerId == tab_id`), calling `Page.stopLoading` / closing disallowed child tabs if a JS redirect crosses a domain boundary.

#### B2. Fix `snapshot.js` `role(e)` Dropping `input[type="password"]` and Add `secretNodes` Taint Tracking + `fill` Guard
* **Sources**: Stage 7 `Blocker 2`, Stage 7 `Major 8`, Stage 8 `#2`, Stage 9 `Blocker 1`
* **Affected Lines**: `74`, `152`, `189–198`, `303–316`, `366–368`, `424`, `440`, `442`, `517`
* **Problem**:
  1. Changing `safe(e)` in `snapshot.js` to allow `'password'` (line 424) still leaves `<input type="password">` invisible because `role(e)` (`snapshot.js:40`) only maps `['text','email','url','tel']` to `'textbox'` and returns `null` for `'password'`, causing line 59 (`if (!rname) continue;`) to drop the element.
  2. If an opaque `$secret:<id>` is typed into a non-password input (e.g., an API key textbox or toggled password field), `isSensitive(e)` is `false` on the next `observe()` or `verify_tab()` call, leaking the raw secret value into `actions`, `pageKey()`, `guards`, and TypeSafe prompts.
  3. `Browser.fresh(page, action)` (`browser.py:89`) only checks the per-node guard for `action["kind"] in {"click", "select"}`, skipping `"fill"` before dereferencing a secret.
* **Consolidated Fix**:
  - In `snapshot.js` (line 424), update `role(e)` to map `['text','email','url','tel','password']` to `'textbox'`.
  - Add `secretNodes: new WeakSet()` to `window.__jevFast`; when `Browser.act()` resolves and inserts a `$secret:<id>` value into `action["node"]`, add that DOM element to `window.__jevFast.secretNodes` and treat `cache.secretNodes.has(e)` as `sensitive: true` in both `snapshot.js` and `verify_tab`.
  - Update `Browser.fresh(page, action)` in `browser.py` to check `[c.pageKey(), c.guard(c.nodes.get(node))]` for `action["kind"] in {"click", "select", "fill"}`.
  - Record `"[SECRET]"` in `history["text"]` / `recent_actions` (never exposing secret values or raw `$secret:<id>` IDs to external models), and suppress screenshot capture (`screenshot_b64 = None`) whenever a filled sensitive/secret-tainted element is visible rather than mutating DOM styles.

#### B3. Guard Post-`insertText` Event Dispatch in `browser.py` So Completed Mutations Never Raise `StalePage`
* **Sources**: Stage 9 `Blocker 2` (`egm-implementation-readiness` #4 & `egm-codebase-details` #2)
* **Affected Lines**: `219–220`, `441`
* **Problem**:
  - Line 441 instructs `browser.py` to dispatch `input` and `change` events via `Runtime.evaluate` *after* `Input.insertText`. In `browser.py:127–133`, `evaluate()` raises `StalePage` on `exceptionDetails` when `kind == "fill"`. If a listener navigates on input/change, `StalePage` is raised *after* the browser mutation already occurred, causing `Agent` to retry the mutation in violation of `AGENTS.md`. Additionally, `Input.insertText` already fires a native `input` event.
* **Consolidated Fix**:
  - Specify in line 441 that after `Input.insertText`, `browser.py` dispatches a bubbling `change` event (and only synthesizes `input` if `e.value` did not update) inside a guarded post-mutation `Runtime.evaluate` call that ignores navigation context destruction, and that any unexpected CDP failure occurring *after* `mousePressed` or `Input.insertText` raises `RuntimeError` (`UNCERTAIN_MUTATION_STATE`) rather than `StalePage`.

#### B4. Define Complete Schemas for `PendingApproval`, `ConstraintEvent`, and `VerificationReport` and Fix `awaiting_approval` Resume State
* **Sources**: Stage 5 `#2`, Stage 6 `#2`, Stage 6 `#3`, Stage 7 `Blocker 3`, Stage 7 `Major 5`, Stage 8 `#3`, Stage 8 `#4`, Stage 9 `Blocker 3`, Stage 9 `Blocker 4`
* **Affected Lines**: `65–75`, `151–152`, `241–316`, `384–405`, `472–477`
* **Problem**:
  1. `PendingApproval`, `ConstraintEvent`, and `VerificationReport` are referenced in `contracts.py` (line 391) and `verify_tab` (line 314), but their dataclass fields are never defined in Section 6, and `SubgoalResult` types `constraint_events` and `pending_approval` as `dict[str, Any]`.
  2. Line 152 uses `approval_challenge_id` while line 403 uses `approval_token`.
  3. `PendingApproval` / `TabRecord` does not store the paused `page` snapshot (`marker`, `page_key`, `guards`) and `history` required by `Browser.fresh(page, action)` and `Browser.act(action, page, text)` upon resume, nor does it distinguish between resuming a pre-act-gated concrete action (`action != None`) vs. a model-escalated `NEEDS_APPROVAL` (`action == None` or late-bound credential input).
* **Consolidated Fix**:
  - Define `PendingApproval`, `ConstraintEvent`, and `VerificationReport` explicitly in Section 6 and type `SubgoalResult.constraint_events: list[ConstraintEvent]` and `SubgoalResult.pending_approval: PendingApproval | None`.
  - Standardize on `approval_token: str` (removing `approval_challenge_id`).
  - Store `(paused_page, history, action, input_source)` on `TabRecord` when entering `awaiting_approval`, and specify both resume branches: (a) if `action` is present and has a bound input (or a newly supplied `request.inputs[key]`), verify `Browser.fresh(paused_page, action)` and execute it once; (b) if `action is None` (`NEEDS_APPROVAL`) or requires selecting a newly supplied credential key, verify `Browser.fresh(paused_page)` and resume at `choose()` with the updated `inputs` and approved category.

#### B5. Remove `custom_js_predicate` from `verify_tab`
* **Sources**: Stage 5 `#4`, Stage 7 `Blocker 4`, Stage 8 `Summary`
* **Affected Lines**: `312`, `487`, `489`
* **Problem**:
  - Exposing `custom_js_predicate: str | None = None` on `verify_tab` violates `AGENTS.md` (*"Never let the model emit selectors or executable code"*), Non-Goal 6 (line 45), and the read-only invariant of the `verifying` state (line 153).
* **Consolidated Fix**:
  - Remove `custom_js_predicate` from `verify_tab` in Section 6 (line 312) and Target File 9 (`verifier.py`, lines 487–489). Keep `verify_tab` strictly declarative (`url_matches`, `title_matches`, `required_text`, `forbidden_text`, `control_values`).

#### B6. Use `state.get("omitted_actions", 0)` in `model.py` to Preserve Existing Offline Unit Tests
* **Sources**: Stage 9 `Blocker 5` (`egm-codebase-details` #3)
* **Affected Lines**: `461`
* **Problem**:
  - The `page()` fixture in `tests/test_agent.py:15–29` does not contain `"omitted_actions"`. Direct `state["omitted_actions"]` indexing in `model.py` raises `KeyError` in existing tests.
* **Consolidated Fix**:
  - Specify `state.get("omitted_actions", 0)` in `model.py` (line 461).

---

### 🟡 Majors (Architectural, Completeness & Correctness Refinements)

#### M1. Unify `Agent` and `SubgoalRunner` Loop in `agent.py` & Prune Speculative Schema Fields
* **Sources**: Stage 5 `#2`, Stage 5 `#3`, Stage 8 `#4`, Stage 8 `#5`, Stage 9 `Major 8`
* **Affected Lines**: `100`, `206`, `234–236`, `251`, `262–299`, `389–391`, `414–416`, `465–481`
* **Consolidated Fix**:
  - Extend `Agent` directly in `agent.py` to support leased tabs, `inputs`, `constraints`, and `needs_approval` state transitions, making `run_subgoal(request)` a thin wrapper around `Agent` instead of maintaining two parallel step loops.
  - Unify consequence categories (`credentials`, `payment`, `destructive`, `submit_in_read_only`, `external_navigation`, `send`) and deterministic keyword/DOM matching rules in `policy.py`, forbid calling `field_text()` (`__GENERATE__`) on `sensitive: true` fields, and define exact triggers for `ReasonCode` values while pruning unpopulated fields.

#### M2. Complete `TabRegistry` State Diagram & Lease Rules (`verifying`, Child-Tab Handoff, Lease Validation)
* **Sources**: Stage 7 `Major 6` (`egm-design` #5 & `egm-structural` #2)
* **Affected Lines**: `132–158`, `400–406`
* **Consolidated Fix**:
  - Update `stateDiagram-v2` and `session.py` to include `verifying --> verifying : acquire_lease(verifier, mode="read_only")`, `verifying --> leased : acquire_lease(actor, mode="mutate")`, `awaiting_approval --> available : stale_page_on_resume`, and `close_tab()` from `available | verifying | awaiting_approval | failed`.
  - Specify that on `follow_new_tabs="auto"` child-tab handoff, the parent tab's mutation lease is released (`leased -> available`) while the child tab acquires a mutation lease (`available -> leased`), and `assert_active_lease(tab_id, lease_token, mode="mutate")` is checked before every mutating call in `Browser.act()`.

#### M3. Specify Shadow-Aware `pageKey`/`name`/`guard`, Composed Hit-Testing, and `state["scroll"]` Offset
* **Sources**: Stage 7 `Major 7`, Stage 9 `Major 6`, Stage 9 `Major 7`
* **Affected Lines**: `213–218`, `426–427`, `438`
* **Consolidated Fix**:
  - In `snapshot.js` (lines 426–427), use the recursive open-Shadow-DOM collector in both `actions` and `cache.pageKey`, resolve `aria-labelledby` via `e.getRootNode().getElementById(id)`, walk `e.getRootNode()?.host` in `guard(e)`, and include the active scroll container's `scrollTop`/`scrollHeight` in `state["scroll"]` so `fingerprint(state)` updates when scrolling inside a modal.
  - In `browser.py` (line 438), drill down via `while (hit?.shadowRoot) ...` and walk upward across composed ancestors (`cur = cur.parentElement || cur.getRootNode()?.host`) instead of `e.contains(hit)`.

#### M4. Elevate Low-Level JS Weeds Out of `FeatureRange` (`1..379`) and Standardize Repository Links
* **Sources**: Stage 5 `#5`, Stage 7 `Major 9`, Stage 7 `Major 10`, Stage 8 `#5`, Stage 9 `Major 10`
* **Affected Lines**: `11`, `20–25`, `162`, `187`, `190–194`, `213–220`, `326`, `332–374`, `421`
* **Consolidated Fix**:
  - Keep `Technical Plan §3 & §5` (`1..379`) at the architectural contract level (moving literal JS expressions to `Detailed Implementation Plan §4 & §5`), replace machine-local `file:///c:/Users/Albe/...` URIs with portable relative repository links (`../../jev_ultrafast/...` or `./jev_ultrafast/...`), and replace subjective/internal phrasing (`"poorly designed web applications"` at line 326, `"Goldfish audit"` at line 421).

---

### 🟢 Nits (Clarity, Style & Test Compatibility)

* **N1. Brief Inline Explanation of `TypeSafe` & Consistent `autocomplete` List** (Stage 6 `#4`, `#5`, Stage 7 `Nit 14`): Add a one-sentence explanation of TypeSafe (`/v1/systemone` multi-head constrained-choice API) in `Overview`, include `one-time-code` consistently in both Section 3 and Target File 4, and update the `TYPE_TEXT` operation description in `model.py` when `inputs` are present.
* **N2. Scroll Container Performance Check Ordering** (Stage 7 `Nit 14`): In `snapshot.js`, check `el.scrollHeight > el.clientHeight + 20 && el.clientHeight >= 80` before calling `getComputedStyle(el).overflowY`.
* **N3. Backwards-Compatible `browser_operation` Dict Keys & `MAX_STEPS = 60` Default** (Stage 9 `Nit 11`, `Nit 12`): Pass `secret_resolver` and `constraints` via optional `request.get(...)` keys in `browser_operation`, and preserve `MAX_STEPS = 60` when `Agent(url, goals)` is called with `constraints=None`.
* **N4. Formatting, Mermaid Labels & Copy Editing** (Stage 7 `Nits 11–13`, Stage 9 `Nit 13`): Remove internal review jargon (`"& EGM Goldfish Review Panel"`, `"Goldfish objections"`), replace `\n` in Mermaid node labels with `<br/>`, remove the extra horizontal rule at line 362, normalize list markers, and standardize `"Backward Compatibility"` / `"backward-compatible"`.

---

## Cross-Stage Traceability Matrix

| Stage & Reviewer | Raw Finding ID | Severity | Consolidated Action Item |
| :--- | :--- | :--- | :--- |
| **Stage 5 (`egm-zoom-out`)** | Finding 1: Unimplemented Daemon-Side Interception | 🔴 Blocker | **B1** |
| **Stage 5 (`egm-zoom-out`)** | Finding 2: Speculative Fields & Reason Codes | 🟡 Major | **B4, M1** |
| **Stage 5 (`egm-zoom-out`)** | Finding 3: Duplicating Loop (`SubgoalRunner` vs `Agent`) | 🟡 Major | **M1** |
| **Stage 5 (`egm-zoom-out`)** | Finding 4: `custom_js_predicate` in `verify_tab` | 🟡 Major | **B5** |
| **Stage 5 (`egm-zoom-out`)** | Finding 5: Altitude Drops (JS Weeds in `1..379`) | 🟡 Major | **M4** |
| **Stage 5 (`egm-zoom-out`)** | Finding 6: Module Proliferation | 🟢 Nit | **M1** |
| **Stage 6 (`egm-comprehension`)** | Finding 1: `browser-harness` Daemon Mismatch | 🟡 Major | **B1** |
| **Stage 6 (`egm-comprehension`)** | Finding 2: Missing Schemas (`PendingApproval`, etc.) | 🟡 Major | **B4** |
| **Stage 6 (`egm-comprehension`)** | Finding 3: `approval_challenge_id` vs `approval_token` | 🟢 Nit | **B4** |
| **Stage 6 (`egm-comprehension`)** | Finding 4: Sensitive `autocomplete` List Mismatch | 🟢 Nit | **N1** |
| **Stage 6 (`egm-comprehension`)** | Finding 5: Unexplained `TypeSafe` Service | 🟢 Nit | **N1** |
| **Stage 7 (`egm-design`)** | Blocker 1: Phantom Daemon Layer & `frameRequestedNavigation` | 🔴 Blocker | **B1** |
| **Stage 7 (`egm-design`)** | Blocker 2: Post-Injection Secret Exfiltration & `fill` Guard | 🔴 Blocker | **B2** |
| **Stage 7 (`egm-design`)** | Blocker 3: Broken `awaiting_approval` Pause/Resume | 🔴 Blocker | **B4** |
| **Stage 7 (`egm-design`)** | Blocker 4: `custom_js_predicate` in `verify_tab` | 🔴 Blocker | **B5** |
| **Stage 7 (`egm-structural`)** | Major 5: Incomplete Schema Definitions in §6 | 🟡 Major | **B4** |
| **Stage 7 (`egm-design`, `egm-structural`)** | Major 6: `TabRegistry` State Diagram & Lease Transitions | 🟡 Major | **M2** |
| **Stage 7 (`egm-design`)** | Major 7: Open Shadow DOM Hit-Testing (`e.contains`) | 🟡 Major | **M3** |
| **Stage 7 (`egm-design`)** | Major 8: Screenshot Masking & History Secret Leakage | 🟡 Major | **B2** |
| **Stage 7 (`egm-structural`)** | Major 9: Non-Portable `file:///c:/Users/...` Links | 🟡 Major | **M4** |
| **Stage 7 (`egm-style`)** | Major 10: Subjective Phrasing in Alternative 1 | 🟡 Major | **M4** |
| **Stage 7 (`egm-style`, `egm-structural`, `egm-design`)** | Nits 11–14: Jargon, Mermaid `\n`, Scroll Ordering, `TYPE_TEXT` Label | 🟢 Nit | **N1, N2, N4** |
| **Stage 8 (`egm-zoom-out`)** | Finding 1: Reject `addScriptToEvaluateOnNewDocument` Monkey-Patch | 🟡 Major | **B1** |
| **Stage 8 (`egm-zoom-out`)** | Findings 2–5: Minimalist Synthesis of Blockers 2–4 & Stage 5 | 🟡 Major | **B2, B4, B5, M1, M4** |
| **Stage 9 (`egm-implementation-readiness`, `egm-codebase-details`)** | Blocker 1: `role(e)` Drops `input[type="password"]` in `snapshot.js` | 🔴 Blocker | **B2** |
| **Stage 9 (`egm-implementation-readiness`, `egm-codebase-details`)** | Blocker 2: Post-`insertText` `evaluate()` Raises `StalePage` | 🔴 Blocker | **B3** |
| **Stage 9 (`egm-implementation-readiness`, `egm-codebase-details`)** | Blocker 3: Missing `PendingApproval` / `ConstraintEvent` / `VerificationReport` Fields | 🔴 Blocker | **B4** |
| **Stage 9 (`egm-implementation-readiness`, `egm-codebase-details`)** | Blocker 4: Missing `paused_page` / `history` State on Resume | 🔴 Blocker | **B4** |
| **Stage 9 (`egm-codebase-details`)** | Blocker 5: `state["omitted_actions"]` Breaks `tests/test_agent.py` | 🔴 Blocker | **B6** |
| **Stage 9 (`egm-codebase-details`)** | Major 6: Nested Scroll Offset Omitted from `state["scroll"]` | 🟡 Major | **M3** |
| **Stage 9 (`egm-implementation-readiness`, `egm-codebase-details`)** | Major 7: Shadow DOM `pageKey` / `name` / `guard` & Hit-Testing | 🟡 Major | **M3** |
| **Stage 9 (`egm-implementation-readiness`)** | Major 8: Consequence Taxonomy & Blocking `field_text` on Sensitive Fields | 🟡 Major | **M1** |
| **Stage 9 (`egm-codebase-details`)** | Major 9: `BH_TAB_MARKER=0` Environment Variable for Daemon | 🟡 Major | **B1** |
| **Stage 9 (`egm-style`, `egm-codebase-details`)** | Major 10 & Nits 11–13: Style Polish, `browser_operation` Keys, `MAX_STEPS` | 🟢 Nit | **M4, N3, N4** |

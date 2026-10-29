# Stage 8 Intermediate Zoom-Out Review (`intermediate_zoom_out_report.md`)

**Target Document**: [`docs/designs/browser-execution-subsystem.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L1-L549)
**Inputs Evaluated**:
1. [`docs/designs/browser-execution-subsystem.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md)
2. [`zoom_out_report.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/zoom_out_report.md) (Stage 5 Initial Zoom-Out Report)
3. [`comprehension_report.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/comprehension_report.md) (Stage 6 Comprehension Report)
4. [`critic_report.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/critic_report.md) (Stage 7 Critic Report)

---

## Executive Assessment

The Stage 6 (`comprehension_report.md`) and Stage 7 (`critic_report.md`) reviews converge with the Stage 5 Initial Zoom-Out findings on the core architectural issues:
- Removing the phantom `browser-harness` daemon modifications and Layer 5 `Network`/`Fetch` interception claims;
- Removing `custom_js_predicate` from `verify_tab` to uphold `AGENTS.md` and read-only verification guarantees;
- Defining the missing boundary contracts (`PendingApproval`, `ConstraintEvent`, `VerificationReport`) and unifying `approval_token` terminology;
- Fixing real security and correctness gaps in secret taint tracking (`secretNodes`), `Browser.fresh()` guard checks on `fill` actions, `awaiting_approval` resume semantics, and Shadow DOM hit-testing.

However, from a **Zoom-Out / YAGNI / Minimalism** perspective, one part of a Stage 7 Critic recommendation risks introducing fragile browser-side complexity, and the Stage 5 structural simplifications (pruning dead schema fields/reason codes, avoiding a duplicate runner loop in `agent.py`, keeping `1..379` free of low-level JS weeds, and preventing module bloat) must be enforced when applying the Stage 6/7 fixes.

---

## Findings on Stage 6 & Stage 7 Proposed Mitigations

### 1. Guard Against Over-Engineering Critic Blocker 1(b): Skip `Page.addScriptToEvaluateOnNewDocument` Monkey-Patching of `window.open` / `window.navigation`
* **Severity**: Major
* **Affected Scope**: [`critic_report.md` Blocker 1](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/critic_report.md#L10-L20), [`docs/designs/browser-execution-subsystem.md` §1, §2, §4](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L79-L210)
* **Simplicity Rationale**:
  - **What to Adopt from Stage 6 #1 & Stage 7 Blocker 1**: Both reports rightly flag that `browser-harness` is an external package whose daemon is not modified, `Page.frameRequestedNavigation` is a passive notification event rather than a synchronous blocker, and Layer 5 daemon `Fetch`/`Network` interception must be removed.
  - **Where Critic Blocker 1(b) Over-Reaches**: Critic Blocker 1(b) proposes injecting a persistent bootstrap script via `Page.addScriptToEvaluateOnNewDocument` on leased tabs to intercept `window.navigation.addEventListener('navigate', ...)` and monkey-patch `window.open`.
  - Injecting persistent `NewDocument` monkey-patches into arbitrary third-party web applications violates YAGNI and creates high breakage risk: the Navigation API is not supported uniformly across frames/contexts, `window.open` monkey-patching conflicts with site scripts and COOP/cross-origin frame rules, and `Page.addScriptToEvaluateOnNewDocument` persists across navigations on the target unless explicitly tracked and removed by script identifier.
  - **Minimalist Resolution**: Keep domain enforcement purely within `jev_ultrafast`'s existing observation and execution boundary:
    1. **Pre-Choice & Pre-Act Gate (`snapshot.js` + `policy.py` + `browser.py`)**: Resolve `a.href`, `form.action`, and `button.formAction` in `snapshot.js` and block disallowed cross-domain targets before `Browser.act()` dispatches any click/submit.
    2. **Post-Act URL & Child-Target Verification (`browser.py` + `session.py`)**: Immediately after `Browser.act()`, check the current page URL and any newly spawned child targets (`Target.getTargets` where `openerId == tab_id`). If a JS redirect or `window.open` navigated to a disallowed domain, immediately call `Page.stopLoading` / close the disallowed child target, record `DOMAIN_BOUNDARY_BLOCKED`, and halt the sub-goal (`status="blocked"`).

### 2. Adopt Critic Blocker 2 (`secretNodes` Taint Tracking & `fill` Guard) as a Tiny Local Invariant, Not a New Abstraction
* **Severity**: Major
* **Affected Scope**: [`critic_report.md` Blocker 2 & Major 8](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/critic_report.md#L21-L30), [`docs/designs/browser-execution-subsystem.md` §3 & Detailed Plan §4, §5](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L189-L198)
* **Simplicity Rationale**:
  - Critic Blocker 2 identifies a genuine security flaw: if `$secret:<id>` is typed into an input whose DOM attributes do not match static password heuristics (e.g., an API key textbox or a toggled password field), the very next `observe()` or `verify_tab()` call would read back `e.value` in plaintext.
  - **Minimalist Resolution**: Adopt both fixes with minimal code footprint:
    1. Add a `secretNodes: new WeakSet()` property on the existing `window.__jevFast` cache object; when `Browser.act()` fills a `$secret:<id>` value, add the target node to `window.__jevFast.secretNodes` and treat `cache.secretNodes.has(e)` as `sensitive: true` in `snapshot.js` and `verify_tab`.
    2. Include `"fill"` alongside `"click"` and `"select"` in `Browser.fresh(page, action)` (`if action["kind"] in {"click", "select", "fill"}`) so target node identity is always verified before dereferencing a secret.
    3. Per Critic Major 8, never mutate DOM styles to mask screenshots (which would invalidate `pageKey()`/`marker`); simply set `screenshot_b64 = None` when a filled sensitive or secret-tainted control is visible, and always record `"[SECRET]"` in `history["text"]` / `recent_actions`.

### 3. Adopt Critic Blocker 3 & Comprehension #3 with One Unified `approval_token` Flow
* **Severity**: Major
* **Affected Scope**: [`critic_report.md` Blocker 3](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/critic_report.md#L31-L41), [`comprehension_report.md` #3](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/comprehension_report.md#L26-L32), [`docs/designs/browser-execution-subsystem.md` §2 & Detailed Plan §2, §8](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L151-L153)
* **Simplicity Rationale**:
  - Comprehension #3 notes the confusing split between `approval_challenge_id` and `approval_token`, and Critic Blocker 3 notes that `awaiting_approval` can be entered either with a concrete gated action (`action != None`) or when the model escalates (`operation == "NEEDS_APPROVAL"` or a credential input is missing).
  - **Minimalist Resolution**:
    1. Eliminate `approval_challenge_id` completely; use a single `approval_token: str` minted when entering `awaiting_approval` and returned inside `PendingApproval`.
    2. Persist `(paused_page, history, pending_action, input_source)` on the tab's `PendingApproval` record. On resume with `approval_token`: verify `Browser.fresh(paused_page, pending_action)`; if `pending_action` is present and has a bound input (including a newly supplied `inputs[key]` on the resume request), execute it once and continue; if `pending_action is None` (`NEEDS_APPROVAL`), resume the loop at `choose()` with the caller's updated `inputs` and approval flag.

### 4. Reconcile Stage 6 #2 & Stage 7 Major 5 with Stage 5 Finding #2: Define Compact Contracts While Pruning Speculative Fields
* **Severity**: Major
* **Affected Scope**: [`comprehension_report.md` #2](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/comprehension_report.md#L19-L25), [`critic_report.md` Major 5](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/critic_report.md#L53-L58), [`zoom_out_report.md` Finding #2](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/zoom_out_report.md#L20-L27), [`docs/designs/browser-execution-subsystem.md` §6](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L228-L316)
* **Simplicity Rationale**:
  - Stage 6 #2 and Stage 7 Major 5 require defining `PendingApproval`, `ConstraintEvent`, and `VerificationReport` in Section 6 and typing `SubgoalResult.pending_approval` and `SubgoalResult.constraint_events` with them.
  - When adding these definitions, apply Stage 5 Finding #2 so Section 6 does not balloon with dead fields:
    1. Keep `PendingApproval`, `ConstraintEvent`, and `VerificationReport` compact (4–6 fields each).
    2. Remove unpopulated YAGNI fields from `SubgoalRequest` and `SubgoalResult`: drop `expected_result`, `extracted_data`, and `claimed_evidence` (which nothing in `jev_ultrafast` populates or consumes).
    3. Trim `ReasonCode` to deterministic runtime causes rather than speculative model-psychology guesses (`AUTH_WALL_DETECTED`, `MISSING_INPUT_VALUE`), placing the specific gate category (`credentials`, `destructive`, `payment`, `submit_in_read_only`, `external_navigation`, `send`) on `PendingApproval.consequence_classes`.

### 5. Preserve Stage 5 Structural Simplifications During Revision (Single Loop in `agent.py`, Anti-Weeds in `1..379`, and Module Consolidation)
* **Severity**: Major
* **Affected Scope**: [`zoom_out_report.md` Findings #3, #5, #6](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/zoom_out_report.md#L28-L58), [`docs/designs/browser-execution-subsystem.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md)
* **Simplicity Rationale**:
  1. **No Parallel Loop in `agent.py` (Stage 5 Finding #3)**: Do not implement a separate `SubgoalRunner` loop alongside `Agent`'s existing `command("predict" | "act" | "tick")` loop. Extend `Agent` directly to operate over a leased tab with `inputs`, `constraints`, and `needs_approval` pausing, and make `run_subgoal(request)` a thin façade over `Agent`.
  2. **Keep Low-Level JS/DOM Code Out of `1..379` (Stage 5 Finding #5 + Critic Major 7 & Nit 14)**: When applying Critic Major 7 (composed-parent walk for Shadow DOM hit-testing) and Critic Nit 14 (`scrollHeight` check ordering before `getComputedStyle`), state the high-level behavioral invariant in `Technical Plan §3 & §5` (`1..379`) and place the concrete JS/DOM implementation details in `Detailed Implementation Plan §4 & §5` (`380..549`).
  3. **Avoid Unnecessary Module Fragmentation (Stage 5 Finding #6)**: Keep the module structure tight (e.g., fold `verify_tab` into `policy.py` or `session.py`, or keep `contracts.py` minimal) so `jev_ultrafast` remains small and cohesive.

---

## Summary Disposition Matrix for Revision

| Finding / Recommendation Source | Zoom-Out Verdict | Action Required in Document Revision |
| :--- | :--- | :--- |
| **Stage 5 #1 / Stage 6 #1 / Stage 7 Blocker 1(a,c)**: Remove phantom `browser-harness` daemon edits, `Page.frameRequestedNavigation`, and Layer 5 `Fetch` guard | **Adopt** | Scope Tier 3 enforcement to `jev_ultrafast` (`session.py`, `policy.py`, `browser.py`, `snapshot.js`) using pre-act URL/form checks and post-act URL/child-tab verification (`Page.stopLoading` / target cleanup). |
| **Stage 7 Blocker 1(b)**: Inject `Page.addScriptToEvaluateOnNewDocument` to monkey-patch `window.navigation` and `window.open` | **Reject (YAGNI / Fragile)** | Do not inject persistent page-script monkey-patches into third-party sites; rely on pre-act `href`/`formAction` checks + post-act URL and `Target.getTargets` (`openerId`) inspection. |
| **Stage 7 Blocker 2 & Major 8**: `window.__jevFast.secretNodes` taint tracking, `Browser.fresh()` on `fill`, and screenshot suppression (`None`) on sensitive fields | **Adopt** | Add `secretNodes` `WeakSet` taint tracking in `snapshot.js`/`browser.py`/`verify_tab`, check `Browser.fresh()` on `fill`, record `"[SECRET]"` in `history`, and suppress screenshots when sensitive values are present. |
| **Stage 6 #3 & Stage 7 Blocker 3**: Unify `approval_token` and specify both concrete-action and `NEEDS_APPROVAL` resume branches | **Adopt** | Drop `approval_challenge_id`, persist `paused_page` + `history` in `PendingApproval`, and specify both concrete-action and re-choose resume flows. |
| **Stage 5 #4 & Stage 7 Blocker 4**: Remove `custom_js_predicate` from `verify_tab` | **Adopt** | Remove `custom_js_predicate` everywhere (`§6` and `Detailed Plan §9`); keep `verify_tab` strictly declarative and read-only. |
| **Stage 5 #2, Stage 6 #2 & Stage 7 Major 5**: Define `PendingApproval`, `ConstraintEvent`, `VerificationReport` and prune dead fields/reason codes | **Adopt (Combined)** | Define the three missing dataclasses cleanly in `§6`, remove `expected_result`/`extracted_data`/`claimed_evidence`, and simplify `ReasonCode` to deterministic causes. |
| **Stage 5 #3**: Unify `SubgoalRunner` and `Agent` loop in `agent.py` | **Adopt** | Make `run_subgoal` a thin entrypoint over `Agent` rather than duplicating the step loop. |
| **Stage 5 #5 & Stage 7 Major 6, 7, 9, 10, Nits 11–14**: State diagram completeness, Shadow DOM hit-testing, high-level vs. detailed separation, portable links, and style polish | **Adopt** | Update `stateDiagram-v2`, keep concrete JS snippets in `Detailed Implementation Plan` (`380..549`), explain `TypeSafe` (`Comprehension #5`), align `autocomplete` lists (`Comprehension #4`), and normalize links/formatting. |

# Stage 7 Critic Review Report (`critic_report.md`)

**Target Document**: [`docs/designs/browser-execution-subsystem.md`](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L1-L379) (`FeatureRange: 1..379`)
**Reviewers**: `egm-design`, `egm-structural`, `egm-style`

---

## 🔴 Blockers

### 1. Phantom Daemon Enforcement Layer & Invalid CDP Interception Hooks (`egm-design`)
* **Affected Scope**: [Lines 79–82](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L79-L82), [lines 106–111](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L106-L111), [lines 155–157](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L155-L157), [lines 205–209](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L205-L209), [line 327](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L327)
* **Finding**:
  1. **Contradiction with `browser-harness` Package Boundary & IPC Architecture**: CUJ 3 (lines 79–82), the Architecture Diagram (line 108), and Policy Layers 1 & 5 (lines 205, 209) assert that the `browser-harness` daemon intercepts `Page.frameRequestedNavigation`, `Target.targetCreated`, and `Network`/`Fetch` subresource requests to deterministically block disallowed navigations and abort network requests, and that leased tabs disable `TAB_MARKER_JS` inside the daemon (line 156). However, `browser-harness` (`0.1.13`) is an external package dependency whose daemon is never modified by this design. Furthermore, Alternative 1 (line 327) explicitly acknowledges that intercepting `Fetch.requestPaused` over `browser-harness`'s synchronous IPC socket and 500-event ring buffer risks deadlocks and dropped requests.
  2. **Invalid CDP Primitive (`Page.frameRequestedNavigation`)**: In Chrome DevTools Protocol, `Page.frameRequestedNavigation` is a passive notification event, not a synchronous interception gate—it cannot block a navigation.
  3. **Post-Hoc Navigation Check Leaks Cross-Domain Requests**: If domain checking only happens in Python before clicking static `href` attributes and *after* `Browser.act()` completes, any JS-triggered redirect (`window.location.assign(...)` or `window.open`) will already have dispatched network requests before Python inspects the post-action URL.
* **Recommendation**:
  - Align Sections 1, 2, 4, and CUJ 3 with the actual execution architecture in `jev_ultrafast`:
    1. Scope Tier 3 enforcement to `jev_ultrafast` (`session.py`, `policy.py`, `browser.py`, `snapshot.js`) operating over `browser-harness`'s `cdp()` channel without claiming modifications to `browser_harness/daemon.py`.
    2. Replace the invalid `Page.frameRequestedNavigation` and Layer 5 daemon `Fetch` claims with a concrete, deadlock-free mechanism: (a) pre-choice and pre-act validation of resolved `a.href` / `form.action` / `button.formAction` URLs in `snapshot.js` and `policy.py`, (b) a lightweight in-page Navigation API (`window.navigation?.addEventListener('navigate', e => ...)`) + `window.open` guard installed via `Page.addScriptToEvaluateOnNewDocument` on leased tabs that cancels disallowed cross-domain client-side navigations synchronously in-page before network dispatch, and (c) post-step target/URL verification via `Target.getTargets` / `Page.stopLoading` as the fallback safety net.

### 2. Post-Injection Secret Exfiltration via Untainted DOM Value Reads in `snapshot.js` and `verify_tab` (`egm-design`)
* **Affected Scope**: [Lines 189–198](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L189-L198), [lines 303–316](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L303-L316), [lines 366–369](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L366-L369)
* **Finding**:
  1. **Attribute-Only Sensitivity Heuristics Leak Secrets on Subsequent `observe()` Steps**: Section 3 (lines 191–193) defines `isSensitive(e)` solely by static DOM attributes (`e.type === 'password'` or `autocomplete` matching `cc-*`, `current-password`, `new-password`). If a caller passes an opaque `$secret:<id>` for an API key field, token input, custom MFA input, or a password field toggled to `type="text"`, `Browser.act()` injects the plaintext into the DOM via `Input.insertText` (line 197). On the **immediately following `observe()` call**, `snapshot.js` sees `isSensitive(e) === false`, reads `e.value` in plaintext, and serializes the raw secret into `state.page.actions`, `pageKey()`, `guards`, `SubgoalResult.distilled_elements`, and the next TypeSafe `/v1/systemone` payload!
  2. **Missing Target Guard Check for `fill` Actions in `Browser.fresh()`**: CUJ 2 (line 74) and Section 2 (line 152) claim that before dereferencing `$secret:<id>`, the harness verifies that `tab_1`'s target guard still matches via `Browser.fresh()`. However, `Browser.fresh(page, action)` currently only checks `page["guards"].get(str(node))` when `action["kind"] in {"click", "select"}`—skipping the per-node guard check for `action["kind"] == "fill"`.
  3. **Unmasked DOM Inspection in `verify_tab`**: `verify_tab` (lines 303–316) evaluates control values and visible text independently of `snapshot.js`, which will read back plaintext secrets from filled inputs unless secret-tainted nodes are masked.
* **Recommendation**:
  - **Node-Level Secret Taint Tracking**: Whenever `Browser.act()` resolves and types a `$secret:<id>` token into `action["node"]`, mark that node as secret-tainted in `window.__jevFast.secretNodes` (a `WeakSet` of DOM Elements). In `snapshot.js` and `verify_tab`, treat `cache.secretNodes?.has(e)` as `sensitive: true` so any element that ever received a `$secret:<id>` value has its value permanently masked as `"[MASKED]"` across all subsequent snapshots, guards, and verification reports.
  - **Enforce Per-Node Guard on `fill`**: Require `Browser.fresh(page, action)` to check `[c.pageKey(), c.guard(c.nodes.get(node))]` for `action["kind"] in {"click", "select", "fill"}`.

### 3. Broken `awaiting_approval` Pause/Resume Flow for `NEEDS_APPROVAL` and Post-Approval Secret Supply (`egm-design`)
* **Affected Scope**: [Lines 65–75](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L65-L75), [lines 151–152](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L151-L152), [lines 207–208](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L207-L208), [lines 241–254](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L241-L254)
* **Finding**:
  1. **No Target/Action Exists When Model Chooses `operation="NEEDS_APPROVAL"`**: When `operation == "NEEDS_APPROVAL"`, no target head (`click_target` or `type_text_target`) is selected. Yet Section 2 (line 152) states that `awaiting_approval` stores `pending_action` and `target_guard`, and resuming with `approval_token` executes that pending action (line 75).
  2. **Late-Bound Secret Inputs on Resume**: In CUJ 2 (lines 67–74), Jev encounters a password prompt when the caller has *not yet* supplied `inputs={"password": "$secret:corp_pw"}`. When Antigravity resumes the sub-goal by passing both `approval_token` and the newly provided `inputs={"password": "$secret:corp_pw"}` (line 73), executing a stale `pending_action` whose `input_source` was `"__GENERATE__"` (or `None`) fails instead of binding the newly supplied `$secret:corp_pw`.
  3. **Loss of Step History on Resume**: `TabRecord` / `PendingApproval` must persist the paused `page` snapshot and `history` across an `awaiting_approval -> leased` transition so resumed execution retains prior step context and step budgets.
* **Recommendation**:
  - Distinguish the two resume branches explicitly:
    1. **Concrete Gated Action (`CLICK` / `SELECT` / `TYPE_TEXT` with an already-bound `input:<key>`)**: `PendingApproval` stores `(action, input_source, page_fingerprint, target_guard)`. Resuming with `approval_token` verifies `Browser.fresh(paused_page, action)`, executes that exact action once, and continues the loop with persisted `history`.
    2. **Model-Escalated `NEEDS_APPROVAL` or Missing Credential (`action=None` or unbound `sensitive: true` field)**: Resuming with `approval_token` and updated `inputs` verifies `Browser.fresh(paused_page)`, grants approval for the gated category on that page state, and re-runs `choose()` with the newly supplied `inputs`.

### 4. `custom_js_predicate` in `verify_tab` Violates `AGENTS.md`, Non-Goal 6, and Read-Only `verifying` Guarantees (`egm-design`)
* **Affected Scope**: [Line 45](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L45), [line 153](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L153), [lines 303–316](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L303-L316), [line 369](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L369)
* **Finding**:
  - `AGENTS.md` and Non-Goal 6 strictly forbid letting models emit selectors or executable JavaScript, and Section 2 states that in `verifying` state all mutating `Runtime.evaluate` calls are blocked. Yet `verify_tab` (line 312) exposes `custom_js_predicate: str | None = None` to Tier 1 callers, creating an arbitrary JS execution surface that can mutate DOM state during verification.
* **Recommendation**:
  - Remove `custom_js_predicate` from `verify_tab`. Keep verification purely declarative (`url_matches`, `title_matches`, `required_text`, `forbidden_text`, `control_values`) evaluated by code-owned read-only inspection logic.

---

## 🟡 Majors

### 5. Incomplete Schema Definitions & Type Mismatch in `### 6. Subsystem API & Data Schemas` (`egm-structural`)
* **Affected Scope**: [Lines 258–316](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L258-L316)
* **Finding**:
  - `verify_tab` returns `VerificationReport` (line 314), and `contracts.py` (line 391) references `PendingApproval`, `ConstraintEvent`, and `VerificationReport` dataclasses, but none of those three dataclasses are defined in Section 6. Additionally, `SubgoalResult` (lines 294–295) types `constraint_events` and `pending_approval` as untyped `dict[str, Any]` instead of `ConstraintEvent` and `PendingApproval`.
* **Recommendation**: Define `PendingApproval`, `ConstraintEvent`, and `VerificationReport` explicitly in Section 6 and reference them in `SubgoalResult`.

### 6. Incomplete `TabRegistry` State Machine: `verifying` Transitions, Child-Tab Lease Handoff, and Lease TTL Enforcement (`egm-design`, `egm-structural`)
* **Affected Scope**: [Lines 130–158](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L130-L158), [lines 283–285](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L283-L285), [lines 303–316](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L303-L316)
* **Finding**:
  1. The `stateDiagram-v2` (lines 133–148) omits `verifying --> verifying : acquire_lease(verifier, mode="read_only")` (needed by CUJ 1 when a sub-goal ends in `verifying` and `verify_tab` attaches), omits `verifying --> leased` (for recovery sub-goals when verification fails), and omits `close_tab()` from `verifying` and `awaiting_approval`.
  2. Line 157 does not specify releasing the parent tab's mutation lease (`leased -> available`) when switching active execution to a child tab under `follow_new_tabs="auto"`.
  3. Lease validation (`assert_active_lease`) must be checked immediately prior to every mutating call in `Browser.act()` to prevent expired leases from mutating state.
* **Recommendation**: Update the `stateDiagram-v2` and lifecycle invariants in Section 2 to include these transitions, parent-lease release on child handoff, and pre-act lease validation.

### 7. Open Shadow DOM Hit-Testing Fails with `e.contains(document.elementFromPoint(x, y))` (`egm-design`)
* **Affected Scope**: [Lines 217–218](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L217-L218)
* **Finding**:
  - `Node.prototype.contains()` returns `false` across Shadow DOM boundaries. Drilling down via `shadowRoot.elementFromPoint(x, y)` must be paired with walking up the composed ancestor chain (`cur = cur.parentElement || cur.getRootNode()?.host`) to verify `cur === e`.
* **Recommendation**: Specify shadow-piercing deepest hit-testing and composed-parent ancestor traversal.

### 8. Screenshot Masking Side Effects & History/Trace Secret Leakage (`egm-design`)
* **Affected Scope**: [Lines 195–198](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L195-L198), [lines 366–368](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L366-L368)
* **Finding**:
  - Mutating DOM styles to mask a screenshot during `observe()` risks invalidating `pageKey()` / `marker`, and recording raw `"$secret:<id>"` key names inside `history["text"]` exposes internal secret store identifiers to external LLMs (`recent_actions`).
* **Recommendation**: Record `"[SECRET]"` in `history["text"]` / `recent_actions` (while recording `input_source="input:<key>"` in `action_trace`), and suppress screenshot capture (`screenshot_b64 = None`) whenever a secret-tainted or non-password `sensitive: true` field with a non-empty value is visible.

### 9. Non-Portable Machine-Local `file:///c:/Users/Albe/...` Links (`egm-structural`)
* **Affected Scope**: Lines 11, 20, 22, 23, 25, 162, 187, 214, 216, 220, 332, 335, 340, 342, 356, 374
* **Finding**: Section `1..379` uses machine-local `file:///c:/Users/Albe/...` URIs, whereas `380..549` uses portable relative repository paths.
* **Recommendation**: Standardize all repository links across `docs/designs/browser-execution-subsystem.md` to portable relative paths (`../../jev_ultrafast/...` or `./jev_ultrafast/...`).

### 10. Subjective Quality Judgment in Alternative 1 (`egm-style`)
* **Affected Scope**: [Line 326](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L326)
* **Finding**: Calling target systems `"poorly designed web applications"` is subjective.
* **Recommendation**: Replace with `"Conversely, web applications can trigger consequential state mutations via GET links or client-side event handlers."`

---

## 🟢 Nits

11. **Internal Review-Process Jargon (`egm-style`)**: Remove `"& EGM Goldfish Review Panel"` (line 3) and `"from brainstorming; all architectural boundaries and Goldfish objections..."` (line 50).
12. **Promotional / Vague Wording (`egm-style`)**: Replace `"Deliver Evidence-Rich Results"` (line 35) with `"Return Structured Execution Telemetry"`, `"with full diagnostic telemetry"` (line 82) with `"containing the recorded constraint_events and diagnostic_message"`, `"only ever record"` (line 196) with `"only record"`, and standardize on `"Backward Compatibility"` / `"backward-compatible"` (lines 36, 373–374).
13. **Formatting & Diagram Polish (`egm-structural`)**: Remove the redundant horizontal rule before `## Non-Functional Requirements` (line 362), normalize nested unordered list markers to `*   `, and replace literal `\n` in Mermaid flowchart node labels (lines 96–110) with `<br/>` or single-line labels.
14. **Scroll Container Style-Check Ordering & `TYPE_TEXT` Label (`egm-design`)**: Check `el.scrollHeight > el.clientHeight + 20` before calling `getComputedStyle(el).overflowY` in `snapshot.js` (line 214), and update the `TYPE_TEXT` operation description in `model.py` when caller `inputs` are present (line 180).

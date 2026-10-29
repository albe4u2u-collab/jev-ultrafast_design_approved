**What this proposes**
This design upgrades `jev-ultrafast` and `browser-harness` from a single-goal demo into a three-tier browser execution subsystem for Antigravity, where Antigravity handles high-level planning, approvals, and verification; `jev-ultrafast` executes bounded tactical UI sub-goals (3–15 steps); and `browser-harness` + `jev_ultrafast/session.py` manage tab state and safety boundaries. In the feature specification (`1..379`), it introduces a per-tab lifecycle state machine (`available`, `leased`, `awaiting_approval`, `verifying`, `released`, `failed`) with single-mutator lease tokens, an orthogonal `type_text_input` choice head in TypeSafe requests so caller-supplied inputs bypass the secondary text LLM, masked password/sensitive DOM observation paired with lowest-layer `$secret:<id>` dereferencing at `Input.insertText`, a 5-layer consequence-based safety and human-approval gate (`needs_approval`), a read-only verifier API (`verify_tab`), and DOM compatibility upgrades for nested scroll containers, open Shadow DOM, child tabs (`openerId`), and controlled inputs. In the detailed implementation plan (`380..549`), it maps these features onto 13 repository files—creating `contracts.py`, `session.py`, `policy.py`, and `verifier.py`, and modifying `snapshot.js`, `browser.py`, `questions.py`, `model.py`, `agent.py`, `__init__.py`, `test_agent.py`, `check_guards.py`, and docs—while maintaining backward compatibility for the existing `Agent(url, goals)` API and offline test suite.

**How the existing system works, per this document**
Per the document, `jev-ultrafast` currently runs a fast (<200ms decision latency) tactical loop that captures an atomic indexed DOM snapshot via `snapshot.js` and sends a single-round-trip request to `TypeSafe` (`/v1/systemone`) to select an operation (`CLICK`, `TYPE_TEXT`, `SELECT`, `SCROLL_UP`, `SCROLL_DOWN`, `WAIT`, `DONE`, `BLOCKED`) and target element index. Whenever `TYPE_TEXT` is selected, `agent.py` calls a secondary text helper LLM (`field_text`, with a `pending_text` stale-retry cache) to generate the string to type, while `snapshot.js` completely hides `input[type="password"]` elements (`safe = e => !['password','file','hidden'].includes(e.type)`). Both `Browser.__init__` and `Agent.__init__` open a fresh tab for a single natural-language goal and unconditionally close it on exit, while `browser-harness` (`0.1.13`) tracks singleton daemon state (`self.session`, `self.target_id`, `self.dialog`) and injects a `'🐴 '` prefix into `document.title` on load events, which can trigger spurious `StalePage` errors in `snapshot.js`. Finally, `snapshot.js` and `browser.py` currently scroll only the root document at fixed coordinates `(550, 650)`, ignore open Shadow DOM trees and child tabs opened via `target="_blank"`/`window.open()`, and truncate indexed elements past 250 without surfacing truncation telemetry to the model.

---

### Comprehension Findings

#### 1. Architectural vs. Implementation Mismatch on `browser-harness` Daemon Enforcement
*   **Severity**: Major
*   **Affected Scope**: [Section 1 (`Tier 3: Browser Harness`)](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L106-L111), [Section 2 (`Title-Marker Isolation & Child-Tab Tracking`)](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L155-L157), [Section 4 (`Consequence-Based Policy & Safety Engine`, Layers 1 & 5)](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L205-L209), and [Detailed Implementation Plan (`Target Files to Create/Modify`)](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L382-L531)
*   **Discrepancy Description & Rationale**:
    *   In `FeatureRange` (`1..379`), the architecture states that the `browser-harness` daemon disables `TAB_MARKER_JS` (`'🐴 '` title prefix injection) on leased tabs (line 156), listens for `Page.frameRequestedNavigation` and `Target.targetCreated` events (lines 79–82, 157, 205), and enforces a secondary `Network` / `Fetch` guard inside the daemon (lines 108, 209).
    *   In `DetailedRange` (`380..549`), all 13 target files belong to `jev-ultrafast`; no `browser-harness` files are modified. Instead, `jev-ultrafast` strips `'🐴 '` in `snapshot.js` (line 423), polls child targets via `poll_child_tabs(tab_id)` in `jev_ultrafast/session.py` (line 406), and checks `check_domain_allowed` before/after clicks in `jev_ultrafast/browser.py` (line 439), with no mention of the Layer 5 `Network` / `Fetch` guard.
    *   A new reader cannot tell whether `browser-harness` is an external read-only dependency (making `session.py` and `browser.py` the actual enforcement points) or whether `browser-harness` daemon modifications are missing from the implementation plan.

#### 2. Undefined Schema Definitions and Type Contradiction for `PendingApproval`, `ConstraintEvent`, and `VerificationReport`
*   **Severity**: Major
*   **Affected Scope**: [Section 6 (`Subsystem API & Data Schemas`)](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L258-L316) vs. [Detailed Implementation Plan (`1. ./jev_ultrafast/contracts.py`)](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L384-L393)
*   **Discrepancy Description & Rationale**:
    *   Line 391 states that `contracts.py` defines output dataclasses `PendingApproval`, `ConstraintEvent`, `SubgoalResult`, and `VerificationReport`, and line 314 shows `verify_tab(...) -> VerificationReport`.
    *   However, the fields of `PendingApproval`, `ConstraintEvent`, and `VerificationReport` are never specified in Section 6 or Section 1, and `SubgoalResult` in Section 6 (lines 294–295) types those fields as untyped dicts (`constraint_events: list[dict[str, Any]]` and `pending_approval: dict[str, Any] | None`) rather than `list[ConstraintEvent]` and `PendingApproval | None`.

#### 3. Ambiguous Relationship Between `approval_challenge_id` and `approval_token`
*   **Severity**: Nit
*   **Affected Scope**: [Section 2 (`Paused Approval State`)](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L152) vs. [Detailed Implementation Plan (`2. ./jev_ultrafast/session.py`)](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L403-L404)
*   **Discrepancy Description & Rationale**:
    *   Line 152 says that upon entering `awaiting_approval`, the tab record stores a cryptographic `approval_challenge_id`, and cannot mutate until the caller supplies a matching `approval_token`.
    *   Line 403 says `pause_for_approval(tab_id, lease_token, pending_approval) -> str` directly mints a one-time `approval_token` and line 404 validates that `approval_token` in `resume_with_approval`. It is unclear whether `approval_challenge_id` and `approval_token` are two names for the same token or two steps in a challenge-response handshake.

#### 4. Minor Inconsistency in Sensitive `autocomplete` Values Between Sections
*   **Severity**: Nit
*   **Affected Scope**: [Section 3 (`Masked Sensitive Fields`)](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L192) vs. [Detailed Implementation Plan (`4. ./jev_ultrafast/snapshot.js`)](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L424)
*   **Discrepancy Description & Rationale**:
    *   Line 192 specifies `isSensitive` as checking `e.type === 'password'`, `cc-*`, `current-password`, and `new-password`.
    *   Line 424 adds `one-time-code` to that list (`current-password`, `new-password`, `one-time-code`, or `cc-*`). The two specifications should list the same set of `autocomplete` values.

#### 5. Unexplained Project-Specific Service: `TypeSafe` (`/v1/systemone`)
*   **Severity**: Nit
*   **Affected Scope**: [Overview](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L15) and [Section 3 (`Orthogonal TYPE_TEXT Heads`)](file:///c:/Users/Albe/Documents/CLI-Project/jev-ultrafast/docs/designs/browser-execution-subsystem.md#L161-L188)
*   **Discrepancy Description & Rationale**:
    *   The document refers to "TypeSafe" and `POST /v1/systemone` throughout (lines 15, 20, 117, 165, 335, 460) and mentions `questions` and `criteria` dicts without a one-sentence inline definition or link explaining that TypeSafe is an external multi-head constrained-choice inference API that returns probability distributions over discrete options for each question key in a single HTTP call.

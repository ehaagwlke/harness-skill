# Harness — Gate Rules

Quick reference. Full gate definitions are in SKILL.md.

## Gate table

| Gate | Name | Type | Trigger | Block condition |
|------|------|------|---------|-----------------|
| 0 | CAPTURE | Auto | Every request | Never blocked — always runs |
| 1 | PLAN | Soft | Every request | Must log before executing |
| 2 | CLARIFY | Conditional | Ambiguity / destructive action | Ask before proceeding |
| 3 | EXECUTE | Auto | After plan confirmed | All file edits delegated to DeepSeek; Claude verifies output |
| 4 | SMOKE | Conditional | Any code/config changed | Must pass before Gate 5 |
| 5 | CONFIRM | **Hard** | End of every work unit | User must explicitly accept |
| 6 | CLOSE | Auto | After acceptance | Final VCS ops happen here |

---

## Hard gates

Only Gate 5 is a hard gate. The loop **must** stop and wait for explicit user input. All other gates proceed autonomously subject to block conditions.

---

## Gate 4 — three-step logic

```
code or config changed?
  ├─ NO  → skip Gate 4
  └─ YES → Step A: does project require deployment to test?
               ├─ YES → smoke commit + AUTO-PUSH (no ask) → deploy → Step B
               └─ NO  → Step B directly
             Step B: run designated smoke test
               ├─ PASS → Step C (if UI changed) or Gate 5
               └─ FAIL → fix → re-smoke
                            ├─ PASS → Step C (if UI changed) or Gate 5
                            └─ FAIL (unresolvable) → surface to user, ask how to proceed
                                                      DO NOT proceed to Gate 5
             Step C: frontend-delivery-verify (UI/frontend changes only)
               ├─ PASS → Gate 5
               └─ FAIL → fix → back to Step B
```

The smoke-commit push is part of Gate 4 mechanics — the remote *is* the test environment, so getting the commit there is what enables the gate to run. It does not require a separate user ask. Final-commit push at Gate 6 is a different operation and still requires explicit user ask.

---

## Two commit types — the distinction

| Type | When | Purpose | Gate |
|------|------|---------|------|
| **Smoke commit** | Gate 4 Step A | Gets code to the test environment; pre-acceptance checkpoint | Allowed before Gate 5 |
| **Final commit** | Gate 6 | Canonical record of accepted, delivered work | Only after Gate 5 |

A smoke commit may carry a WIP label. If Gate 5 rejects the work, amend it or layer fix commits on top. Clean up the message at Gate 6 if needed.

---

## Forbidden before Gate 5

These actions claim work is done — they are only allowed after explicit user acceptance:

- A commit whose message presents the work as complete/delivered (no WIP label)
- Any message to an external party about the work being done
- Any write to a production database representing the delivered state

These actions are **not** forbidden before Gate 5 (they are required for smoke testing) and run **without a separate user ask**:

- Smoke commit (`git commit`) to enable deployment-based testing
- `git push` of a smoke commit to reach the test environment (auto-push)
- Deployment to a **non-production** test/staging environment

---

## Soft vs hard clarification

| Situation | Action |
|-----------|--------|
| Minor ambiguity, safe default exists | Use the default, note it in session log |
| Ambiguity affects acceptance criteria | Ask (Gate 2) |
| Destructive / irreversible action | Always ask, even if intent seems clear |
| Deliverable is a standalone HTML file | Ask language(s) (default: EN+ZH) and theme (default: both, follows OS) |

---

## Design-principle checks (Gate 5, UI work)

Any task that produces or modifies user-visible UI must verify before requesting acceptance:

| Principle | Quick check |
|-----------|-------------|
| i18n | All visible strings go through the translation layer; `zh-CN` / `en-US` / `ar-AR` keys exist |
| Theme | Light + dark both render correctly; defaults follow OS setting |
| Mobile | Layout verified at mobile breakpoint |
| Undo | Reversible actions have an undo affordance |
| Progress | Any operation > 300 ms shows progress/status |

Missing any of the above → loop back to Gate 3, not forward to Gate 5.

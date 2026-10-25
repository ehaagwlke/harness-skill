---
name: harness
version: 1.0.0
description: "Structured execution harness that wraps any coding or design task in a disciplined gate-driven loop. Always on. Use for any task that changes code, config, content, or infrastructure."
metadata:
  requires:
    bins: []
---

# Harness

A gate-driven loop that ensures nothing ships without being captured, planned, tested, and explicitly accepted by the user.

See [WORKFLOW.md](WORKFLOW.md) for detailed guidance and [GATES.md](GATES.md) for the gate rule table.

---

## Non-negotiable principles

These apply on every relevant task — Gate 1 (PLAN) must explicitly account for them when they're in scope, and Gate 5 (CONFIRM) must verify them before acceptance.

### Design
- **Easy to use, easy to understand, visually appealing, straightforward.**
- **i18n from day one** — every user-facing string goes through the project's translation layer. Default languages: `zh-CN`, `en-US`, `ar-AR` (Arabic kept specifically for RTL UI verification).
- **Light + dark theme from day one** — follow OS setting by default.
- **Mobile-friendly by default** — verify layouts on mobile breakpoints before Gate 5.
- **Undo for anything reversible** — any action that *can* be undone *must* be easy to undo.
- **Progress feedback for anything > 300 ms** — show progress/status; never leave the user wondering.

### HTML deliverables

When the task produces a **standalone HTML file**, trigger Gate 2 (CLARIFY) to confirm — unless the user has already specified both:

1. **Languages** — which language(s) should the content use? Default: `EN` + `ZH` (English and Simplified Chinese, side-by-side or toggled).
2. **Theme** — dark only, light only, or both? Default: **both**, following OS `prefers-color-scheme` via a CSS media query.

Apply these as non-negotiable defaults when the user accepts or skips the ask:
- `@media (prefers-color-scheme: light)` block overrides all color variables for light mode.
- All body text provided in both `EN` and `ZH` unless the user explicitly restricts to one language.
- Background textures (if any) must not interfere with content readability — prefer dot grids (`radial-gradient` 1 px dots) over line grids; always test in both themes.

---

### Development
- **Everything under git.** No untracked work.
- **Don't change working code unless necessary** — if a file already works and is out of scope, leave it. Drive-by edits are forbidden.
- **Maintain a release-style changelog** — markdown for AI agents (`CHANGELOG.md`), HTML for end users (project-specified location). Update at Gate 6, not before.
- **Smoke commits push automatically** — server-deployed projects need the commit on the remote to test; pushing the WIP smoke commit is part of Gate 4 Step A, no separate ask needed.
- **Final commits do not push automatically** — Gate 6 produces the canonical commit; pushing it requires explicit user ask.
- **Remote = source of truth for deploy** — pull latest on the remote and rebuild after every deploy.

## Loop

```
CAPTURE → PLAN → [CLARIFY?] → EXECUTE → [SMOKE COMMIT →] SMOKE → CONFIRM → [REDO?] → CLOSE
```

---

### Gate 0 — CAPTURE
Append the request to the project session log (default: `docs/SESSION.md`; create `docs/` if absent).

```markdown
## [YYYY-MM-DD HH:MM:SS] Request
<request, verbatim or faithfully paraphrased>
```

Acknowledge: _"Got it. Decomposing…"_

---

### Gate 1 — PLAN
Decompose into the smallest independently executable actions. Log the full list **before touching any file**:

```markdown
## Plan
- [ ] Action 1
- [ ] Action 2
```

Show the plan. Incorporate feedback if the user responds. Then proceed — this is a soft gate.

---

### Gate 2 — CLARIFY (conditional)
Trigger if **any** of these are true:
- Scope or acceptance criteria is ambiguous
- Two or more actions depend on an unconfirmed design decision
- A destructive or irreversible action is in the plan

Ask **one focused question per ambiguity**, max 3 total. Never ask about things inferable from context. Once answered, proceed.

---

### Gate 3 — EXECUTE

**Role split — mandatory:**

| Who | Does what |
|-----|-----------|
| **Claude** | Reads files for context, writes the DeepSeek delegation prompt, calls `mcp__deepseek__delegate_to_deepseek`, verifies output, handles SSH/deploy/git ops |
| **DeepSeek** | All file edits and writes (Edit, Write, NotebookEdit tool calls) |

**DeepSeek delegation is required for every file edit.** Claude must not call Edit or Write directly unless DeepSeek is unavailable (tool error / timeout) — in that case, log the fallback reason in SESSION.md and proceed with direct edits.

**Delegation prompt requirements:**
- Exact file paths and line numbers for each change
- The specific old/new strings or full replacement content — not vague descriptions
- Project conventions from CLAUDE.md that affect the change (naming, i18n, CSS patterns)
- Success criteria: what the output should look like when done

**After DeepSeek returns:** Claude must sample-read at least one modified file to verify the change is correct before advancing to Gate 4. If the output is wrong, correct it with a targeted re-delegation (or direct fix as a last resort).

Update the session log live:

```markdown
- [x] Action 1  ✓ (DeepSeek)
- [ ] Action 2  ← in progress
```

Brief status lines to the user. No walls of text.

---

### Gate 4 — SMOKE (any code or config changed)

**Step A — Smoke commit + auto-push (only if the project requires deployment to test)**

Some projects cannot be tested without first deploying to a target environment. For those, commit **and push** the changes before running the smoke test. This commit is a **pre-acceptance checkpoint** — it enables testing; it does not claim the work is done or accepted.

Label it clearly so it cannot be mistaken for a final commit:
```bash
git commit -m "wip: smoke — <short description>"
git push           # auto-push is part of Gate 4 Step A — no separate ask required
```

The push is automatic here because the remote is the test environment — without it, Gate 4 Step B (deploy + smoke) cannot run. Final commits at Gate 6 are different: they still require explicit user ask before pushing.

If Gate 5 rejects the work, layer a fix commit on top (don't amend a pushed commit). The final commit message is cleaned up at Gate 6 if needed.

**Step B — Run the smoke test**

Use the project's designated smoke test (defined in the project's CLAUDE.md). If none is defined, apply this priority order:

1. Existing test suite (`npm test`, `pytest`, `go test ./...`, etc.)
2. Type-check / lint (`tsc --noEmit`, `ruff check`, etc.)
3. Direct invocation of the changed entry point
4. Manual verification — describe exactly what was checked and what the result was

Log the result:
```markdown
## Smoke test
Command: `<command>`
Result: PASS  (or FAIL — details below)
```

**If FAIL:** fix, re-smoke, log the new result. Do not proceed to Gate 5 with a failing smoke. If the failure cannot be resolved, surface it to the user and ask how to proceed — never silently proceed.

**Step C — Frontend delivery verification (UI changes only)**

When any task produces or modifies user-visible UI, run the `frontend-delivery-verify` skill after the smoke test passes:

1. Open the changed page in a browser (dev server or deployed smoke environment).
2. Interact with the change directly — click, input, toggle; screenshot before/after.
3. Browser console: zero new errors or warnings.
4. Chrome DevTools performance trace + Core Web Vitals audit — no regressions.

All four steps must pass before advancing to Gate 5. A failing frontend verify is treated the same as a failing smoke: fix and re-run Step B + Step C.

---

### Gate 5 — CONFIRM ← hard gate
Present the deliverable. Summarise what was done.

For UI/UX work, verify the non-negotiable design principles before presenting:
- i18n strings routed through translation layer (zh-CN / en-US / ar-AR available)
- Light + dark theme both render correctly
- Mobile breakpoints checked
- Reversible actions have an undo path
- Any > 300 ms operation shows progress/status

Ask explicitly:
> "Does this look good? Accept, or tell me what to change."

Wait for the user's response. Do **not** auto-proceed.

- **Accepted** → Gate 6
- **Changes requested** → log the feedback, build a delta plan (Gate 1), execute the delta, re-smoke, re-confirm

---

### Gate 6 — CLOSE
Mark the outcome in the session log:

```markdown
## Outcome
Status: ACCEPTED
Accepted at: [timestamp]
Commit: <hash or message>
```

**Changelog (when user-visible behavior changed):**
- Append a release-style entry to `CHANGELOG.md` (markdown, for AI agents).
- If the project maintains a human-facing HTML changelog, update it too (location per project CONTEXT/CLAUDE.md).
- Skip for pure refactors, internal docs, or session-log-only changes.

**VCS:**
- The smoke commit has already been pushed (Gate 4 Step A). Layer a final cleanup commit on top — do **not** amend a pushed commit.
- **Push the final commit only when the user explicitly asks.** Smoke-commit auto-push does not extend to the final commit.

---

## Rules

| Rule | Detail |
|------|--------|
| **DeepSeek edits, Claude plans/reviews** | All file edits at Gate 3 go through `mcp__deepseek__delegate_to_deepseek`. Claude reads, plans, delegates, verifies, and handles SSH/git/deploy ops. |
| **Log before doing** | Plan is written before any file is touched |
| **One question at a time** | Never ask compound questions |
| **Smoke before confirm** | All code changes must pass smoke before Gate 5 |
| **Two commit types** | Smoke commit (pre-acceptance, enables testing) vs final commit (post-acceptance, canonical record). Both are allowed; only the final commit marks work as delivered. |
| **No final commit before Gate 5** | A smoke commit is allowed and expected for server-deployed projects; a commit that claims work is done is not |
| **Smoke push is automatic, final push is not** | Gate 4 Step A pushes the WIP smoke commit without asking (it's how the test environment gets the code). Gate 6's final commit still requires explicit user ask to push. |
| **Design principles are gate criteria** | i18n, light/dark, mobile, undo, > 300 ms progress — checked at Gate 5 for any UI work |
| **Don't change working code unless necessary** | Drive-by edits to working, out-of-scope code are forbidden |
| **Changelog at Gate 6** | User-visible changes update `CHANGELOG.md` (and HTML changelog if the project has one) |
| **Redo is a mini-loop** | Rejected work restarts from Gate 1 for the delta only |
| **Session log is the truth** | All state lives in the project session log; it survives context resets |

---

## Session log location

Default: `docs/SESSION.md` (create `docs/` if absent, append if the file exists).

A project may specify a different location in its own `CLAUDE.md`. Always use the project-specified location when one exists.

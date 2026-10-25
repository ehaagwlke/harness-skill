# Harness — Workflow Reference

Detailed phase guidance, decision trees, and edge cases.

## Phase decision tree

```
User input received
        │
        ▼
   Gate 0: CAPTURE
   Write request to SESSION.md
        │
        ▼
   Gate 1: PLAN
   Decompose → log actions → show to user
        │
        ├── User gives feedback → revise plan
        │
        ▼
   Gate 2: CLARIFY?
   ┌─ Ambiguous scope?         ─┐
   ├─ Unconfirmed design dep?  ─┤─ YES → ask (max 3 Qs) → loop back
   └─ Destructive action?      ─┘
                │ NO
                ▼
   Gate 3: EXECUTE
   Work through actions, update SESSION.md
        │
        ▼
   Gate 4: SMOKE? (code or config changed)
        │
        ├── Requires deployment to test?
        │       ├── YES → smoke commit + AUTO-PUSH (no ask) → deploy → Step B
        │       └── NO  → Step B directly
        │
        Step B: run smoke test
        ├── PASS → Gate 5
        └── FAIL → fix → re-smoke
                    ├── PASS → Gate 5
                    └── FAIL (unresolvable) → surface to user, ask how to proceed
        │
        ▼
   Gate 5: CONFIRM  ← hard stop
        ├── ACCEPTED → Gate 6
        └── CHANGES  → log delta → back to Gate 1 (delta only)
        │
        ▼
   Gate 6: CLOSE
   Mark done in SESSION.md
   Update CHANGELOG.md (and HTML changelog) if user-visible
   VCS: final commit (no push unless asked)
```

---

## Two commit types

| Commit | When | Purpose |
|--------|------|---------|
| **Smoke commit** | Gate 4 Step A | Gets code to test environment; pre-acceptance checkpoint. May carry `wip:` label. |
| **Final commit** | Gate 6 | Canonical record of accepted, delivered work. Clean message, no WIP label. |

The hard gate at Gate 5 controls **claiming work is done** — not whether any commit exists. A smoke commit is required, not forbidden, when the project needs deployment to run smoke tests.

---

## Handling different work types

### Server-deployed projects
The smoke test requires code to be running in the target environment. Workflow:
1. Stage and smoke-commit all changes (labeled `wip: smoke — <description>`)
2. **Auto-push** to the branch the server tracks — no user ask required; this is part of Gate 4 mechanics
3. Pull/build/restart on the server (build always runs on remote; never locally for server-deployed projects)
4. Run the smoke check (build passes, process manager shows `online`, spot-check the feature)
5. If Gate 5 accepts: add a cleanup commit at Gate 6 (do not amend a pushed commit). The Gate 6 push still requires explicit user ask.

### Design work
- Smoke test → a **visual check description**: state what you verified (e.g. "component renders without errors, responsive breakpoints checked")
- Deliverable is the artifact (HTML, mockup, doc, etc.)
- Must satisfy the non-negotiable design principles (see SKILL.md):
  - i18n routing (`zh-CN` / `en-US` / `ar-AR`)
  - Light + dark theme, system default
  - Mobile-friendly layout
  - Undo for reversible actions
  - Progress indicator for any > 300 ms operation
- Verify each at Gate 5 before requesting acceptance; missing any → back to Gate 3.

### Coding work (local test capable)
- Smoke is mandatory (see Gate 4 in SKILL.md)
- If the repo has a CI config, list which checks would run and state whether you expect them to pass

### Refactor / cleanup
- Smoke = existing test suite must still pass green
- If no tests exist, note this explicitly and ask the user if a smoke description is acceptable

### Content / copy / docs
- No smoke step
- Confirm step must show the final text, not a summary

---

## Clarify gate — what counts as ambiguous?

**Ambiguous** (ask):
- "Make it look better" — better than what? by what measure?
- "Clean up the code" — what scope? what style guide?
- "Add a feature" — what's the acceptance criterion?

**Not ambiguous** (infer, don't ask):
- Tech stack already visible in the repo
- Style conventions already in use
- Whether to write tests — yes, always, per project conventions

**Destructive actions** (always confirm explicitly before executing):
- `rm -rf` / bulk delete
- Overwriting a file with no backup
- Database migrations that drop columns
- Publishing / deploying to production

---

## Redo loop

When Gate 5 returns changes:

1. Log the feedback verbatim in SESSION.md under `## Feedback [N]`
2. Build a delta plan — only the changes needed, not the full original plan
3. Execute the delta
4. Re-smoke if code was touched
5. Re-confirm

Each redo cycle increments N. If N > 3 and you're still not accepted, pause
and ask: _"We've iterated 3 times. Can you show me an example or describe
exactly what accepted looks like?"_

---

## VCS rules

```
Gate 5 not passed → final commit BLOCKED (smoke commit allowed and required for server-deployed projects)
Gate 5 passed     → git add -p, final commit allowed
Smoke commit      → AUTO-PUSH (Gate 4 Step A mechanic; no user ask required)
Final commit push → ONLY on explicit user ask
```

A smoke commit that enables testing is **not** the same as a final commit that claims work is done.
Only the latter is blocked before Gate 5. Push behavior is asymmetric on purpose: the smoke commit
has to reach the remote for testing to happen at all, while the final commit is a deliverable the
user controls the publication of.

If you are about to run a *final* commit before Gate 5, stop:
> "I haven't received acceptance yet. Completing Gate 5 first."

Branch strategy: follow whatever the project's CONTEXT.md or CLAUDE.md says.
If nothing is documented, commit to the current branch and note that a PR branch would be safer.

---

## Changelog workflow

User-visible changes must be reflected in a release-style changelog at Gate 6.

| Format | Audience | Where |
|--------|----------|-------|
| Markdown (`CHANGELOG.md`) | AI agents, devs | Project root (or per project CLAUDE.md) |
| HTML | End users | Per project (skip if no human-facing changelog exists yet) |

Entry style:
```markdown
## [unreleased] — YYYY-MM-DD
### Added
- ...
### Changed
- ...
### Fixed
- ...
```

Skip the changelog for: pure refactors with no behavior change, internal docs, session-log-only edits.

---

## SESSION.md full example

```markdown
# Harness Session — YYYY-MM-DD

## [YYYY-MM-DD HH:MM] Request
Add a CSV export button to the Orders table.

## Plan
- [ ] Locate Orders table component
- [ ] Add export button to toolbar
- [ ] Wire onClick to a generateCSV utility
- [ ] Write/update generateCSV utility

## Clarify
Q: Should export respect current filters or always export all rows?
A: Current filters only.

## Execution
- [x] Locate Orders table component  ✓ — src/admin/OrdersTable.tsx
- [x] Add export button to toolbar   ✓
- [x] Wire onClick                   ✓
- [x] generateCSV utility            ✓ — src/utils/csv.ts (new file)

## Smoke test
Command: `npm test -- --testPathPattern=csv`
Result: PASS (3 tests)

## Outcome
Status: ACCEPTED
Accepted at: YYYY-MM-DD HH:MM
Commit: feat(admin): add filtered CSV export to Orders table
```

---
name: story-done
description: "End-of-story completion review. Reads the story file, verifies each acceptance criterion against the implementation, checks for GDD/ADR deviations, prompts code review, updates story status to Complete, and surfaces the next ready story from the sprint."
argument-hint: "[story-file-path]"
user-invocable: true
allowed-tools: Read, Glob, Grep, Bash, Edit
---

# Story Done

This skill closes the loop between design and implementation. Run it at the end
of implementing any story. It ensures every acceptance criterion is verified
before the story is marked done, GDD and ADR deviations are explicitly
documented rather than silently introduced, code review is prompted rather than
forgotten, and the story file reflects actual completion status.

**Output:** Updated story file (Status: Complete) + surfaced next story.

---

## Phase 1: Find the Story

**If a file path is provided** (e.g., `/story-done production/epics/core/story-damage-calculator.md`):
read that file directly.

**If no argument is provided:**

1. Check `production/session-state/active.md` for the currently active story.
2. If not found there, read the most recent file in `production/sprints/` and
   look for stories marked IN PROGRESS.
3. If multiple in-progress stories are found, use `AskUserQuestion`:
   - "Which story are we completing?"
   - Options: list the in-progress story file names.
4. If no story can be found, ask the user to provide the path.

---

## Phase 2: Read the Story

Read the full story file. Extract and hold in context:

- **Story name and ID**
- **GDD Requirement TR-ID(s)** referenced (e.g., `TR-combat-001`)
- **Manifest Version** embedded in the story header (e.g., `2026-03-10`)
- **ADR reference(s)** referenced
- **Acceptance Criteria** — the complete list (every checkbox item)
- **Implementation files** — files listed under "files to create/modify"
- **Engine notes** — any engine-specific constraints noted
- **Definition of Done** — if present, the story-level DoD
- **Estimated vs actual scope** — if an estimate was noted

Also read:
- `docs/architecture/tr-registry.yaml` — look up each TR-ID in the story.
  Read the *current* `requirement` text from the registry entry. This is the
  source of truth for what the GDD required — do not use any requirement text
  that may be quoted inline in the story (it may be stale).
- The referenced GDD section — just the acceptance criteria and key rules, not
  the full document. Use this to cross-check the registry text is still accurate.
- The referenced ADR(s) — just the Decision and Consequences sections
- `docs/architecture/control-manifest.md` header — extract the current
  `Manifest Version:` date (used in Phase 4 staleness check)

---

## Phase 3: Verify Acceptance Criteria

For each acceptance criterion in the story, attempt verification using one of
three methods:

### Automatic verification (run without asking)

- **File existence check**: `Glob` for files the story said would be created.
- **Test pass check**: if a test file path is mentioned, run it via `Bash`.
- **No hardcoded values check**: `Grep` for numeric literals in gameplay code
  paths that should be in config files.
- **No hardcoded strings check**: `Grep` for player-facing strings in `src/`
  that should be in localization files.
- **Dependency check**: if a criterion says "depends on X", check that X exists.

### Manual verification with confirmation (use `AskUserQuestion`)

- Criteria about subjective qualities ("feels responsive", "animations play correctly")
- Criteria about gameplay behaviour ("player takes damage when...", "enemy responds to...")
- Performance criteria ("completes within Xms") — ask if profiled or accept as assumed

Batch up to 4 manual verification questions into a single `AskUserQuestion` call:

```
question: "Does [criterion]?"
options: "Yes — passes", "No — fails", "Not tested yet"
```

### Unverifiable (flag without blocking)

- Criteria that require a full game build to test (end-to-end gameplay scenarios)
- Mark as: `DEFERRED — requires playtest session`

### Test-Criterion Traceability

After completing the pass/fail/deferred check above, map each acceptance
criterion to the test that covers it:

For each acceptance criterion in the story:

1. Ask: is there a test — unit, integration, or confirmed manual playtest — that
   directly verifies this criterion?
   - **Unit test**: check `tests/unit/` for a test file or function name that
     matches the criterion's subject (use `Glob` and `Grep`)
   - **Integration test**: check `tests/integration/` similarly
   - **Manual confirmation**: if the criterion was verified via `AskUserQuestion`
     above with a "Yes — passes" answer, count that as a manual test

2. Produce a traceability table:

```
| Criterion | Test | Status |
|-----------|------|--------|
| AC-1: [criterion text] | tests/unit/test_foo.gd::test_bar | COVERED |
| AC-2: [criterion text] | Manual playtest confirmation | COVERED |
| AC-3: [criterion text] | — | UNTESTED |
```

3. Apply these escalation rules:

   - If **>50% of criteria are UNTESTED**: escalate to **BLOCKING** — test
     coverage is insufficient to confirm the story is actually done. The verdict
     in Phase 6 cannot be COMPLETE until coverage improves.
   - If **some (≤50%) criteria are UNTESTED**: remain ADVISORY — does not block
     completion, but must appear in Completion Notes.
   - If **all criteria are COVERED**: no action needed beyond including the
     table in the report.

4. For any ADVISORY untested criteria, add to the Completion Notes in Phase 7:
   `"Untested criteria: [AC-N list]. Recommend adding tests in a follow-up story."`

---

## Phase 4: Check for Deviations

Compare the implementation against the design documents.

Run these checks automatically:

1. **GDD rules check**: Using the current requirement text from `tr-registry.yaml`
   (looked up by the story's TR-ID), check that the implementation reflects what
   the GDD actually requires now — not what it required when the story was written.
   `Grep` the implemented files for key function names, data structures, or class
   names mentioned in the current GDD section.

2. **Manifest version staleness check**: Compare the `Manifest Version:` date
   embedded in the story header against the `Manifest Version:` date in the
   current `docs/architecture/control-manifest.md` header.
   - If they match → pass silently.
   - If the story's version is older → flag as ADVISORY:
     `ADVISORY: Story was written against manifest v[story-date]; current manifest
     is v[current-date]. New rules may apply. Run /story-readiness to check.`
   - If control-manifest.md does not exist → skip this check.

3. **ADR constraints check**: Read the referenced ADR's Decision section. Check
   for forbidden patterns from `docs/architecture/control-manifest.md` (if it
   exists). `Grep` for patterns explicitly forbidden in the ADR.

4. **Hardcoded values check**: `Grep` the implemented files for numeric literals
   in gameplay logic that should be in data files.

5. **Scope check**: Did the implementation touch files outside the story's stated
   scope? (files not listed in "files to create/modify")

For each deviation found, categorize:

- **BLOCKING** — implementation contradicts the GDD or ADR (must fix before
  marking complete)
- **ADVISORY** — implementation drifts slightly from spec but is functionally
  equivalent (document, user decides)
- **OUT OF SCOPE** — additional files were touched beyond the story's stated
  boundary (flag for awareness — may be valid or scope creep)

---

## Phase 5: Code Review Prompt

After criteria verification and deviation check, use `AskUserQuestion`:

```
question: "Implementation verified. Run /code-review on the changed files?"
options:
  - "Yes — run /code-review now"
  - "Skip — I'll review manually"
  - "Skip — already reviewed"
```

If "Yes": list the files to review and say:
"Run `/code-review [file1] [file2]` to review the implementation before
marking complete."

Do not run code-review inline — surface it and let the developer decide when
to invoke it.

---

## Phase 6: Present the Completion Report

Before updating any files, present the full report:

```markdown
## Story Done: [Story Name]
**Story**: [file path]
**Date**: [today]

### Acceptance Criteria: [X/Y passing]
- [x] [Criterion 1] — auto-verified (test passes)
- [x] [Criterion 2] — confirmed
- [ ] [Criterion 3] — FAILS: [reason]
- [?] [Criterion 4] — DEFERRED: requires playtest

### Test-Criterion Traceability
| Criterion | Test | Status |
|-----------|------|--------|
| AC-1: [text] | [test file::test name] | COVERED |
| AC-2: [text] | Manual confirmation | COVERED |
| AC-3: [text] | — | UNTESTED |

### Deviations
[NONE] OR:
- BLOCKING: [description] — [GDD/ADR reference]
- ADVISORY: [description] — user accepted / flagged for tech debt

### Scope
[All changes within stated scope] OR:
- Extra files touched: [list] — [note whether valid or scope creep]

### Verdict: COMPLETE / COMPLETE WITH NOTES / BLOCKED
```

**Verdict definitions:**
- **COMPLETE**: all criteria pass, no blocking deviations
- **COMPLETE WITH NOTES**: all criteria pass, advisory deviations documented
- **BLOCKED**: failing criteria or blocking deviations must be resolved first

If the verdict is **BLOCKED**: do not proceed to Phase 7. List what must be
fixed. Offer to help fix the blocking items.

---

## Phase 7: Update Story Status

Ask before writing: "May I update the story file to mark it Complete and log
the completion notes?"

If yes, edit the story file:

1. Update the status field: `Status: Complete`
2. Add a `## Completion Notes` section at the bottom:

```markdown
## Completion Notes
**Completed**: [date]
**Criteria**: [X/Y passing] ([any deferred items listed])
**Deviations**: [None] or [list of advisory deviations]
**Code Review**: [Pending / Complete / Skipped]
```

3. If advisory deviations exist, ask: "Should I log these as tech debt in
   `docs/tech-debt-register.md`?"

4. **Update `production/sprint-status.yaml`** (if it exists):
   - Find the entry matching this story's file path or ID
   - Set `status: done` and `completed: [today's date]`
   - Update the top-level `updated` field
   - This is a silent update — no extra approval needed (already approved in step above)

### Session State Update

After updating the story file, silently append to
`production/session-state/active.md`:

    ## Session Extract — /story-done [date]
    - Verdict: [COMPLETE / COMPLETE WITH NOTES / BLOCKED]
    - Story: [story file path] — [story title]
    - Tech debt logged: [N items, or "None"]
    - Next recommended: [next ready story title and path, or "None identified"]

If `active.md` does not exist, create it with this block as the initial content.
Confirm in conversation: "Session state updated."

---

## Phase 8: Surface the Next Story

After completion, help the developer keep momentum:

1. Read the current sprint plan from `production/sprints/`.
2. Find stories that are:
   - Status: READY or NOT STARTED
   - Not blocked by other incomplete stories
   - In the Must Have or Should Have tier

Present:

```
### Next Up
The following stories are ready to pick up:
1. [Story name] — [1-line description] — Est: [X hrs]
2. [Story name] — [1-line description] — Est: [X hrs]

Run `/story-readiness [path]` to confirm a story is implementation-ready
before starting.
```

If no more stories are ready in this sprint:
"No more stories ready in this sprint. Consider running `/sprint-status` to
assess sprint health."

If all Must Have stories are complete:
"All Must Have stories are complete. Consider running `/milestone-review` or
pulling from the Should Have list."

---

## Collaborative Protocol

- **Never mark a story complete without user approval** — Phase 7 requires an
  explicit "yes" before any file is edited.
- **Never auto-fix failing criteria** — report them and ask what to do.
- **Deviations are facts, not judgments** — present them neutrally; the user
  decides if they are acceptable.
- **BLOCKED verdict is advisory** — the user can override and mark complete
  anyway; document the risk explicitly if they do.
- Use `AskUserQuestion` for the code review prompt and for batching manual
  criteria confirmations.

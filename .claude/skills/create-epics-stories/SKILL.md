---
name: create-epics-stories
description: "Translate approved GDDs + architecture + ADRs into implementable epics and stories. Each story embeds the GDD requirement it satisfies, the ADR governing implementation, acceptance criteria, engine compatibility notes, and control manifest rules. Programmers need nothing else to implement a story correctly."
argument-hint: "[system-name | layer: foundation|core|feature|presentation | all]"
user-invocable: true
allowed-tools: Read, Glob, Grep, Write, Task
context: fork
agent: technical-director
---

# Create Epics & Stories

This skill translates the approved design and architecture into implementable
work units. Each **epic** maps to one architectural module. Each **story** is a
single implementable behaviour — small enough for one session, self-contained,
and fully traceable to the GDD requirement it satisfies and the ADR that governs
how to implement it.

**Output:** `production/epics/[epic-slug]/`

**When to run:** After `/architecture-review` passes and `/create-control-manifest`
has been run. All MVP-tier GDDs should be complete and reviewed.

---

## 1. Parse Arguments

**Modes:**
- `/create-epics-stories all` — process all systems in layer order
- `/create-epics-stories layer: foundation` — process Foundation layer only
- `/create-epics-stories [system-name]` — process one specific system
- No argument — ask the user which mode to use

If no argument, use `AskUserQuestion`:
- "What would you like to create epics for?"
  - Options: "All systems (Foundation → Presentation)", "Foundation layer only",
    "Core layer only", "A specific system", "Feature + Presentation layers"

---

## 2. Load All Inputs

### Step 2a — L0: Summary Scan

Before full-reading any documents, Grep all GDDs for their `## Summary` sections:

```
Grep pattern="## Summary" glob="design/gdd/*.md" output_mode="content" -A 5
```

For `all` mode: display a manifest of all Approved/Designed GDDs with their
summaries so the user can confirm scope before the full load begins.

For `layer:` or `[system-name]` modes: filter to only the target GDDs based
on the Summary quick-reference line (Layer + Priority). Skip full-reading
GDDs outside the requested scope entirely.

### Step 2b — L1/L2: Full Document Load

Read everything for the in-scope systems before generating any output:

### Design Documents
- `design/gdd/systems-index.md` — authoritative system list, layers, status
- In-scope GDDs in `design/gdd/` — read every file with "Approved" or "Designed" status
- For each GDD, extract:
  - System name and layer (from systems-index.md)
  - All acceptance criteria (these become story acceptance criteria)
  - All technical requirements (these trace to ADR coverage)
  - Dependencies on other systems

### Architecture Documents
- `docs/architecture/architecture.md` — module ownership and API boundaries
- All Accepted ADRs in `docs/architecture/adr-*.md` — read "GDD Requirements
  Addressed", "Implementation Guidelines", "Engine Compatibility", and
  "ADR Dependencies" sections
- `docs/architecture/control-manifest.md` if it exists — extract layer-specific
  rules to embed in stories; also extract the `Manifest Version:` date from the
  header block (this gets embedded in every story)

### TR Registry
- `docs/architecture/tr-registry.yaml` if it exists — read all active entries,
  indexed by `id`. Used to resolve TR-IDs for each GDD requirement. If the
  registry does not exist, note it once; stories will use `TR-[system]-???`
  placeholders with a warning.

### Engine Reference
- `docs/engine-reference/[engine]/VERSION.md` — engine name + version + risk levels

Report: "Loaded [N] GDDs, [M] ADRs, control manifest: [exists/missing],
manifest version: [date or N/A], TR registry: [N entries or missing],
engine: [name + version]."

---

## 3. Determine Processing Order

Process systems in dependency-safe layer order:
1. **Foundation** (no dependencies)
2. **Core** (depends on Foundation)
3. **Feature** (depends on Core)
4. **Presentation** (depends on Feature + Core)

Within each layer, process systems in the order they appear in systems-index.md.

For each system, determine the corresponding epic:
- Epic = one architectural module from `architecture.md` that implements this system
- If a system doesn't map cleanly to a single module, present options to the user

---

## 4. For Each System: Define the Epic

Present to the user before creating stories:

```
## Epic [N]: [System Name]

**Layer**: [Foundation / Core / Feature / Presentation]
**GDD**: design/gdd/[filename].md
**Architecture Module**: [module name from architecture.md]
**GDD Requirements Traced by ADRs**: [list TR-IDs that have ADR coverage]
**Untraceable Requirements**: [list TR-IDs with no ADR — these need ADRs first]
**Governing ADRs**: [ADR-NNNN, ADR-MMMM, ...]
**Engine Risk**: [LOW/MEDIUM/HIGH — highest risk among governing ADRs]
```

Ask: "Shall I break Epic [N]: [name] into stories?"
- Options: "Yes, break it down", "Skip this epic", "Pause — I need to write ADRs first"

If there are untraced requirements (no ADR covers them), warn:
> "⚠️  [N] requirements in [system] have no ADR. Stories for these cannot embed
> implementation guidance. Consider running `/architecture-decision` first, or
> I can create stories with a placeholder note to add ADR guidance later."

---

## 5. Break Each Epic into Stories

For each epic, decompose the GDD's acceptance criteria into stories:

**Story sizing rules:**
- One story = one implementable behaviour from the GDD
- Small enough to complete in a single focused session (~2-4 hours of focused work)
- Self-contained — no story depends on in-progress work from another story in
  the same epic (stories within an epic run sequentially; they can depend on
  previous stories being DONE, not in-progress)

**Story identification process:**
1. Read every acceptance criterion in the GDD
2. Group related criteria that require the same core implementation
3. Each group = one story
4. Order stories within the epic: foundation behaviour first, edge cases last

For each story, map:
- **GDD requirement**: Which specific acceptance criterion does this satisfy?
- **TR-ID**: Look up the matching entry in `tr-registry.yaml` by normalizing the
  requirement text (lowercase, trimmed). Use the stable ID from the registry.
  If no matching entry exists, note `TR-[system]-???` and warn the user to run
  `/architecture-review` to register the ID before this story is assigned.
- **Governing ADR**: Which ADR tells the programmer how to implement this?
  **Important — ADR status gate**: check the ADR's `Status:` field.
  - If `Status: Accepted` → embed normally.
  - If `Status: Proposed` → set story `Status: Blocked` and note:
    `BLOCKED: ADR-NNNN is still Proposed — story cannot be safely implemented
    until the ADR is Accepted. Run /architecture-decision to advance it.`
  - Do not embed implementation guidance from a Proposed ADR.
- **Engine risk**: Inherited from the ADR's Knowledge Risk field
- **Control manifest rules**: Which layer rules apply?

---

## 6. Generate Story Files

For each story, produce a story file embedding full context:

```markdown
# Story [NNN]: [title]

> **Epic**: [epic name]
> **Status**: Ready
> **Layer**: [Foundation / Core / Feature / Presentation]
> **Manifest Version**: [date from control-manifest.md header — or "N/A" if manifest not yet created]

## Context

**GDD**: `design/gdd/[filename].md`
**Requirement**: `TR-[system]-NNN`
*(Requirement text is stored in `docs/architecture/tr-registry.yaml` — look up by ID at review time)*

**ADR Governing Implementation**: [ADR-NNNN: title]
**ADR Decision Summary**: [1-2 sentence summary of what the ADR decided]

**Engine**: [name + version] | **Risk**: [LOW / MEDIUM / HIGH]
**Engine Notes**: [relevant text from the ADR's Engine Compatibility section,
especially Post-Cutoff APIs Used and Verification Required fields]

**Control Manifest Rules (this layer)**:
- Required: [relevant required pattern from manifest]
- Forbidden: [relevant forbidden approach from manifest]
- Guardrail: [relevant performance guardrail]

---

## Acceptance Criteria

*From GDD `design/gdd/[filename].md`, scoped to this story:*

- [ ] [criterion 1 — directly from GDD, scoped to this story's behaviour]
- [ ] [criterion 2]
- [ ] [performance criterion if applicable]

---

## Implementation Notes

*Derived from ADR-NNNN Implementation Guidelines:*

[Specific guidance for the programmer. This should be the ADR's Implementation
Guidelines section, filtered to what's relevant for this story. Do not
paraphrase in ways that change meaning.]

---

## Out of Scope

*These are handled by neighbouring stories — do not implement here:*

- [Story NNN+1]: [what it handles]
- [Other epic story]: [what it handles]

This boundary prevents scope creep and keeps stories independently reviewable.

---

## Dependencies

- Depends on: [Story NNN-1 must be DONE, or "None"]
- Unlocks: [Story NNN+1, or "None"]
```

---

## 7. Lead-Programmer Architecture Review

Before asking for approval, spawn a `lead-programmer` subagent via Task to review
the drafted stories for the current epic. Provide:
- The epic summary (layer, module, governing ADRs)
- All drafted story files for this epic
- The control manifest (if it exists)

Ask the lead-programmer to check:
- **Story boundaries**: Are any stories too large (should be split) or too small (should be merged)?
- **ADR adherence**: Do the implementation notes in the stories faithfully reflect the ADR decisions? Flag any that drift or contradict.
- **Dependency ordering**: Is the story sequence safe? Can each story be implemented independently once its prerequisite is done?
- **Missing stories**: Are there any infrastructure concerns (interfaces, base classes, test hooks) that need their own story before others can proceed?
- **Control manifest compliance**: Do any stories ask for patterns explicitly forbidden in the manifest?

The lead-programmer produces one of:
- **APPROVED** — proceed to user approval as-is
- **REVISIONS NEEDED** — list of specific changes; revise the affected stories before proceeding
- **BLOCKED** — a fundamental issue (missing ADR, wrong layer, circular dependency) that must be resolved first

If REVISIONS NEEDED: make the changes and re-present the updated stories to the lead-programmer before requesting user approval. Do not ask the user to approve stories with known structural issues.

## 8. Approval Before Writing

For each epic, after stories pass lead-programmer review, present a summary:

```
Epic [N]: [name]
Stories: [N total]
  Story 001: [title] — [1-line summary]
  Story 002: [title] — [1-line summary]
  ...
```

Ask: "May I write these [N] stories and the epic index to
`production/epics/[epic-slug]/`?"

---

## 8. Write Output Files

After approval, write:

### Story files
`production/epics/[epic-slug]/story-[NNN]-[slug].md`

### Epic index
`production/epics/[epic-slug]/EPIC.md`:

```markdown
# Epic: [name]

> **Layer**: [Foundation / Core / Feature / Presentation]
> **GDD**: design/gdd/[filename].md
> **Status**: Ready
> **Stories**: [N total]

## Overview

[1 paragraph describing what this epic implements]

## Stories

| # | Story | Status | ADR |
|---|-------|--------|-----|
| 001 | [title] | Ready | ADR-NNNN |
| 002 | [title] | Ready | ADR-MMMM |

## Definition of Done

This epic is complete when:
- All stories are implemented and reviewed
- All acceptance criteria from [GDD filename] are passing
- No Foundation or Core layer stories have open blockers
```

### Master epics index
`production/epics/index.md` (create or update):

```markdown
# Epics Index

Last Updated: [date]
Engine: [name + version]

| Epic | Layer | System | Stories | Status |
|------|-------|--------|---------|--------|
| [name] | Foundation | [system] | [N] | Ready |
```

---

## 9. Gate-Check Integration

After writing all epics for the requested scope, remind the user:

- **Foundation + Core complete**: Pre-Production → Production gate requires
  Foundation and Core layer epics. Run `/gate-check production` to validate.
- **All layers complete**: Full epic coverage achieved. The sprint plan can now
  reference real story file paths instead of GDD references.

---

## Collaborative Protocol

1. **Load silently** — read all inputs before presenting the first epic
2. **One epic at a time** — present each epic before creating stories; don't
   batch all epics and ask once
3. **Warn on gaps** — flag untraced requirements before proceeding; let the user
   decide whether to pause for ADRs or proceed with placeholders
4. **Ask before writing** — per-epic approval before writing story files
5. **No invention** — all acceptance criteria come from GDDs; all implementation
   notes come from ADRs; all rules come from the control manifest. Never invent
   requirements or rules that don't trace to a source document.
6. **Reference by TR-ID, not by quote** — embed the stable `TR-[system]-NNN` ID
   in the story, not the GDD requirement text. The text lives in the registry and
   is read fresh at review time. This prevents false deviation flags when GDD
   wording is clarified after the story was written.
7. **Never embed Proposed ADRs** — if the governing ADR is Proposed, set the
   story status to Blocked. Embedding implementation guidance from an unaccepted
   ADR creates silent risk when the ADR changes.

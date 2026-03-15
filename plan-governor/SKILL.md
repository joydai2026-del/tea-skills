---
name: plan-governor
description: "Plan lifecycle linter that prevents plan drift, duplicates, and broken references. PROACTIVE: Automatically trigger this skill (silently, in background) whenever the user changes scope, modifies features, reports plan-related bugs, or completes a milestone — not just when explicitly asked. Also triggers on /plan-governor, 'check my plans', session start with multiple plans, or when the user mentions plan drift, duplicate plans, stale plans, or asks 'which plan is current?'. Run silently when auto-triggered — only surface findings if violations exist. Also enforces plan naming conventions, 3-tier hierarchy, and auto-archival of session plans."
---

# Plan Governor — Plan Lifecycle & Naming System

Plans drift. Duplicates accumulate. References go stale. Agents read the wrong plan and build the wrong thing. The Plan Governor prevents this by enforcing a naming convention, a 3-tier hierarchy, and lifecycle rules that keep plans organized across all projects.

## When to Run

### Manual triggers
- `/plan-governor` or "check my plans"
- Before creating a new plan

### Automatic triggers (run WITHOUT being asked)
The governor should run automatically — silently and in the background — whenever:
- **The user changes scope**: "actually, let's do X instead of Y", "add this feature", "remove that"
- **The user reports a bug** that implies a plan gap
- **A milestone completes**: phase/track done → check if plans need status updates
- **Session start** when working on a project with multiple plans
- **A new plan is created** — rename it to follow the convention before proceeding

When auto-triggered, run silently. Only surface findings if there are violations.

---

## Plan Hierarchy — 3 Tiers

All plans live in `<project-root>/docs/plans/`. Every project gets this directory.

### Tier 1: Roadmap (1 per project)
The strategic plan. Covers the full project lifecycle, all phases. Rarely changes — updated only when major direction shifts happen.

**Naming**: `{project}-roadmap.md`
**Examples**:
- `acme-app-roadmap.md`
- `my-saas-roadmap.md`

**Rules**:
- Exactly ONE per project. If a second roadmap appears, it's a violation.
- Never archived — only updated in place or superseded when the project pivots.

### Tier 2: Phase Plan (1 per active phase)
Tactical plan for a specific phase or major feature. Created when a phase begins, archived when it completes. Links back to the roadmap.

**Naming**: `{project}-phase-{phase}-{purpose}.md`
**Examples**:
- `acme-phase-1c-qa-completion.md`
- `acme-phase-2-ux-research.md`
- `my-saas-phase-1-auth-system.md`

**Rules**:
- Links to parent roadmap via frontmatter `parent` field.
- Only 1-2 ACTIVE phase plans at a time (parallel phases are OK).
- Archived when phase completes — changes merge into the roadmap if needed.

### Tier 3: Session Plan (1 per work session)
Operational plan for a single coding session. Created at session start (or when entering plan mode). Auto-archived at session end via `/wrap-up`.

**Naming**: `{date}-{purpose}.md`
**Examples**:
- `2025-01-15-qa-infrastructure.md`
- `2025-01-16-auth-bugfix.md`
- `2025-01-17-api-integration.md`

**Rules**:
- Links to parent phase plan via frontmatter `parent` field.
- Must align with the phase plan — if the session plan contradicts the phase plan, that's a violation.
- **Auto-archived at session end** — `/wrap-up` changes status to ARCHIVED.
- Any scope changes from the session get merged UP into the phase plan before archiving.

---

## Naming Convention

```
Tier 1:  {project}-roadmap.md
Tier 2:  {project}-phase-{phase}-{purpose}.md
Tier 3:  {date}-{purpose}.md
```

All lowercase, hyphens for spaces, no special characters.

The `{purpose}` should be 2-4 words describing the plan's focus:
- Good: `qa-completion`, `ux-research`, `api-integration`
- Bad: `plan`, `stuff`, `v2`

---

## Storage

```
<project-root>/
  docs/
    plans/
      acme-app-roadmap.md               # Tier 1
      acme-phase-1c-qa-completion.md    # Tier 2
      acme-phase-2-ux-research.md       # Tier 2 (RESEARCH)
      2025-01-15-qa-infrastructure.md   # Tier 3
      2025-01-14-auth-bugfix.md         # Tier 3 (ARCHIVED)
      INDEX.md                          # Auto-generated index
```

**Migration from ~/.claude/plans/**: When the governor detects plans in `~/.claude/plans/` that belong to a project, it should recommend moving them to the project's `docs/plans/` with proper naming. Claude Code's plan mode may still create files in `~/.claude/plans/` — the governor renames and moves them on first run.

---

## Frontmatter

Every plan file must have YAML frontmatter:

```yaml
---
status: ACTIVE              # ACTIVE | RESEARCH | APPROVED | SUPERSEDED | ARCHIVED
tier: 1                     # 1 (roadmap) | 2 (phase) | 3 (session)
phase: "1C"                 # Which project phase this covers
project: acme-app           # Project identifier
description: "..."          # One-line summary
created: 2025-01-15         # Creation date
parent: "acme-app-roadmap.md"  # (tier 2+) Parent plan filename
supersedes: "old-plan.md"   # (optional) Which plan this replaces
---
```

**New field: `tier`** — 1, 2, or 3. Determines naming rules and lifecycle behavior.
**New field: `parent`** — For tier 2 and 3 plans, points to the parent plan. Creates the hierarchy chain: session → phase → roadmap.

**Status lifecycle:**
- **ACTIVE** — Currently being executed.
- **RESEARCH** — Contains research findings, not yet actionable.
- **APPROVED** — Reviewed and ready for implementation.
- **SUPERSEDED** — Replaced by a newer plan. Must have `supersedes` link.
- **ARCHIVED** — Completed or session ended. Read-only historical reference.

---

## Auto-Archive at Session End

When `/wrap-up` runs, it should:

1. Find all tier 3 (session) plans with status ACTIVE in the current project
2. For each session plan:
   a. Check if any scope changes in the session plan differ from the parent phase plan
   b. If yes: merge those changes UP into the phase plan (add/update sections)
   c. Change the session plan's status to ARCHIVED
3. Update INDEX.md

This ensures the phase plan is always the single source of truth after a session ends. Session plans become historical records of what was decided and done.

---

## Scan Procedure

### Step 1: Locate All Plans

Scan `<project-root>/docs/plans/` for `.md` files. Also check legacy locations:
- `~/.claude/plans/` — Claude Code plan files (may need migration)
- `<project-root>/docs/` — Old location (pre-convention plans)

Build a table: filename, tier, status, phase, parent, naming compliance.

### Step 2: Validate Naming

For each plan, check naming convention:
- Tier 1: Must match `{project}-roadmap.md`
- Tier 2: Must match `{project}-phase-{phase}-{purpose}.md`
- Tier 3: Must match `{date}-{purpose}.md` where date is `YYYY-MM-DD`

Flag misnaming as a violation with suggested rename.

### Step 3: Validate Frontmatter

For each plan file, check:
1. Has frontmatter with `---` blocks
2. Has required fields: `status`, `tier`, `phase`, `project`, `description`, `created`
3. Tier 2+ has `parent` field pointing to an existing file
4. Valid status value
5. SUPERSEDED plans have `supersedes` link

### Step 4: Validate Hierarchy

- Exactly 1 tier-1 roadmap per project (not 0, not 2+)
- Each tier-2 plan's `parent` points to the tier-1 roadmap
- Each tier-3 plan's `parent` points to a tier-2 plan
- No orphan plans (every plan in the chain has a valid parent)
- Session plans (tier 3) with status ACTIVE should not exist if the session is over

### Step 5: Detect Duplicates

Two plans are likely duplicates if:
- Same tier AND same phase AND same project
- Content overlap >50% in the first 20 lines

### Step 6: Validate INDEX.md

Check `docs/plans/INDEX.md`:
- Missing? Flag as violation.
- Exists? Verify all ACTIVE/RESEARCH/APPROVED plans are listed. Flag stale entries.

### Step 7: Report

```
PLAN GOVERNOR REPORT
====================
Project: {project}
Plans: {count} ({tier1} roadmap, {tier2} phase, {tier3} session)
  ACTIVE: {count}
  RESEARCH: {count}
  ARCHIVED: {count}
  SUPERSEDED: {count}

HIERARCHY:
  {roadmap-name}
    ├── {phase-plan-1} (ACTIVE)
    │   ├── {session-plan-1} (ARCHIVED)
    │   └── {session-plan-2} (ARCHIVED)
    └── {phase-plan-2} (RESEARCH)

VIOLATIONS ({count}):
  [V1] {file} — Wrong naming (expected: {correct name})
  [V2] {file} — Missing frontmatter
  [V3] {file} — Missing parent link
  [V4] {file} — Duplicate of {other}
  [V5] {file} — Orphan (parent {parent} not found)
  [V6] No roadmap found for project {project}

WARNINGS ({count}):
  [W1] {file} — Tier 3 ACTIVE but session appears over
  [W2] {count} ACTIVE phase plans — consider if all are current

VERDICT: {CLEAN | {count} VIOLATIONS — resolve before planning}
```

---

## Creating a New Plan

When the agent needs to create a plan (entering plan mode, user asks for a plan):

1. Run the governor first (silent check for violations)
2. Determine the tier:
   - User says "roadmap" or "big picture" → tier 1
   - User says "plan for phase X" or "plan this feature" → tier 2
   - Default for a coding session → tier 3
3. Generate the filename using the naming convention
4. Create in `docs/plans/` with full frontmatter (including `parent` link)
5. Update INDEX.md

### Handling Claude Code Plan Mode

Claude Code hardcodes `~/.claude/plans/` as its plan file location. We can't change this. The governor handles it automatically:

1. **During plan mode**: Claude Code creates `~/.claude/plans/{random-name}.md`. This is a temporary workspace. Don't interfere.
2. **After plan mode exits**: The governor auto-triggers and detects the new file.
3. **Migration**: Read the plan content, determine the tier (from context or ask user), generate the proper filename, and MOVE it to `docs/plans/` with full frontmatter.
4. **Cleanup**: Delete the `~/.claude/plans/` copy. It served its purpose.
5. **Update INDEX.md** with the new plan.

`docs/plans/` is ALWAYS the single source of truth. `~/.claude/plans/` is just a temporary workspace during plan mode.

If old plans from previous sessions still exist in `~/.claude/plans/`, migrate them all on first run. After migration, `~/.claude/plans/` should have no project plans left (only the auto-generated plan mode files that Claude Code manages).

---

## INDEX.md Format

```markdown
# {Project} — Plan Index

## Active Plans
| Tier | Plan | Status | Phase | Description |
|------|------|--------|-------|-------------|
| 1 | [roadmap](acme-app-roadmap.md) | ACTIVE | 0-5 | Master roadmap |
| 2 | [phase-1c](acme-phase-1c-qa.md) | ACTIVE | 1C | QA completion |

## Archived (recent)
| Date | Plan | Phase | Description |
|------|------|-------|-------------|
| 2025-01-15 | [session](2025-01-15-qa-infrastructure.md) | 1C | QA infra setup |

## Reading Order
1. Start with the roadmap for full context
2. Read the active phase plan for current sprint
3. Check recent session plans for latest decisions
```

The index is auto-maintained. Active plans on top, recent archived below (last 10), older archived omitted (still in the directory, just not indexed).

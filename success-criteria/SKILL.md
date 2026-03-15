---
name: success-criteria
description: "Generates measurable, machine-readable success criteria from any plan or spec file. PROACTIVE: Automatically trigger this skill whenever the user gives feedback that changes features, scope, or requirements — regenerate or update the affected checks in qa-checklist.yaml without being asked. Also triggers on: /success-criteria, 'generate checklist', 'define success criteria', 'what does done look like', 'create acceptance criteria', 'make a QA checklist', or when transitioning from planning to implementation. When auto-triggered by user feedback, update only the affected checks (don't regenerate the entire checklist). This is the bridge between a plan and the /qa-runner — without it, QA has nothing to verify against."
---

# Success Criteria Generator

Plans say what to build. Success criteria say how to know it's built correctly. This skill reads any plan, spec, or task list and produces a structured YAML checklist where every item is either automatically testable (run a command, check the output) or code-auditable (read specific files, verify specific logic).

The output format is designed to feed directly into `/qa-runner`, which uses Ralph Loop to iterate until all checks pass.

## Why This Matters

Bad success criteria are the root cause of QA loops that never close. If criteria are vague ("app works well"), subjective ("UI looks good"), or incomplete (missing edge cases), the QA agent has no clear target. Every iteration becomes a judgment call, and judgment calls are where drift happens.

Good success criteria are:
- **Specific**: "GET /health returns HTTP 200" not "API works"
- **Measurable**: A command you can run, a file you can read, a condition you can check
- **Complete**: Covers every requirement in the plan, not just the obvious ones
- **Independent**: Each check can pass or fail on its own

## Reactive Triggers (run WITHOUT being asked)

This skill should auto-trigger whenever the user gives feedback that changes what "done" looks like:

- **Feature change**: "actually, make it do X instead of Y" → update affected checks
- **Bug report**: "this doesn't work" → add or modify the relevant check, then trigger `/qa-runner`
- **Scope addition**: "also add Z" → add new checks to the checklist
- **Scope removal**: "we don't need W anymore" → mark those checks as blocked/ARCHIVED

When auto-triggered, run the **incremental update** path (see "Updating Existing Checklists" section), not a full regeneration. Update only the checks affected by the user's feedback, then hand off to `/qa-runner`.

## Procedure

### Step 1: Read the Plan

Accept a plan file path as argument: `/success-criteria path/to/plan.md`

If no path given, look for the most relevant plan:
1. Check if `docs/qa-checklist.yaml` already exists (if so, offer to regenerate or update)
2. Look for plan files in `docs/`, `.vault/`, and `~/.claude/plans/` matching the current project
3. If multiple plans exist, ask which one to use (or use the plan index if available)

Read the entire plan. Extract every requirement, feature, endpoint, UI flow, and constraint mentioned.

### Step 2: Read the Codebase Structure

Understanding the actual code is essential for generating useful criteria. Without it, you'd produce generic checks that don't map to real files.

1. Read the project's file tree (focus on `src/`, `lib/`, `app/` directories)
2. Identify key entry points (main.ts, App.tsx, index.ts, main.dart, etc.)
3. Note the tech stack (this determines what automated checks are possible)
4. If a previous checklist exists, read it to preserve check IDs and any manual annotations

### Step 3: Classify Requirements

For each requirement from the plan, determine its type:

**Automated** — Can be verified by running a command and checking output:
- API endpoints (curl + status code check)
- Build commands (tsc, flutter analyze, npm run build — exit code 0)
- Test suites (flutter test, npm test — all pass)
- File existence checks (ls path/to/file)
- Database state (query + expected result)
- Process health (port listening, PID exists)

**Code-audit** — Requires reading source code and evaluating logic:
- UI conditional rendering ("if no user profile, show Onboarding")
- Event handlers ("keyboard shortcut 'a' calls handleApprove")
- Data flow ("form submission posts to /settings endpoint")
- Business logic ("user permissions check based on subscription tier")
- Error handling ("API failure shows user-friendly message, not stack trace")

The classification matters because automated checks are faster, more reliable, and can run without human judgment. Maximize automated checks — only use code-audit when there's no command that can verify the requirement.

### Step 4: Generate the Checklist

Output to `docs/qa-checklist.yaml` in this format:

```yaml
project: {project-name}
created: {today's date}
source_plan: {path to plan file used}
max_retries: 3
build_command: "{the project's build/compile command}"
summary:
  total: {count}
  passed: 0
  failed: 0
  blocked: 0
  pending: {count}

checks:
  - id: {CATEGORY}{NUMBER}
    name: "{human-readable description}"
    type: automated
    command: "{exact bash command to run}"
    expected: "{expected output or comparison value}"
    comparison_mode: exact   # exact | regex | numeric | exit_code
    side_effects: false      # true if the check writes data or costs money
    status: pending
    retries: 0
    blocked_reason: null

  - id: {CATEGORY}{NUMBER}
    name: "{human-readable description}"
    type: code-audit
    files: [{list of files to read}]
    criteria: "{precise, unambiguous condition to verify}"
    status: pending
    retries: 0
    blocked_reason: null
```

Read `references/checklist-format.md` for the complete format specification, including the `expected` field's comparison modes.

### Step 5: Organize by Category

Group checks into logical categories using letter prefixes. Choose categories that match the plan's structure. Common patterns:

- By feature area: A (Onboarding), B (Dashboard), C (Actions), D (Navigation)
- By layer: A (API), B (Frontend), C (Database), D (Build)
- By priority: A (P0 Critical), B (P1 Important), C (P2 Nice-to-have)

Number checks sequentially within each category: A1, A2, A3, B1, B2, etc.

### Step 6: Quality Review

Before presenting the checklist, verify:

1. **Coverage**: Every requirement from the plan has at least one check. Read the plan again and cross-reference.
2. **Precision**: Automated check commands are syntactically correct and will actually work. Code-audit criteria are specific enough that two people would agree on pass/fail.
3. **Independence**: No check depends on another check passing first (order shouldn't matter).
4. **Build gate**: At least one check verifies the project compiles/builds cleanly.
5. **No gaps in IDs**: Categories are contiguous (A1, A2, A3 — not A1, A3, A5).

### Step 7: Present Summary

Show the user:

```
SUCCESS CRITERIA GENERATED
==========================
Source plan: {plan file}
Output: docs/qa-checklist.yaml

Categories:
  A. {name} ({count} checks: {auto} automated, {audit} code-audit)
  B. {name} ({count} checks: {auto} automated, {audit} code-audit)
  ...

Total: {count} checks ({auto_total} automated, {audit_total} code-audit)
Build gate: {build_command}

Ready for /qa-runner.
```

## Comparison Mode & Side Effects

Every automated check MUST have explicit `comparison_mode` and `side_effects` fields.

### comparison_mode (required for automated checks)

| Mode | `comparison_mode` | `expected` value | Example |
|------|-------------------|------------------|---------|
| Exact string | `exact` | `"200"` | HTTP status code |
| Regex match | `regex` | `"/No issues found/"` | Substring pattern |
| Numeric comparison | `numeric` | `">= 354"` | Test count baseline |
| Exit code | `exit_code` | `"0"` | Build must exit clean |

### side_effects (required for automated checks)

Set `side_effects: true` when a check:
- Writes to a database (POST/PUT/DELETE endpoints)
- Calls external APIs that cost money (LLM generation, SMS, email)
- Mutates application state that other checks depend on

The qa-runner handles side-effect checks differently: they run **once, at the end**, outside the retry loop. This prevents data pollution and wasted credits.

Read-only checks (GET endpoints, build commands, file scans) are always `side_effects: false`.

## Project Templates

Check `references/` for reusable check patterns. These are starting points — always customize based on the actual plan. When generating criteria for a project that matches a template, adapt it rather than starting from scratch.

## Updating Existing Checklists

If `docs/qa-checklist.yaml` already exists:

1. Read the existing checklist
2. Preserve check IDs that still apply (don't renumber)
3. Mark removed checks as ARCHIVED (don't delete — the QA run log may reference them)
4. Add new checks with the next available ID in each category
5. Update the summary counters
6. Show a diff: "Added X checks, archived Y checks, modified Z checks"

This matters because the QA runner tracks progress by check ID. Renumbering breaks the run log.

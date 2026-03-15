---
name: qa-runner
description: "Closed-loop QA execution using Ralph Loop. PROACTIVE: After /success-criteria updates the checklist (whether manually or auto-triggered by user feedback), automatically invoke this skill to re-verify affected checks — don't wait to be asked. Also triggers on: 'run QA', 'start the QA loop', 'verify the checklist', /qa-runner, or when docs/qa-checklist.yaml exists with pending/failed checks. Reads qa-checklist.yaml and iterates autonomously — fixing bugs, re-verifying, looping — until ALL checks pass or are blocked. This is the execution engine that closes the QA loop humans are tired of doing manually."
---

# QA Runner — Closed-Loop Execution via Ralph Loop

The QA Runner is the execution engine that closes the QA loop. It reads a structured checklist, works through each check, fixes failures, and doesn't stop until everything passes or is provably blocked. No human feedback needed between iterations.

It works by invoking Ralph Loop — a Claude Code plugin that re-feeds the same prompt every time the agent tries to exit. The agent sees its own previous work in the modified files and picks up where it left off. The loop only breaks when the completion promise ("ALL_CHECKS_PASS") is genuinely true.

## Reactive Triggers (run WITHOUT being asked)

The qa-runner should auto-trigger after `/success-criteria` updates the checklist:

- **After checklist update**: success-criteria added/modified checks → re-verify those specific checks
- **After bug fix outside the loop**: if the agent fixes a bug manually (not via Ralph Loop), re-run affected checks to confirm the fix
- **After build/deploy**: verify build gates (J1, J2) still pass

When auto-triggered, the runner can do a **targeted run** — only re-verify the checks that changed status or were newly added, rather than starting the full loop from scratch. If targeted checks all pass, report silently. If any fail, enter the full Ralph Loop.

## Prerequisites

1. **`docs/qa-checklist.yaml` must exist.** Generate it with `/success-criteria` first.
2. **Ralph Loop plugin must be installed.** Check: `ls ~/.claude/plugins/` should show ralph-loop.
3. **Project-specific prerequisites** (examples):
   - Node projects: `node_modules/` must exist (`npm install` if needed), server running
   - Flutter projects: `flutter` must be on PATH
   - Python projects: virtual env activated, dependencies installed

## Procedure

### Step 1: Read the Checklist

Read `docs/qa-checklist.yaml`. Parse the summary section:

```
QA RUNNER — Pre-flight
======================
Project: {project}
Checklist: {total} checks ({passed} passed, {failed} failed, {blocked} blocked, {pending} pending)
Build gate: {build_command}
```

If all checks are already passed or blocked, report completion and stop.

### Step 2: Verify Prerequisites

Run project-specific prerequisite checks. These vary by project type:

**Node.js / TypeScript:**
- Server alive: `curl -s http://localhost:3000/health` returns 200
- If not: try `cd <server-path> && node dist/index.js &` and wait 3s
- TypeScript compiles: `npx tsc --noEmit` exits 0

**Flutter:**
- Flutter available: `flutter --version` exits 0
- Dependencies: `flutter pub get` if `pubspec.lock` missing

**Python:**
- Virtual env: `python -c "import sys; print(sys.prefix)"` shows venv path
- Dependencies: `pip install -r requirements.txt` if needed

If prerequisites fail and can't be auto-fixed, report what's needed and stop. Don't enter the loop with broken prerequisites — it wastes iterations.

### Step 3: Build the Ralph Loop Prompt

Read the prompt template from `references/prompt-template.md`. Substitute these variables:

| Variable | Source |
|----------|--------|
| `{PROJECT_NAME}` | From `qa-checklist.yaml` → `project` field |
| `{PROJECT_PATH}` | Current working directory |
| `{BUILD_COMMAND}` | From `qa-checklist.yaml` → `build_command` field |
| `{MAX_RETRIES}` | From `qa-checklist.yaml` → `max_retries` field (default: 3) |
| `{PROJECT_CONTEXT}` | Project-specific notes (see below) |

**Project context** is critical — it gives the agent the knowledge it needs to fix bugs without asking for help. Include:

- Key file paths and what they contain
- Known gotchas and patterns (from memory.md or CLAUDE.md)
- Architecture overview (how components connect)
- Common failure modes and their fixes

### Step 4: Invoke Ralph Loop

```bash
/ralph-loop "{built_prompt}" --completion-promise 'ALL_CHECKS_PASS' --max-iterations {MAX_ITERATIONS}
```

Default `MAX_ITERATIONS`: 120 (generous for ~60 checks at ~2 iterations per check). Override with `/qa-runner --max-iterations 50`.

### Step 5: Monitor (Ralph Loop handles this automatically)

Each iteration, the agent inside the loop will:
1. Read `docs/qa-checklist.yaml`
2. Pick the next check to work on
3. Fix it or mark it blocked
4. Update the YAML
5. Append to `docs/qa-run-log.md`
6. Try to exit — Ralph Loop blocks unless ALL_CHECKS_PASS is true

You don't need to monitor this. Ralph Loop handles iteration management. When it completes, the agent will output a final summary.

## The Run Log

Each iteration appends to `docs/qa-run-log.md`:

```markdown
### Iteration {N} — {timestamp}
- Check: {id} ({name})
- Result: {passed | failed | blocked}
- Action: {what was fixed, or why it was blocked}
- Files changed: {list}
- Build gate: {pass | fail}
```

The run log is append-only. It provides a complete audit trail of what the agent did and why. If a check passes and then fails later (regression), the log shows when and how.

## Handling Regressions

The most dangerous failure mode is fixing one check and breaking another. The prompt template addresses this:

1. After every code edit, the agent runs the build command as a regression gate
2. If the build breaks, the agent reverts the edit immediately
3. If a previously-passing check fails, the agent prioritizes re-fixing it over new checks
4. The checklist YAML tracks status changes — if a check goes from `passed` back to `failed`, that's a regression signal

## When the Loop Ends

The loop ends when:
- **All checks are `passed` or `blocked`** — the agent outputs `<promise>ALL_CHECKS_PASS</promise>`
- **Max iterations reached** — Ralph Loop stops automatically
- **User cancels** — `/cancel-ralph` stops the loop

After completion, read `docs/qa-checklist.yaml` for the final state:

```
QA RUNNER — Final Report
========================
Total: {total} checks
Passed: {passed} ({percentage}%)
Blocked: {blocked} ({list of blocked IDs with reasons})
Build: {pass | fail}

{If blocked checks exist:}
BLOCKED ITEMS (require human action):
  {id}: {blocked_reason}
```

## Configuring for Different Projects

The QA runner is project-agnostic. The project-specific knowledge lives in two places:

1. **`docs/qa-checklist.yaml`** — The checks themselves (generated by /success-criteria)
2. **The project context block** — Injected into the Ralph Loop prompt (Step 3)

To use this for a new project:
1. Write a plan
2. Run `/success-criteria` to generate the checklist
3. Run `/qa-runner` — it auto-detects the project and builds the prompt

No project-specific code in the runner itself. The intelligence is in the checklist and the prompt.

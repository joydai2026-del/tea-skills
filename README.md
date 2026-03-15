# QA Loop Skills

3 Claude Code skills that close the plan-drift loop automatically. Plans stay organized, success criteria stay in sync, and QA runs itself.

## What's in here

| Skill | What it does |
|-------|-------------|
| **plan-governor** | Enforces a 3-tier plan hierarchy (Roadmap > Phase > Session). Prevents drift, duplicates, and stale plans. |
| **success-criteria** | Reads any plan and generates a machine-readable YAML checklist of testable checks. |
| **qa-runner** | Autonomous verify-and-fix loop. Reads the checklist, fixes failures, can't exit until all checks pass. |

## How they connect

```
Plan Governor → Success Criteria → QA Runner
(organize)      (define done)      (fix + verify)
```

Each triggers the next. Scope changes update the plan, which updates the checklist, which re-triggers verification.

## Setup

### 1. Install the skills

```bash
git clone https://github.com/joydai2026-del/qa-loop-skills.git
cp -r qa-loop-skills/plan-governor ~/.claude/skills/
cp -r qa-loop-skills/success-criteria ~/.claude/skills/
cp -r qa-loop-skills/qa-runner ~/.claude/skills/
```

### 2. Wire them together

The skills don't auto-chain on their own. Add this block to your project's `CLAUDE.md` (create one in your project root if you don't have it):

```markdown
## QA Pipeline

Use the 3-tier plan system in `docs/plans/`:
- Tier 1: `{project}-roadmap.md` (one per project)
- Tier 2: `{project}-phase-{N}-{purpose}.md` (one per active phase)
- Tier 3: `{YYYY-MM-DD}-{purpose}.md` (one per session, archived at session end)

After creating or updating any plan, run `/success-criteria` to generate `docs/qa-checklist.yaml`.
After any scope change, update the affected plan first, then re-run `/success-criteria`.
After success criteria are generated or updated, run `/qa-runner` to verify all checks pass.
Run `/plan-governor` at session start and before creating new plans.
```

That's it. The skills read your plans and generate everything else. The `CLAUDE.md` block tells Claude Code when to trigger each one.

### 3. Use them

- `/plan-governor` — audit your plans
- `/success-criteria path/to/plan.md` — generate a QA checklist
- `/qa-runner` — run the autonomous QA loop

## Requirements

- [Claude Code](https://claude.ai/claude-code) CLI
- For qa-runner: Ralph Loop plugin (autonomous loop execution)

## License

MIT

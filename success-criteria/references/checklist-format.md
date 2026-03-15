# QA Checklist YAML Format Specification

## Overview

The `qa-checklist.yaml` file is the shared state between `/success-criteria` (which generates it) and `/qa-runner` (which executes against it). It tracks every success criterion, its verification method, and its current status.

## Schema

```yaml
# Header — project metadata
project: string          # Project identifier (e.g., "acme-app", "my-saas")
created: date            # YYYY-MM-DD when checklist was generated
source_plan: string      # Path to the plan file used to generate criteria
max_retries: integer     # Max fix attempts per check before marking blocked (default: 3)
build_command: string    # Command that verifies the project compiles/builds

# Summary — updated by qa-runner after each iteration
summary:
  total: integer
  passed: integer
  failed: integer
  blocked: integer
  pending: integer

# Checks — the actual success criteria
checks:
  - id: string           # Unique ID: {LETTER}{NUMBER} (e.g., A1, B3, H10)
    name: string         # Human-readable description
    type: enum           # "automated" or "code-audit"

    # For automated checks:
    command: string      # Exact bash command to run
    expected: string     # Expected output (see comparison modes below)
    comparison_mode: enum  # "exact" | "regex" | "numeric" | "exit_code" (explicit, don't infer from string)
    side_effects: boolean  # true if the check mutates state (writes to DB, calls external APIs, costs money). Default: false.
                           # qa-runner runs side_effects checks ONCE at the end, outside the retry loop.
                           # Read-only checks (GET endpoints, build commands, file scans) are always false.

    # For code-audit checks:
    files: list[string]  # File paths to read
    criteria: string     # Precise condition to verify in the code

    # Status tracking (managed by qa-runner):
    status: enum         # "pending" | "passed" | "failed" | "blocked"
    retries: integer     # Number of fix attempts so far
    blocked_reason: string|null  # Why this check can't be fixed (null if not blocked)
```

## Comparison Modes

Each automated check MUST have an explicit `comparison_mode` field. Do not infer the mode from the `expected` string — always set it explicitly.

| Mode | `comparison_mode` | `expected` value | Semantics |
|------|-------------------|------------------|-----------|
| Exact string | `exact` | `"200"` | stdout must exactly equal this string (trimmed) |
| Regex match | `regex` | `/No issues found/` | stdout must match this regex |
| Numeric >= | `numeric` | `">= 354"` | numeric stdout must satisfy the comparison |
| Exit code | `exit_code` | `"0"` | command's exit code must equal this number |

## Status Values

| Status | Meaning | Set by |
|--------|---------|--------|
| `pending` | Not yet attempted | success-criteria (initial) |
| `passed` | Check verified successfully | qa-runner |
| `failed` | Check attempted but failed | qa-runner |
| `blocked` | Cannot fix after max_retries attempts | qa-runner |

## ID Convention

- Letters = categories (A, B, C, ..., Z)
- Numbers = sequential within category (1, 2, 3, ...)
- IDs are stable — once assigned, never renumber
- When removing a check, mark it `blocked` with reason "ARCHIVED" rather than deleting
- When adding checks to an existing category, use the next available number

## Example

```yaml
project: acme-app
created: 2025-01-15
source_plan: "docs/plans/my-phase-plan.md"
max_retries: 3
build_command: "cd /path/to/project && npx tsc --noEmit"
summary:
  total: 5
  passed: 2
  failed: 1
  blocked: 0
  pending: 2

checks:
  - id: H1
    name: "GET /health returns 200"
    type: automated
    command: "curl -s -o /dev/null -w '%{http_code}' http://localhost:3000/health"
    expected: "200"
    comparison_mode: exact
    side_effects: false
    status: passed
    retries: 0
    blocked_reason: null

  - id: H2
    name: "GET /content returns array"
    type: automated
    command: "curl -s http://localhost:3000/content | node -e \"process.stdin.on('data',d=>{const j=JSON.parse(d);console.log(Array.isArray(j))})\""
    expected: "true"
    comparison_mode: exact
    side_effects: false
    status: failed
    retries: 1
    blocked_reason: null

  - id: A1
    name: "App launches to onboarding if no user profile"
    type: code-audit
    files: [src/App.tsx, src/components/Onboarding.tsx]
    criteria: "App.tsx conditionally renders Onboarding when user store is empty. Verify: useUserStore() hook checks for profile existence, renders Onboarding if null/empty, MainLayout otherwise."
    status: passed
    retries: 0
    blocked_reason: null

  - id: G3
    name: "POST /users creates a user"
    type: automated
    command: "curl -s -X POST http://localhost:3000/users -H 'Content-Type: application/json' -d '{\"name\":\"test\"}'"
    expected: "/created|already exists/"
    comparison_mode: regex
    side_effects: true   # POST that writes data
    status: pending
    retries: 0
    blocked_reason: null

  - id: J1
    name: "TypeScript compiles with zero errors"
    type: automated
    command: "cd /path/to/project && npx tsc --noEmit"
    expected: "0"
    comparison_mode: exit_code
    side_effects: false
    status: pending
    retries: 0
    blocked_reason: null
```

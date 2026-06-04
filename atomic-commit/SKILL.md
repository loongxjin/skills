---
name: atomic-commit
description: >
  Driving coding workflow: write code in small atomic steps, commit after each step with Conventional Commits.
  The agent must proactively commit after every logical change — not batch all changes at the end.
  Use this skill when the user asks to implement features, fix bugs, refactor code, or any coding task.
  Triggers: coding, writing code, implementing, 开发, 编码, 实现功能, 写代码, fix bug, refactor, 小步提交, 原子提交, atomic commit.
---

# Atomic Commit Workflow

**Core principle:** Break all work into small atomic steps. Commit after each step. Never accumulate uncommitted changes.

## How to Work

When the user gives you a coding task, **do NOT implement everything then commit once at the end.** Instead:

1. **Plan** — Break the task into small, logical steps.
2. **Execute one step** — Write the minimal code for this step only.
3. **Test** — Verify the step works (run tests, build, or manual check).
4. **Commit** — Stage ONLY files changed in this step, commit immediately.
5. **Next step** — Repeat from step 2.

## Commit Format

```
<type>: <imperative description>
```

| Type | When to use |
|------|-------------|
| `feat` | New feature or new behavior |
| `fix` | Bug fix |
| `refactor` | Restructure code without changing behavior |
| `test` | Add or update tests |
| `docs` | Documentation changes |
| `style` | Formatting only, no logic change |
| `perf` | Performance improvement |
| `chore` | Tooling, dependencies, CI |

Rules:

- Imperative mood: "add user model" not "added user model".
- Under 72 characters.
- No unrelated files — `git add` specific files only, never `git add .`.

## Concrete Example

User asks: "Add user registration with email validation."

Do this:

```
Step 1: Create user model
  → write model code → test → commit: "feat: add user model"

Step 2: Add email validation
  → write validation → test → commit: "feat: add email validation for user"

Step 3: Add registration endpoint
  → write handler → test → commit: "feat: add user registration endpoint"

Step 4: Add registration tests
  → write tests → run all tests → commit: "test: add user registration tests"
```

NOT this:

```
❌ Write all code → commit once: "feat: add user registration"
```

## What to Commit

Only files that belong to the current step's logical change. Always exclude:

- IDE configs (`.idea/`, `.vscode/`)
- OS files (`.DS_Store`)
- Build artifacts (`dist/`, `build/`, `*.pyc`)
- Dependencies (`node_modules/`, `vendor/`)
- Env files (`.env`)

## Staging Checklist

Before each commit, run:

```bash
git diff --cached --name-only
```

If any staged file is unrelated to this step, unstage it:

```bash
git reset HEAD <file>
```

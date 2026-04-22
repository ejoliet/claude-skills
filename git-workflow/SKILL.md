---
name: git-workflow
description: >
  Advanced git workflow skill covering conventional commits, safe staging, pre-push validation,
  branching strategy, worktree usage, and release automation. Use when committing, reviewing
  pre-push changes, managing release branches, or setting up git conventions for a new repo.
  Enforces no-secrets policy and Conventional Commit message format.
license: MIT
metadata:
  source: https://github.com/alirezarezvani/claude-skills
  skill-author: alirezarezvani (adapted for ejoliet)
---

# Git Workflow

Advanced git conventions, safe commit practices, and pre-push validation.

---

## When to Use This Skill

- Creating a commit (Conventional Commit format)
- Pre-push validation and security review
- Managing feature branches, release branches, or worktrees
- Setting up git conventions for a new repo
- Rebasing, squashing, or cleaning up history

---

## Conventional Commit Format

```
<type>(<scope>): <subject>   ← max 72 chars

[optional body]

[optional footer: Co-Authored-By, Fixes #N, BREAKING CHANGE]
```

**Types:** `feat` | `fix` | `docs` | `style` | `refactor` | `perf` | `test` | `build` | `ci` | `chore` | `revert`

**Scope:** kebab-case module name, e.g., `feat(roman-soc):`, `fix(vo-explorer):`

**Rules:**
- Subject uses imperative mood ("add X", not "added X")
- Subject ≤ 72 characters
- Body wraps at 100 characters
- `BREAKING CHANGE:` in footer for API changes

---

## Safe Staging Protocol

**Never use `git add .` or `git add -A`** without explicit review.

```bash
# 1. Review pending changes
git status --short

# 2. Diff each file before staging
git diff -- path/to/file

# 3. Stage intentionally, file by file
git add path/to/specific/file

# 4. Check for secrets before committing
git diff --cached | grep -iE "(password|secret|token|key|credential)" && echo "⚠️ SECRET RISK"
```

**Never commit:**
- `.env` files
- AWS credentials or access keys
- API tokens or private keys
- Database connection strings with passwords

---

## Pre-Push Validation Checklist

Run before every push:

```bash
# 1. Lint workflows (if GitHub Actions present)
yamllint .github/workflows/

# 2. Python syntax check
python -m compileall src/

# 3. Markdown link check
markdown-link-check README.md

# 4. Dependency security audit
pip-audit -r requirements.txt   # Python
npm audit                        # Node.js

# 5. Run tests
pytest -x -q                     # Python
npm test                         # Node.js
```

---

## Branching Strategy

```
main          ← production; protected; no direct pushes
develop       ← integration branch (optional for large teams)
feature/*     ← new features: feature/roman-soc-graphql
fix/*         ← bug fixes: fix/soc-delivery-status
release/*     ← release candidates: release/v1.2.0
hotfix/*      ← emergency production fixes
```

**Branch naming rules:**
- kebab-case only
- Prefix must match type: `feature/`, `fix/`, `release/`, `hotfix/`
- Include ticket ID if available: `feature/ROM-42-file-status-endpoint`

---

## Worktree Usage

Worktrees let you check out multiple branches simultaneously without stashing:

```bash
# Create a worktree for a hotfix while keeping feature branch open
git worktree add ../hotfix-soc hotfix/soc-null-pointer

# List active worktrees
git worktree list

# Remove when done
git worktree remove ../hotfix-soc
```

Use cases:
- Running tests on main while developing on feature branch
- Reviewing a PR without disturbing local changes
- Parallel work on independent features

---

## Release Workflow

```bash
# 1. Create release branch from main
git checkout -b release/v1.2.0 main

# 2. Bump version in pyproject.toml / package.json
# 3. Update CHANGELOG.md
# 4. Commit: chore(release): bump version to 1.2.0

# 5. Tag
git tag -s v1.2.0 -m "Release v1.2.0"

# 6. Merge back to main and develop
git checkout main && git merge --no-ff release/v1.2.0
git checkout develop && git merge --no-ff release/v1.2.0

# 7. Push with tags
git push origin main develop --follow-tags
```

---

## Common Fixes

| Problem | Command |
|---|---|
| Undo last commit (keep changes staged) | `git reset --soft HEAD~1` |
| Unstage a file | `git restore --staged path/to/file` |
| Discard unstaged changes in file | `git restore path/to/file` |
| Amend last commit message (not yet pushed) | `git commit --amend` |
| Interactive rebase last N commits | `git rebase -i HEAD~N` |
| Find commit that introduced a bug | `git bisect start` |
| Show who changed a line | `git blame -L 40,50 path/to/file` |

---

## IPAC/ejoliet Defaults

- Default branch: `main`
- Commit signing: GPG when available
- PR merge strategy: squash-and-merge for features; merge commit for releases
- No force-push to `main` or `develop`; use `--force-with-lease` on feature branches only

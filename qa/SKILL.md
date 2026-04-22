---
name: qa
description: >
  Full QA pass on all session changes using a two-pass diff + file review with severity-tiered
  findings (P1/P2/P3). Covers correctness, security, data integrity, regressions, and
  completeness. Use after implementing a feature or fix to catch issues before pushing.
  Proactively fixes P1 issues, confirms P2 with user, skips P3 unless requested.
license: MIT
metadata:
  source: https://github.com/SZoloth/skill-pack
  skill-author: SZoloth (adapted for ejoliet)
---

# QA — Quality Assurance Pass

Six-step dual-pass code review for all session changes.

---

## When to Use This Skill

- After implementing a feature, fix, or refactor before pushing
- Before opening a PR on Roman SOC, ADS MCP, or any IPAC service
- When the user asks "review this", "check my changes", or "QA this"
- As a final gate after `tdd` cycle is complete

---

## Six-Step Process

### Step 1: Gather Changes

```bash
git status
git diff --stat
git diff HEAD
```

Capture all staged and unstaged modifications.

### Step 2: Build Context

Collect 2–3 sentences explaining:
- What feature/task was implemented
- Which files changed and why
- Any known edge cases or constraints

### Step 3: Diff-Based Review (Pass 1)

Review the diff line-by-line against this checklist:

| Category | What to check |
|---|---|
| **Correctness** | Logic matches stated intent; edge cases handled |
| **Runtime errors** | Null dereferences, index out of bounds, unhandled exceptions |
| **Security** | No hardcoded credentials, SQL injection, XSS, command injection |
| **Data integrity** | DB writes are transactional; no partial updates possible |
| **Integration** | API contracts unchanged or versioned; no silent breaking changes |
| **Regressions** | Existing tests still pass; no behavior removed unintentionally |
| **Completeness** | All acceptance criteria met; no TODOs left in new code |

Report findings with `file:line` locations and severity:
- **P1** — Bug or security issue; blocks merge
- **P2** — Likely problem or design smell; requires discussion
- **P3** — Style, minor optimization, or low-risk suggestion

### Step 4: File-Based Audit (Pass 2)

Read each changed file end-to-end (not just the diff) checking:

- [ ] Imports are used and not shadowed
- [ ] Type annotations correct (Python) / types sound (TypeScript)
- [ ] Error handling covers all external calls
- [ ] Edge cases (empty list, zero, None/null, large input)
- [ ] Naming conventions consistent with surrounding code
- [ ] Test coverage for new public interfaces
- [ ] Build/lint integrity (no syntax errors, no broken imports)

### Step 5: Synthesize Findings

Merge both passes, deduplicate overlapping issues, present organized by severity tier:

```
P1 Issues (must fix before merge):
  - src/api/files.py:47 — unhandled None on conn.fetchrow(); can raise AttributeError

P2 Issues (should discuss):
  - src/api/files.py:31 — no pagination on list endpoint; will OOM on large tables

P3 Suggestions (optional):
  - src/api/files.py:12 — DSN should come from env var not module-level constant
```

### Step 6: Fix Offer

- **P1** — Proactively fix immediately; show diff before applying
- **P2** — Show the issue; ask user to confirm before fixing
- **P3** — List but do not fix unless user explicitly requests

---

## Critical Rules

- Always present findings before applying fixes
- Include task context for better analysis quality
- Never skip P1 issues to save time
- P3 suggestions are informational only — do not moralize

---

## Python Security Quick-Checks

```python
# ❌ SQL injection risk
query = f"SELECT * FROM files WHERE name = '{name}'"

# ✅ Parameterized
query = "SELECT * FROM files WHERE name = $1"
await conn.fetch(query, name)

# ❌ Shell injection
subprocess.run(f"convert {filename}", shell=True)

# ✅ Safe
subprocess.run(["convert", filename])
```

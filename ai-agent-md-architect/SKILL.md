---
name: ai-agent-md-architect
description: "Generates or upgrades a CLAUDE.md (or AGENTS.md, .cursor/rules, .github/copilot-instructions.md) for any software project, optimized for token-efficient agent guidance. Centers the file on Project Invariants (load-bearing rules + reasons + blast radius) and a Recently Burned log (recent mistakes), instead of folder maps and setup commands the agent can infer itself. Trigger whenever Emmanuel says: \"create a CLAUDE.md\", \"write CLAUDE.md\", \"update my CLAUDE.md\", \"add to CLAUDE.md\",  \"agent rules for this repo\", \"what should go in CLAUDE.md\", \"AGENTS.md\",  \"cursor rules\", \"copilot instructions\", \"memory file for this project\", or any request to bootstrap/improve agent-facing guidance for an existing or new repo. Also trigger when reviewing an existing CLAUDE.md for noise reduction or staleness."
---

# CLAUDE.md Architect Skill

Produces a CLAUDE.md that is:

1. **Invariant-first** — load-bearing rules at top, reasons + blast radius beside each
2. **Inference-aware** — omits anything the agent can read from `pyproject.toml`, `package.json`, or `tree`
3. **Burn-logged** — "Recently Burned" section tracks last 3–5 mistakes with dates
4. **Greppable** — uses `AIDEV-*` comment markers consistent with Emmanuel's coding tips
5. **Token-lean** — target ≤ 200 lines; if longer, split into `CLAUDE.md` + `docs/agent-context/*.md`

---

## Phase 0 — Detect Mode

Scan conversation and repo state. Pick one:

| Mode | Trigger | Output |
|------|---------|--------|
| **NEW** | No CLAUDE.md exists; new or empty repo | Full template, mark unknowns `<!-- TODO: confirm -->` |
| **UPGRADE** | CLAUDE.md exists but is bloated / generic / stale | Diff: invariants added, noise removed, burn log started |
| **AUDIT** | User asks "review my CLAUDE.md" | Inline annotated critique, no rewrite unless asked |

If mode unclear → ask once, single question, 2 options.

---

## Phase 1 — Gather Invariants (the only required input)

Ask up to **3 short questions**. Skip any answerable from context (memory, repo files, prior chat).

1. **What rule, if broken, causes real damage?** (data corruption, audit failure, security incident, person-you-must-apologize-to)
2. **What looks wrong but is intentional?** (weird import order, pinned old version, custom fork)
3. **What did an agent (or you) recently get wrong here?** (last 30 days)

If user has nothing → flag the file as **premature**: "CLAUDE.md without invariants is just noise. Come back when you have one rule worth writing down." Offer minimal stub instead of full file.

---

## Phase 2 — Inference Pass (do NOT ask, read instead)

Pull from repo so user doesn't repeat themselves:

| Source | Extract |
|--------|---------|
| `pyproject.toml` / `package.json` | Language, version, deps, scripts |
| `Dockerfile` / `compose.yml` | Runtime, ports, services |
| `.github/workflows/` | CI commands, test runners |
| `Makefile` / `justfile` | Canonical commands |
| `README.md` | Purpose, audience |
| `git log --oneline -20` | Recent change patterns |
| Memory / past chats | Stack conventions (AWS account, region, DBAs, etc.) |

**Anything inferable goes in a one-line "Stack" footer, not a multi-section dump.**

---

## Phase 3 — Generate (use template below)

### Template

```markdown
<!-- CLAUDE.md — agent guidance for {project-name}
     Last reviewed: {YYYY-MM-DD}  Owner: {name}
     AIDEV-NOTE: keep ≤ 200 lines; split overflow into docs/agent-context/ -->

# {project-name}

{One sentence: what this is and who runs it.}

---

## 🛑 Project Invariants (DO NOT VIOLATE)

<!-- AIDEV-INVARIANT: rule + reason + blast radius. If you can't write all three, drop it. -->

1. **{rule}** — {reason}.
   Blast: {what breaks, who notices}.
2. **{rule}** — {reason}.
   Blast: {what breaks, who notices}.
3. ...

> If an instruction below conflicts with an invariant, the invariant wins.

---

## 🔥 Recently Burned

<!-- AIDEV-BURN: last 3–5 mistakes. Date each. Prune quarterly. -->

- **{YYYY-MM-DD}**: {what went wrong} → {corrective rule}.
- **{YYYY-MM-DD}**: {what went wrong} → {corrective rule}.

---

## ✅ Workflow Expectations

<!-- Only include items the agent CANNOT infer from CI files or scripts. -->

- Confirm plan before code when scope > 1 file.
- Use `uv` for Python envs (not `pip`/`venv`).
- Run `{test command}` before declaring done.
- Update `docs/{relevant}.md` when public API changes.

---

## 🧭 Conventions

<!-- Code-style choices that disagree with the language default. Skip standard idioms. -->

- Comments tagged `AIDEV-*` are intentional anchors — preserve when refactoring.
- {other intentional deviations}

---

## 📦 Stack (inferred — single line each, no narration)

- Lang: {python 3.12} · Pkg: {uv} · Test: {pytest -q} · Lint: {ruff}
- Cloud: {AWS us-east-1, acct 765894972596} · Compute: {EKS} · DB: {Aurora PG}
- CI: {Jenkins on AL2023} · Container: {Dockerized agent, label cdms}

---

## 🚫 Out of Scope for the Agent

<!-- Hard "don't do" list. Saves the agent from helpful but wrong moves. -->

- Don't run migrations against {prod-db}.
- Don't regenerate `{lockfile}` without `--no-dev`.
- Don't touch `{legacy-dir}/` — owned by {team}, separate review process.

---

## 📚 Deeper Context (optional reads)

<!-- Link out instead of inlining. Keeps this file lean. -->

- Architecture: `docs/agent-context/architecture.md`
- Runbooks: `docs/agent-context/runbooks/`
- Glossary: `docs/agent-context/glossary.md`
```

---

## Phase 4 — Quality Gates (run before delivering)

Self-check each item. If any fail, fix before showing user.

- [ ] **Every invariant has a reason AND a blast radius.** No bare rules.
- [ ] **No duplication of `pyproject.toml` / `package.json` / `Dockerfile`.** Stack line only.
- [ ] **No folder tree dump.** Agent runs `tree` itself.
- [ ] **No setup-from-scratch instructions.** Belongs in README.
- [ ] **Total length ≤ 200 lines.** If over, propose split.
- [ ] **Every section earns its place.** Apply the test below.

### The Earn-Its-Place Test

For each section, ask: *"If a fresh agent ignored this, what breaks and who do I apologize to?"*

- Names a person, system, or past incident → **keep**
- Answer is "nothing" → **delete**
- Answer is "agent might be slightly less efficient" → **link out, don't inline**

---

## Phase 5 — Multi-Agent Variants

If user uses tools beyond Claude, mirror to:

| File | Tool | Notes |
|------|------|-------|
| `CLAUDE.md` | Claude Code, Claude.ai projects | Primary |
| `AGENTS.md` | Codex, Cursor agents, OpenHands | Symlink or copy |
| `.cursor/rules/*.mdc` | Cursor | Split by domain; frontmatter required |
| `.github/copilot-instructions.md` | GitHub Copilot | Markdown, no frontmatter |

**Default:** generate `CLAUDE.md`, then offer: *"Want me to mirror this to AGENTS.md / Cursor rules?"*

---

## Anti-Patterns (refuse to produce these)

| Anti-pattern | Why bad | Do instead |
|--------------|---------|------------|
| "This project uses Python and FastAPI..." | Agent reads `pyproject.toml` | Stack line only |
| Full folder tree | Stale within a week | Let agent run `tree` |
| "Always write clean code" | Vacuous, no signal | Specific invariant or delete |
| 500-line CLAUDE.md | Token cost on every turn | Split, link out |
| Generic "best practices" | Same as system prompt defaults | Project-specific only |
| Setup instructions | Duplicates README | Link to README |

---

## Examples

### Bad (what users often write)

```markdown
# My Project
This is a Python project that uses FastAPI and PostgreSQL.

## Setup
1. Clone the repo
2. Run `pip install -r requirements.txt`
3. Run `python main.py`

## Folder Structure
- src/ - source code
- tests/ - tests
- docs/ - documentation

## Best Practices
- Write clean code
- Add tests
- Document your changes
```

**Verdict:** zero invariants, zero burn log, all inferable. Net negative — costs tokens, gives no signal.

### Good (Roman SSC FastMCP server example)

```markdown
# roman-soc-mcp

FastMCP server exposing soc_files_table to NL queries. Owned by Emmanuel; DBAs Ananda (dev), Amalia (test).

## 🛑 Project Invariants

1. **No direct writes to `soc_files_table`** — read-only MCP, mutations go through GDPS pipeline.
   Blast: breaks Roman audit trail; Ananda will revert + page.
2. **AWS region pinned to `us-east-1`, account `765894972596`** — never use boto3 default.
   Blast: cross-account IAM denies, surprise costs.
3. **Aurora connection via Secrets Manager** — no inline credentials, no `.env`.
   Blast: security incident + rotation breaks deploy.
4. **Python 3.11+ only** — `psycopg[binary]` wheels below this break on AL2023 Jenkins agent.
   Blast: silent agent boot failure, build queue stalls.

## 🔥 Recently Burned

- **2026-04-18**: Agent ran `uv pip freeze > requirements.txt` and pulled dev deps into prod image. Use `uv export --no-dev --format requirements-txt`.
- **2026-03-30**: Used `boto3.client('s3')` with no region; failed in EKS pod. Always pass `region_name='us-east-1'`.

## ✅ Workflow

- Confirm plan before edits touching > 1 file.
- Run `uv run pytest -q` before declaring done.
- New MCP tool → update `docs/agent-context/tools.md`.

## 📦 Stack

- Lang: Python 3.12 · Pkg: uv · Test: pytest · Lint: ruff
- Cloud: AWS us-east-1 / 765894972596 · Compute: EKS · DB: Aurora PG
- Server: FastMCP · CI: Jenkins (label `cdms`, Dockerized agent)
```

**Verdict:** every line earns its place. Agent now knows what to fear and what's intentional.

---

## Output Format

When invoked, respond in this order:

1. **Mode detected** (NEW / UPGRADE / AUDIT) — one line.
2. **Up to 3 questions** if invariants unknown — single message, use `ask_user_input_v0` if mobile.
3. **Inference summary** — bullet list of what was pulled from repo/memory.
4. **Generated CLAUDE.md** — in artifact if creating file, inline diff if upgrading.
5. **Quality gate results** — checklist with ✅/❌.
6. **Next action** — one sentence ("Mirror to AGENTS.md?" or "Want me to commit this?").

---

## Skill Boundary

- ✅ Use for: CLAUDE.md, AGENTS.md, Cursor rules, Copilot instructions, project-level agent memory.
- ❌ Don't use for: end-user README (→ `readme-driven-dev`), API docs, architecture deep-dives (→ `emmanuel-markdown`).
- 🤝 Composes with: `readme-driven-dev` (run after, not instead), `emmanuel-engineering` (for invariant phrasing).

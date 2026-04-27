# AGENTS.md
*Last updated: 2026-04-21 — ejoliet / IPAC Caltech*

> **Purpose** — Onboarding manual for every AI assistant (Claude Code, Cursor, GPT, etc.) and every human who edits this repository.  
> Sections marked **🔧 RDD** are placeholder blocks: fill them in during README-Driven Development for each new project.  
> Sections with no marker are reusable as-is across projects.

---

## 0. Project Overview

> 🔧 **RDD** — Replace this block with a 4–6 line description of the project: what it does, who/what calls it, and the key components. Reference the Phase 0 intent-gathering table in `readme-driven-dev/SKILL.md`.

**Template:**
```
<project-name> is a <one-liner>.

Key components:
- **<component-a>**: <what it does>
- **<component-b>**: <what it does>
- **<component-c>**: <what it does>

Golden rule: When unsure about implementation details, consult the developer rather than assuming.
```

**This repo (claude-skills):**  
`claude-skills` is Emmanuel Joliet's personal skill library for Claude Code.  
Skills encode engineering style, document templates, and workflow patterns used at IPAC Caltech.  
Read `CLAUDE.md` at the start of every session for the full skills index and trigger rules.

---

## 1. Non-Negotiable Golden Rules

> These rules are **reusable as-is**. Adapt column 2 if the project has stricter boundaries.

| # | AI *may* do | AI *must NOT* do |
|---|-------------|-----------------|
| G-0 | When unsure about something project-specific, ask the developer before making changes. | ❌ Write code or call tools when lacking context for a feature or decision. |
| G-1 | Generate code inside relevant source directories or explicitly pointed files. | ❌ Touch test files, spec files, or generated files unless explicitly instructed. |
| G-2 | Add/update `AIDEV-NOTE:` anchor comments near non-trivial edited code. | ❌ Delete or modify existing `AIDEV-` comments without explicit instruction. |
| G-3 | Follow the project's lint/style config (ruff, mypy, eslint, etc.). | ❌ Reformat code to any other style. |
| G-4 | For changes > 300 LOC or > 3 files, ask for confirmation first. | ❌ Refactor large modules without human guidance. |
| G-5 | Stay within the current task context. Flag if a fresh session would be cleaner. | ❌ Continue unrelated prior-session work without a reset. |

---

## 2. Build, Test & Utility Commands

> 🔧 **RDD** — Fill this section during project setup. List the exact commands a new contributor needs. Use a task runner (`poe`, `make`, `just`, `npm run`) for consistency.  
> Reference: `git-workflow/SKILL.md` for pre-push validation commands; `tdd/SKILL.md` or `senior-qa/SKILL.md` for test runner setup.

**Template:**
```bash
# Install
uv sync / pip install -e ".[dev]" / npm install

# Lint & format
ruff format .          # format
ruff check .           # lint
mypy src/              # type check

# Test
pytest                                  # all tests
pytest tests/unit/ -v                   # unit only
pytest --cov=src --cov-report=term      # with coverage

# Run
python -m src.server                    # or: uvicorn src.main:app --reload

# Codegen / migrations (if applicable)
<command>
```

---

## 3. Coding Standards

Default standards for all Emmanuel Joliet projects. Override individual items per-project in a local `AGENTS.md`.

### Language & Runtime
- **Python**: 3.12+ preferred; 3.11 minimum
- **Async**: `async/await` everywhere I/O is involved; no sync calls from async context
- **Typing**: strict — Pydantic v2 models, `from __future__ import annotations`, `mypy --strict` passes

### Formatting & Lint
- **Formatter**: `ruff format` (96-char lines, double quotes, sorted imports)
- **Linter**: `ruff check` — standard rule set; add `# noqa` only with a comment explaining why
- **Type checker**: `mypy --strict` (Python) or `pyright` (TypeScript)

### Naming
| Construct | Convention |
|-----------|-----------|
| Functions / variables | `snake_case` |
| Classes | `PascalCase` |
| Constants | `SCREAMING_SNAKE` |
| MCP tool names | `snake_case`, verb-first (`search_papers`, `get_file`) |
| Files | `snake_case.py` |

### Error Handling
- Typed, hierarchical exceptions defined in `exceptions.py`
- Catch specific exceptions, never bare `except Exception`
- Context managers for all resources (DB connections, file handles, HTTP clients)
- `try/finally` in async code to ensure cleanup

### Documentation
- Google-style docstrings for public functions and classes
- Inline comments only when the *why* is non-obvious — never narrate what the code does
- No multi-paragraph docstrings or block-comment walls

> 🔧 **RDD** — Add project-specific overrides below this line if the project deviates (e.g., different line length, different framework conventions, TypeScript-only).

---

## 4. Repository Layout

> 🔧 **RDD** — Replace with an annotated directory tree for the new project. Keep to two levels of depth; add a `←` annotation for every non-obvious directory.  
> Reference: `codebase-onboarding/SKILL.md` for the `codebase_analyzer.py` script that auto-generates this.

**Template:**
```
<project-root>/
├── AGENTS.md                  ← this file
├── README.md                  ← RDD spec / user-facing docs
├── src/
│   ├── <module>/              ← core logic
│   │   ├── __init__.py
│   │   ├── server.py          ← entrypoint
│   │   ├── tools.py           ← MCP tool definitions (if MCP server)
│   │   └── exceptions.py      ← typed exception hierarchy
│   └── config.py              ← env-var config (pydantic-settings)
├── tests/
│   ├── unit/
│   └── integration/
├── docker-compose.yml
├── Dockerfile
├── pyproject.toml
└── .env.example
```

**This repo (claude-skills):**
```
claude-skills/
├── AGENTS.md                  ← this file (also the reusable template)
├── CLAUDE.md                  ← skills index + trigger rules
├── skills.json                ← machine-readable manifest
├── <skill-name>/
│   ├── SKILL.md               ← skill definition with YAML frontmatter
│   ├── references/            ← supporting reference documents
│   └── scripts/               ← runnable helper scripts (optional)
└── readme-driven-dev/
    └── references/
        └── rdd-template.md    ← RDD base template
```

---

## 5. Anchor Comments

Add specially formatted comments throughout the codebase for AI and human readers. Easily `grep`-able.

### Tags
| Tag | Use for |
|-----|---------|
| `AIDEV-NOTE:` | Non-obvious constraint, invariant, or known limitation |
| `AIDEV-TODO:` | Known gap or deferred work — must include a ticket/issue ref |
| `AIDEV-QUESTION:` | Unresolved design decision needing human input |

### Rules
- Keep to ≤ 120 characters
- Before scanning files, first `grep -r "AIDEV-" .` to find existing anchors
- **Update** relevant anchors when modifying associated code
- **Never delete** an `AIDEV-NOTE` without explicit human instruction
- Add an anchor whenever code is: too long, too complex, very important, confusing, or has a potential bug unrelated to the current task

```python
# AIDEV-NOTE: perf-hot-path; avoid extra allocations — see ADR-24
async def render_feed(...):
    ...

# AIDEV-NOTE: uses lsdb spatial join — see lsdb/SKILL.md for chunking strategy
def cross_match_catalogs(...):
    ...

# AIDEV-TODO: replace simpleeval with safer sandbox — issue #42
def evaluate_expression(expr: str) -> Any:
    ...
```

---

## 6. Commit Discipline

> Full conventions in `git-workflow/SKILL.md`. Summary below.

- **Conventional Commits**: `type(scope): summary` — types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `ci`, `perf`, `build`
- **Granular commits**: one logical change per commit
- **AI-generated commits**: tag with `[AI]` — e.g., `feat(tools): add search_papers tool [AI]`
- **Breaking changes**: append `!` to type — `feat(api)!: rename endpoint` — triggers major bump
- **No secrets in commits**: run `git diff --staged` before every commit; use `.env.example` not `.env`
- **Worktrees** for parallel or long-running AI branches: `git worktree add ../wip-foo -b wip-foo`
- **Review before merge**: never merge AI-generated code you don't understand

For changelog generation from commit history: `changelog-generator/SKILL.md`.

---

## 7. Domain-Specific Patterns

> 🔧 **RDD** — Fill with project-specific implementation patterns: API route structure, database query patterns, message formats, data models, workflow step conventions, etc.  
> For MCP servers, reference `mcp-builder/SKILL.md` for tool schema patterns.  
> For VO/astronomy queries, reference `vo-explorer/SKILL.md`.

**Template:**
```
### <Pattern name>
- Where: `src/<module>/`
- Pattern: <description>
- Example: <code snippet>
- Do NOT: <common mistake>
```

---

## 8. Testing

Default test stack. Override per project.

| Layer | Default tool | Trigger skill |
|-------|-------------|--------------|
| Unit / integration (Python) | `pytest` + `pytest-asyncio` | `tdd/SKILL.md` |
| Mocking HTTP | `respx` (httpx) / `moto` (AWS) | — |
| React / Next.js components | Jest + React Testing Library | `senior-qa/SKILL.md` |
| E2E (web) | Playwright | `senior-qa/SKILL.md` |
| Coverage gate | ≥ 80% (lines + branches) | `senior-qa/SKILL.md` |
| MCP tool evaluation | custom harness | `mcp-builder/SKILL.md` |

### Rules
- Tests live in `tests/` and are **human-owned** — AI generates stubs only
- Never hit real external APIs in unit or CI tests; use mocks/fixtures
- One test file per source module; mirror the `src/` structure
- Use descriptive test names: `test_search_returns_empty_list_when_no_results`

> 🔧 **RDD** — Add project-specific test runner commands and any non-standard fixtures or test data setup below this line.

---

## 9. Directory-Specific AGENTS.md Files

- **Always check for an `AGENTS.md`** in the specific directory before working on code within it
- If a directory's `AGENTS.md` is outdated or wrong, **update it**
- After significant changes to a directory's structure or patterns, **document them** in its `AGENTS.md`
- If a complex directory lacks an `AGENTS.md`, **suggest creating one**

For this repo, candidate directories for local `AGENTS.md` files:

| Directory | Complexity | Suggested content |
|-----------|-----------|-------------------|
| `senior-prompt-engineer/` | High | RAG eval metrics, prompt pattern index |
| `vo-explorer/` | High | ADQL gotchas, service timeout defaults |
| `changelog-generator/` | Medium | CI integration steps, monorepo scope rules |
| `engineering/` | High | AWS defaults, EKS manifest conventions |

---

## 10. Common Pitfalls

> 🔧 **RDD** — Add project-specific pitfalls discovered during development. Start with the general list below and append as the project matures.

### General (all projects)
- Large AI refactors in a single commit — makes `git bisect` unusable
- Delegating test/spec writing entirely to AI — leads to false confidence
- Deleting `AIDEV-NOTE` anchors without reading why they existed
- Committing `.env` or credentials — use `.env.example` and secrets manager
- Mixing async and sync DB/HTTP calls in the same code path
- Forgetting `from __future__ import annotations` when using self-referential types

### This repo (claude-skills)
- Updating `CLAUDE.md` skill index without updating `skills.json` (or vice versa)
- Adding a skill directory without an `AGENTS.md` for complex skills
- Writing trigger rules that are too broad — causes unintended skill loads
- Skipping the `references/` directory for skills with rich domain content

---

## 11. Versioning Conventions

Semantic Versioning (`MAJOR.MINOR.PATCH`) for all components.

| Bump | When |
|------|------|
| `MAJOR` | Incompatible API or interface changes |
| `MINOR` | New backward-compatible functionality |
| `PATCH` | Bug fixes, doc updates, internal refactors |

Changelog generation from commit history: `changelog-generator/SKILL.md`.  
Commit linting before tagging: `python changelog-generator/scripts/commit_linter.py --strict`.

> 🔧 **RDD** — If the project has independent sub-components with their own versions (monorepo), document per-package versioning strategy here and reference `changelog-generator/references/monorepo-strategy.md`.

---

## 12. Key File & Pattern References

> 🔧 **RDD** — Fill with pointers to the most important files in the project. Aim for ≤ 10 entries. Include: entrypoint, config, exception hierarchy, key domain model, and any generated/auto-managed files.

**Template:**

| File / Directory | Role | Key symbols / notes |
|-----------------|------|-------------------|
| `src/<module>/server.py` | Entrypoint | `app`, `lifespan` |
| `src/<module>/tools.py` | MCP tool definitions | `@mcp.tool()` decorators |
| `src/<module>/exceptions.py` | Exception hierarchy | Base: `AppError` |
| `src/config.py` | Env-var config | `Settings` (pydantic-settings) |
| `autogen/` | **Do not edit** — generated from TypeSpec/OpenAPI | — |
| `.env.example` | Env-var reference | Copy to `.env`, never commit `.env` |

---

## 13. Domain Terminology

> 🔧 **RDD** — Add project-specific terms below the astronomy glossary. If the project is not astronomy-related, replace the whole section.

### Astronomy / IPAC defaults (pre-filled)

| Term | Definition |
|------|-----------|
| **TAP** | Table Access Protocol — IVOA-standard SQL-over-HTTP for catalog queries |
| **ADQL** | Astronomical Data Query Language — SQL dialect for sky coordinates |
| **ObsCore** | IVOA data model for observational metadata |
| **MOC** | Multi-Order Coverage map — HEALPix-based spatial footprint |
| **HiPS** | Hierarchical Progressive Survey — tiled sky image/catalog format |
| **VOTable** | XML serialization format for VO query results |
| **ASDF** | Advanced Scientific Data Format — YAML+binary used by Roman and JWST |
| **HATS** | Hierarchical Adaptive Tiling Scheme — partitioning format for billion-row catalogs |
| **WFI** | Wide Field Instrument — Roman Space Telescope's primary detector array |
| **SOC** | Science Operations Center — manages Roman data delivery at IPAC |
| **CRDS** | Calibration Reference Data System — manages reference files for pipeline |
| **HLWAS / HLTDS / GBTDS** | Roman High Latitude surveys (Wide Area, Deep Time-Domain, Galactic Bulge) |
| **SkyCoord** | Astropy coordinate object; canonical way to represent celestial positions |
| **FastMCP** | Python framework for building MCP servers — Emmanuel's default |

### Project-specific terms

> 🔧 **RDD** — Add project-specific terms here.

---

## 14. Files to NOT Modify

> 🔧 **RDD** — List files that should never be hand-edited (generated, policy-controlled, or owned by other systems). Include why for each.

**Template:**

| File / Pattern | Reason |
|---------------|--------|
| `autogen/**` | Generated from TypeSpec — overwritten on next codegen run |
| `*.lock` | Managed by package manager — use `uv lock` / `npm install` |
| `migrations/` | DB migrations are append-only; never edit existing files |
| `.agentignore` | Controls AI tool indexing — change only with explicit permission |

**This repo (claude-skills):**

| File | Reason |
|------|--------|
| `skills.json` | Must stay in sync with `CLAUDE.md` — update both together |

---

## 15. Meta: Guidelines for Updating AGENTS.md Files

### When to update
- After any significant change to directory structure, naming conventions, or tooling
- When a new common pitfall is discovered
- When a domain term is introduced or redefined
- When a build command changes

### What makes a good AGENTS.md entry
1. **Decision flowcharts** — "when to use X vs Y" for key architectural choices
2. **Reference links** — pointers to canonical files or implementation examples
3. **Domain glossary** — project-specific terms an AI won't know from training data
4. **Versioning rules** — how the project handles API and component versioning
5. **Tabular format** — structured data in tables, not prose paragraphs
6. **Consistent code block language tags** — always specify `python`, `bash`, `yaml`, etc.

### What does NOT belong here
- Content already in `CLAUDE.md` or a skill's `SKILL.md`
- Ephemeral task state or in-progress work notes
- Information derivable from reading the code

---

## AI Assistant Workflow: Step-by-Step Methodology

When responding to a user instruction, follow this process:

1. **Consult guidance** — Read this `AGENTS.md` and any directory-specific `AGENTS.md` relevant to the task. Check `CLAUDE.md` for applicable skills.
2. **Clarify ambiguities** — Identify what cannot be inferred. Ask ≤ 2 questions, batched in one message.
3. **Break down and plan** — Decompose the task; reference project conventions and skill trigger rules.
4. **Trivial tasks** — Proceed immediately if the plan is straightforward.
5. **Non-trivial tasks** — Present the plan for review; iterate before executing.
6. **Track progress** — Use a to-do list for multi-step tasks. Do not batch completions.
7. **If stuck, re-plan** — Return to step 3; do not force a solution that doesn't fit.
8. **Update documentation** — After completing the task, update `AIDEV-NOTE` anchors and any `AGENTS.md` files in touched directories.
9. **User review** — Ask the user to review the result; repeat as needed.
10. **Session boundaries** — If the request is unrelated to the current context, suggest starting a fresh session.

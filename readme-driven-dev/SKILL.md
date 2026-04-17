---
name: readme-driven-dev
description: >
  README-Driven Development (RDD) skill for Emmanuel at IPAC Caltech. Generates
  a single README.md that simultaneously serves as the public-facing repository
  README AND as a complete, unambiguous specification for a coding agent to
  implement the tool from scratch — no other design doc needed.

  Trigger this skill whenever Emmanuel says: "RDD for X", "create a README-driven
  spec", "write a README as the spec", "README-first for my tool", "agent-ready
  README", "scaffold from README", or any request where a project README should
  also serve as the build blueprint. Also trigger when asked to bootstrap a new
  tool, CLI, service, or library and there's no existing spec.

  Do NOT use for post-hoc documentation of already-built code, pure ADRs,
  runbooks, or meeting notes.
---

# README-Driven Development (RDD) Skill

Produces a **single README.md** that is simultaneously:
1. A polished, public-facing project README
2. A complete agent build specification — no ambiguity, no hand-waving

The agent reading this README should be able to implement the tool **without asking
any follow-up questions**.

---

## Principles

| Principle | What it means |
|-----------|---------------|
| **Spec-first** | README is written before a single line of code |
| **Agent-complete** | Every section answers a question an agent would ask |
  **No orphan prose** | Every description is backed by a schema, signature, or example |
| **Dual audience** | Human readers get context; agents get contracts |
| **Honest scope** | Non-goals are explicit; phase boundaries are enforced |

---

## Phase 0 — Gather Intent (before writing anything)

Ask these questions if not already answered in the conversation:

1. **What does this tool do?** (one sentence)
2. **Who/what calls it?** (human CLI, another service, an agent, a DAG, a test)
3. **What are the hard inputs and outputs?** (files, API calls, DB tables, events)
4. **What is explicitly OUT of scope for v1?**
5. **What is the target runtime?** (Python version, container, Lambda, EKS job…)
6. **What credentials/env vars are required?**
7. **Is there an existing repo structure to match?** (monorepo, standalone, etc.)

Do not proceed to writing until you have clear answers (or can infer them from context).

---

## Phase 1 — Produce the README

Generate the full README.md using the template in `references/rdd-template.md`.

### Mandatory sections (never omit)

| Section | Purpose |
|---------|---------|
| **Header block** | Name, one-liner, badges (CI, coverage, license) |
| **Purpose** | Problem → solution → who benefits |
| **Architecture** | Component diagram or data-flow bullets |
| **Repository Layout** | Exact directory tree with annotations |
| **Prerequisites** | Tool versions, AWS perms, env vars (table format) |
| **Quick Start** | Clone → configure → run — copy-pasteable |
| **Configuration Reference** | Every env var: name, type, default, description |
| **API / Interface Contract** | Signatures, schemas, CLI flags — machine-readable |
| **Data Model** | DB tables, Pydantic models, or JSON schemas |
| **Error Handling** | Named error classes, exit codes, retry behavior |
| **Testing** | How to run tests; what each test suite covers |
| **Deployment** | Exact commands per target (local / EKS / Lambda…) |
| **Agent Build Instructions** | See Phase 2 below |
| **Non-Goals** | What v1 explicitly does NOT do |
| **Open Questions** | Unresolved decisions — block the agent from guessing |
| **Next Steps** | Ordered task list an agent should execute |

Optional (include when relevant):

- `## Observability` — metrics, logs, alerts
- `## Security` — IAM, secrets, network surface
- `## Performance` — SLAs, throughput targets
- `## Changelog` — once code exists

---

## Phase 2 — Agent Build Instructions section

This is the section that transforms a README into a build spec.
It lives inside the README under `## Agent Build Instructions`.

Include all of the following sub-sections:

```markdown
## Agent Build Instructions

> This section is the authoritative build specification.
> A coding agent should be able to implement this tool end-to-end
> using only this README — no clarifying questions needed.

### Build Order

Ordered list of implementation phases. Each phase must be independently
testable before the next begins.

| Phase | Deliverable | Done when |
|-------|-------------|-----------|
| 0 | Repo scaffold + CI skeleton | `make lint` passes on empty project |
| 1 | Data model + DB migrations | Unit tests pass for all schema ops |
| 2 | Core logic | Integration test passes with fixture data |
| 3 | Interface layer (CLI/API/MCP) | Contract tests pass |
| 4 | Observability | Metrics emit to stdout in local mode |
| 5 | Deployment config | `make deploy-dry-run` succeeds |

### File Map

Explicit mapping of every source file the agent must create:

| File | Purpose | Key symbols |
|------|---------|-------------|
| `src/models.py` | SQLAlchemy/Pydantic models | `class Foo(Base)` |
| `src/core.py` | Business logic | `def process(...)` |
| `src/cli.py` | Typer CLI entry point | `app = typer.Typer()` |
| `tests/test_core.py` | Core logic tests | `test_process_happy_path` |
| `Makefile` | Dev commands | `lint`, `test`, `deploy-dry-run` |

### Contracts (machine-readable)

Include at minimum one of: OpenAPI spec snippet, Pydantic model, CLI --help
output, or JSON Schema. This is what the agent uses to verify its output.

\```python
class InputSchema(BaseModel):
    field_a: str
    field_b: int = 0
\```

### Constraints

Hard rules the agent must not violate:

- Python 3.11+
- No synchronous DB calls from async context
- All secrets via environment variables — never hardcoded
- Typed signatures on all public functions
- Tests must not hit real AWS endpoints

### Acceptance Criteria

Checklist the agent runs against before declaring done:

- [ ] `make test` passes with ≥80% coverage
- [ ] `make lint` passes (ruff + mypy)
- [ ] `make deploy-dry-run` exits 0
- [ ] README Quick Start works on a clean machine
- [ ] All Open Questions resolved or explicitly deferred to v2
```

---

## Phase 3 — Self-check before delivering

Run through this checklist before returning the README:

- [ ] Every function/command mentioned has a concrete signature or example
- [ ] Every env var is in the Configuration Reference table
- [ ] No section says "TBD" or "as needed" — use **Open Questions** instead
- [ ] The Build Order phases are sequential and independently testable
- [ ] Non-Goals are specific, not vague ("No auth" not "Minimal scope")
- [ ] Quick Start can be copy-pasted verbatim and work
- [ ] The File Map covers every file the agent needs to create

---

## Formatting Rules (inherits from emmanuel-markdown)

- Max 3 heading levels (H1/H2/H3)
- Tables for all structured data (config, phases, files)
- Fenced code blocks with language tags
- `> ⚠️` callouts for blockers; `> 💡` for design rationale
- No padding — omit sections that don't apply yet
- End with `## Next Steps` as an ordered agent task list

---

## References

- `references/rdd-template.md` — Full README template (copy/fill pattern)
- `references/examples.md` — Annotated example RDD READMEs from prior projects

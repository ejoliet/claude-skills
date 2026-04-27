---
name: readme-driven-dev
description: >
  README-Driven Development (RDD) skill for Emmanuel at IPAC Caltech. Generates
  a README.md matched to the exact type needed: spec-only, runnable build package
  (with working code + integration config), or public OSS README.

  Trigger whenever Emmanuel says: "RDD for X", "create a README-driven spec",
  "README-first for my tool", "agent-ready README", "scaffold from README", or
  any request to bootstrap a new tool, CLI, service, or library. Also trigger
  for casual requests like "help me build X" or "spin up a Y" when no spec exists.
  Always use this skill before writing any code.
---

# README-Driven Development (RDD) Skill

Produces a README.md that is:
1. Matched to the **correct type** for the use case (spec / build package / OSS)
2. Grounded in **real-world stack popularity data** researched at generation time
3. Complete enough that an agent can implement without follow-up questions
4. Includes **usage examples** and **multi-platform installation** for every MCP server

---

## Phase 0 — Gather Intent (minimal, context-first)

**Before asking anything**, scan the conversation and infer:

| # | Question | How to infer |
|---|----------|-------------|
| 1 | What does this tool do? | User's description |
| 2 | Who/what calls it? | Interface mentioned (CLI, MCP client, API, DAG…) |
| 3 | Key inputs and outputs? | Data, files, APIs mentioned |
| 4 | Runtime target? | Python version, container, EKS, Lambda… |
| 5 | Required credentials/env vars? | Services mentioned |
| 6 | Existing repo structure to match? | Monorepo, standalone, org conventions |
| 7 | README type needed? | See type table below — infer from context |

### README Type Selection

| Type | When to use | What it includes |
|------|-------------|-----------------|
| **A — Spec** | Agent will implement from scratch; no code yet needed | Architecture, contracts, file map, acceptance criteria |
| **B — Build package** | User wants something runnable fast | Working code stubs, Claude Project Instructions, Docker Compose, `mcp_config.json`, usage examples, multi-platform install |
| **C — OSS public** | Repo going public | Badges, quickstart, contribution guide, no internal details |

**Default**: Type B if the tool is an MCP server or the user says "get it running". Type A otherwise.

**Rule**: Only ask what cannot be inferred. Target ≤ 2 questions, batched into one message. Never ask sequentially.

### Stack → Section Inference

After Phase 1, automatically add these sections based on stack:

| Stack element | Implied extra sections |
|---------------|----------------------|
| FastMCP / MCP server | Claude Project Instructions, tool routing table, `mcp_config.json`, Usage Examples, Multi-platform Installation |
| Docker | Dockerfile, Docker Compose (if multi-service) |
| Docker + external services | `docker-compose.yml` with all services wired |
| FastAPI | OpenAPI snippet, `/docs` URL |
| Airflow | DAG structure, connection config |
| EKS / Helm | Helm values snippet, K8s manifest |
| AWS Lambda | SAM/CDK snippet, event source config |

---

## Phase 1 — Stack Research (required, run before writing)

### 1a. GitHub Popularity
For each major stack layer, find ≥ 3 candidates:
- Collect: stars, last commit date, open issues, weekly PyPI/npm downloads

### 1b. Community Signal
| Source | Query pattern | What to extract |
|--------|--------------|----------------|
| **Hacker News** | `site:news.ycombinator.com "<lib> vs <lib>"` | "We use X in prod" signals |
| **dev.to** | `site:dev.to "<framework> <year>"` | Tutorial recency, engagement |
| **Reddit** | `site:reddit.com/r/Python "<lib>"` | Practitioner opinions |
| **Official changelog** | Latest release date | Maintenance health |

Search ≥ 2 community sources per major stack decision.

### 1c. Stack Decision Table
Produce before writing. Show to Emmanuel for one round of overrides.

```markdown
| Layer | Chosen | Stars | Last Release | Why chosen | Rejected |
|-------|--------|-------|-------------|------------|---------|
```

> 💡 After stack confirmed, apply Stack → Section Inference to determine extra sections.

---

## Phase 2 — Produce the README

Use `references/rdd-template.md` as base, `references/mcp-server-template.md` for MCP servers.

Select variant by type:
- **Type A**: Agent Build Instructions, File Map, Acceptance Criteria; no working code
- **Type B**: Working code stubs, Claude Project Instructions, Docker Compose, `mcp_config.json`, Usage Examples, Multi-platform Installation; lighter Agent Build Instructions
- **Type C**: Quickstart, Contributing, Badges; no internal infra

### Mandatory sections (all types)

| Section | Purpose |
|---------|---------|
| Header block | Name, one-liner, badges |
| Purpose | Problem → solution → who benefits |
| Architecture | Component diagram or data-flow bullets |
| Recommended Stack | Phase 1c table with sources cited |
| Repository Layout | Annotated directory tree |
| Prerequisites | Versions, perms, env vars |
| Quick Start | Copy-pasteable clone → configure → run |
| Configuration Reference | Every env var: name, type, default, required |
| API / Interface Contract | Signatures, schemas, CLI flags |
| Error Handling | Named error classes, retry behaviour |
| Testing | How to run; what each suite covers |
| Non-Goals (v1) | Specific exclusions |
| Open Questions | Unresolved decisions |
| Next Steps | Ordered task list |

### Type B additional sections (MCP servers)

**`## Usage Examples`** (Fix 6) — always include for MCP servers:

```markdown
## Usage Examples

### Natural language → tool → output

Each example: prompt you type to Claude | tool fired | truncated realistic output.

**Example 1 — Topic search**
> "Find the 5 most cited papers on Roman Space Telescope microlensing since 2022"
→ `ads_search_papers(query="abs:microlensing AND property:refereed AND year:2022-2026", rows=5, sort="citation_count desc")`
Output:
\```
## ADS Results — abs:microlensing ...
### Towards a census of galactic microlensing events
- Authors: Johnson, A., Kim, B. et al.
- Year: 2023 | Bibcode: `2023ApJ...945...12J` | Citations: 47
\```

**Example 2 — Deep paper analysis (multi-step)**
> "Explain the methodology of 2021ApJ...914..100E"
1. `ads_get_paper(bibcode="2021ApJ...914..100E")` → confirms arXiv ID `2104.12345`
2. `ads_get_fulltext(bibcode="2021ApJ...914..100E")` → returns "" (paywalled)
3. `ads_fetch_arxiv_pdf(arxiv_id="2104.12345", section="methods")` → returns methods text
4. Claude synthesizes: "The paper uses a forward modelling approach..."

**Example 3 — Author metrics**
> "What's the h-index of Penny, M?"
1. `ads_author_search(author="Penny, M", refereed_only=True)` → list of bibcodes
2. `ads_get_metrics(bibcodes=[...])` → h-index: 18, total citations: 1240
```

**`## Installation`** (Fix 5) — multi-platform matrix, always include for MCP servers:

```markdown
## Installation

### Prerequisites
| Requirement | Version | Notes |
|-------------|---------|-------|
| Python | 3.11+ | |
| uv | latest | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| <Token> | — | <link> |

### Platform integration

#### Claude Code
File: `~/.claude/mcp_servers.json`
\```json
{
  "mcpServers": {
    "<server>": {
      "command": "uv",
      "args": ["run", "--directory", "/path/to/repo", "python", "-m", "src.server"],
      "env": { "TOKEN": "<token>" }
    }
  }
}
\```

#### Claude Desktop — macOS
File: `~/Library/Application Support/Claude/claude_desktop_config.json`
\```json
{ "mcpServers": { "<server>": { ... same as above ... } } }
\```

#### Claude Desktop — Windows
File: `%APPDATA%\Claude\claude_desktop_config.json`
\```json
{ "mcpServers": { "<server>": { ... same as above ... } } }
\```

#### Docker (stdio via exec)
\```json
{
  "mcpServers": {
    "<server>": {
      "command": "docker",
      "args": ["exec", "-i", "<container>", "python", "-m", "src.server"]
    }
  }
}
\```

#### Docker Compose (multi-service)
\```yaml
services:
  <server>:
    build: .
    environment:
      - TOKEN=${TOKEN}
  # other MCP servers / sidecars
\```

#### EKS / Kubernetes (SSE transport)
\```yaml
# values.yaml snippet
env:
  - name: TRANSPORT
    value: "sse"
  - name: PORT
    value: "8080"
\```
Connect via: `"url": "http://<service>:8080/sse"`

#### pip (no uv)
\```bash
pip install -e ".[dev]"
python -m src.server
\```
```

**`## Claude Project Instructions`** (Fix 4):
```markdown
## Claude Project Instructions

Paste into the Claude Project Instructions field.

\```
You are a <role> with access to <server-name>.

### Tool routing
| Tool | When to call |
|------|-------------|

### Reasoning workflow
1. Step → tool
2. Fallback if empty → other tool

### Key facts / query syntax
- ...
\```
```

**`## Docker Compose`** when multi-service.

---

## Phase 3 — Agent Build Instructions (Type A / B)

```markdown
## Agent Build Instructions

> Implement end-to-end using only this README. Resolve Open Questions first.

### Build Order
| Phase | Deliverable | Done when |
|-------|-------------|-----------|
| 0 | Scaffold + CI | `make lint` passes |
| 1 | Config + client | Tests pass with mocks |
| 2 | Tools | Tool tests pass |
| 3 | Server wiring | `--list-tools` correct |
| 4 | Dockerfile + compose | `make deploy-dry-run` exits 0 |

### File Map
| File | Purpose | Key symbols |
|------|---------|-------------|

### Constraints
- Python 3.11+; typed signatures everywhere
- No sync calls from async context
- Secrets via env vars only
- `ruff` + `mypy --strict` pass
- Tests never hit real APIs (respx / moto)

### Acceptance Criteria
- [ ] `make test` passes, coverage ≥ 80%
- [ ] `make lint` passes
- [ ] `--list-tools` lists all tools with docstrings
- [ ] Usage Examples section smoke-tested end-to-end
- [ ] All platforms in Installation section verified
- [ ] All Open Questions resolved or deferred
```

---

## Phase 4 — Self-check before delivering

- [ ] README type matches user intent
- [ ] Stack Decision Table: ≥ 3 candidates, community sources cited
- [ ] Stack → Section Inference applied
- [ ] Usage Examples: ≥ 3 examples (simple search, multi-step workflow, one more)
- [ ] Installation: all relevant platforms covered (Claude Code, Desktop macOS/Windows, Docker, EKS, pip)
- [ ] Claude Project Instructions: tool routing table + reasoning workflow + key facts
- [ ] Docker Compose present if multi-service
- [ ] Every env var in Configuration Reference
- [ ] No "TBD" — use Open Questions
- [ ] Non-Goals are specific

---

## Formatting Rules

- Max 3 heading levels (H1/H2/H3)
- Tables for all structured data
- Fenced code blocks with language tags
- `> ⚠️` for blockers; `> 💡` for design rationale
- No padding — omit sections that don't apply
- End with `## Next Steps`

---

## Research Sources Reference

| Source | URL pattern | Best for |
|--------|-------------|---------|
| GitHub search | `https://github.com/search?q=<topic>&sort=stars&type=repositories` | Popularity |
| HN Algolia | `https://hn.algolia.com/?q=<lib>` | Production signals |
| dev.to | `https://dev.to/search?q=<lib>+<year>` | Adoption curve |
| PyPI stats | `https://pypistats.org/packages/<package>` | Weekly downloads |
| npm trends | `https://npmtrends.com/<pkg-a>-vs-<pkg-b>` | JS/TS comparisons |
| Libraries.io | `https://libraries.io/search?q=<topic>` | Dependency health |

---

## References

- `references/rdd-template.md` — Full README template (copy/fill)
- `references/mcp-server-template.md` — Type B MCP server template (Claude Project Instructions, Installation matrix, Usage Examples, Docker Compose)
- `references/examples.md` — Annotated example READMEs

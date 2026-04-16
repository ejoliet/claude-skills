# CLAUDE.md — claude-skills

This is Emmanuel Joliet's personal skills repository for Claude Code.
Skills encode engineering style, document templates, and workflow patterns.
Read this file at the start of every session.

---

## Skills Index

| Skill | Path | Trigger |
|-------|------|---------|
| `emmanuel-engineering` | `engineering/SKILL.md` | Architecture, AWS/EKS, Python, MCP, pipelines, astronomy |
| `emmanuel-markdown` | `markdown/SKILL.md` | Any `.md` doc, README, ADR, runbook, RFC, post-mortem |
| `readme-driven-dev` | `readme-driven-dev/SKILL.md` | "RDD for X", "agent-ready README", bootstrapping a new tool |
| `context7-docs-lookup` | `context7/SKILL.md` | Any coding task involving a named library or framework |

---

## How to Load a Skill

Before responding to any task, check if a skill applies:

```bash
# Read a skill
cat engineering/SKILL.md
cat markdown/SKILL.md
cat readme-driven-dev/SKILL.md
cat context7/SKILL.md
```

**Multiple skills can apply.** For example, "RDD for a new FastMCP server" triggers
`readme-driven-dev` + `emmanuel-engineering` + `context7-docs-lookup`. Read all that apply.

---

## Skill Trigger Rules

### `emmanuel-engineering` — read when:
- Architecture decisions, system design, infrastructure planning
- Any AWS/EKS/Airflow/Jenkins/Aurora PostgreSQL work
- Python backends, FastAPI, FastMCP server development
- CI/CD pipelines, Docker, Kubernetes, Helm
- Astronomical data access (pyvo, astroquery, IRSA, MAST)
- VO protocols (TAP, SIA, ADQL, ObsCore, VOTable, FITS, ASDF)
- IaC (CloudFormation, Custodian, Terraform, CDK)
- Casual: "design X", "build me Z", "spin this up"
- **Do NOT trigger** for pure writing, trivia, non-technical requests

### `emmanuel-markdown` — read when:
- Any `.md` document requested
- README, ADR, Runbook, RFC/Design Doc, Meeting Notes, Post-Mortem
- "Write a markdown", "create a doc", "document this", "draft a README"
- **Do NOT trigger** for inline chat answers, code-only responses, slides

### `readme-driven-dev` — read when:
- "RDD for X", "create a README-driven spec", "agent-ready README"
- Bootstrapping a new tool, CLI, service, or library with no existing spec
- Any request where README should also be the build blueprint
- **Do NOT trigger** for post-hoc docs of already-built code, ADRs, runbooks

### `context7-docs-lookup` — read when:
- Any coding task, system design, or technical question involving a named library
- FastAPI, SQLAlchemy, boto3, Airflow, FastMCP, Helm, Kubernetes, Terraform, Pydantic, etc.
- "How do I configure X", "what's the syntax for Y", version migration questions
- **Trigger even when you think you know the answer** — training data may be stale

---

## Active Systems Context

| System | Stack | Key details |
|--------|-------|-------------|
| Roman SOC file exchange | FastMCP + Aurora PG + EKS | `soc_files_table`; GraphQL layer |
| ADS MCP Server v2 | FastMCP + pypdf + httpx | ADS search, ArXiv PDF parsing |
| Transient broker dashboard | HTML + Aladin Lite | GCN, ALeRCE, Fink, ANTARES, LASAIR |
| Jenkins EC2 | AL2023; CloudWatch CPU alarms | SNS: `jenkins-cpu-alerts`; 40% threshold |
| Airflow on EKS | REST API configured | Pipeline orchestration |
| Missions | Roman, SPHEREx, Euclid, NEOWISE | IPAC/Caltech; IRSA access patterns |

**AWS account**: `765894972596` / `us-east-1`
**Author**: Emmanuel Joliet (`ejoliet`) / IPAC Caltech

---

## Output Standards

- **Architecture first** — never jump to code without a design
- **Trade-off tables** whenever multiple approaches exist
- **Phased plans** with clear exit criteria per phase
- **Docker artifacts** on every new service/API/notebook (goal: `docker compose up` works)
- **Production-grade defaults** — no toy examples
- **Honest scope** — state what's out of scope for v1

---

## Repo Layout

```
claude-skills/
├── CLAUDE.md                        ← this file
├── skills.json                      ← machine-readable manifest
├── engineering/
│   ├── SKILL.md
│   └── references/
│       ├── mcp-patterns.md
│       ├── aws-defaults.md
│       ├── kubernetes-manifests.md
│       └── astronomy-patterns.md
├── markdown/
│   └── SKILL.md
├── readme-driven-dev/
│   ├── SKILL.md
│   └── references/
│       ├── rdd-template.md
│       └── examples.md
└── context7/
    └── SKILL.md
```

---

## Adding New Skills

1. Create `<skill-name>/SKILL.md` with YAML frontmatter (`name`, `description`)
2. Add trigger rules to the **Skills Index** table above
3. Update `skills.json` with the new entry
4. If the skill has reference files, add them under `<skill-name>/references/`

# <Tool Name>

> <One-sentence description: what it does and why it exists.>

![CI](https://github.com/<org>/<repo>/actions/workflows/ci.yml/badge.svg)
![Coverage](https://img.shields.io/badge/coverage-XX%25-brightgreen)
![Python](https://img.shields.io/badge/python-3.11+-blue)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## Purpose

**Problem**: <What is broken, missing, or painful today?>

**Solution**: <What this tool does to fix it.>

**Scope**: <Who/what uses this tool and in what context (e.g., Roman SOC pipeline,
local dev, Lambda, Airflow DAG).>

---

## Architecture

```
<ASCII or Mermaid diagram>
```

**Data flow**:
- `<Source>` → `<Component>` → `<Sink>`
- `<Source>` → `<Component>` → `<Sink>`

**Key components**:

| Component | Responsibility |
|-----------|---------------|
| `<module>` | <what it owns> |
| `<module>` | <what it owns> |

---

## Repository Layout

```
<repo-name>/
├── src/
│   ├── __init__.py
│   ├── models.py          # Pydantic / SQLAlchemy models
│   ├── core.py            # Business logic
│   ├── cli.py             # Typer CLI entrypoint
│   └── <other>.py
├── tests/
│   ├── conftest.py        # Fixtures
│   ├── test_core.py       # Unit tests
│   └── test_integration.py
├── infra/                 # IaC (CloudFormation / CDK / Helm values)
├── Makefile
├── pyproject.toml
├── Dockerfile
└── README.md              # This file
```

---

## Prerequisites

| Requirement | Version / Value | Notes |
|-------------|-----------------|-------|
| Python | 3.11+ | |
| AWS CLI | 2.x | Configured for `us-east-1` |
| kubectl | 1.28+ | For EKS targets |
| <tool> | <version> | <why> |

**Required IAM permissions**:
- `<action>` on `<resource>`
- `<action>` on `<resource>`

---

## Quick Start

```bash
# 1. Clone
git clone https://github.com/<org>/<repo>.git
cd <repo>

# 2. Install dependencies
pip install -e ".[dev]"

# 3. Configure
cp .env.example .env
# Edit .env with required values (see Configuration Reference)

# 4. Run
make run
# or
python -m src.cli --help
```

---

## Configuration Reference

All configuration via environment variables. No secrets in code.

| Variable | Type | Default | Required | Description |
|----------|------|---------|----------|-------------|
| `VAR_A` | `str` | — | ✅ | <description> |
| `VAR_B` | `int` | `60` | | <description> |
| `AWS_REGION` | `str` | `us-east-1` | | AWS region |
| `LOG_LEVEL` | `str` | `INFO` | | One of: DEBUG, INFO, WARNING, ERROR |

---

## API / Interface Contract

### CLI

```
Usage: python -m src.cli [OPTIONS] COMMAND [ARGS]...

Commands:
  run      <description>
  status   <description>
  list     <description>

Options:
  --verbose    Enable debug logging
  --dry-run    Validate config and exit
  --help       Show this message and exit.
```

### Python API (if applicable)

```python
from src.core import process

result: OutputSchema = process(
    input_data: InputSchema,
    config: Config,
) -> OutputSchema
```

### MCP Tools (if applicable)

```python
@mcp.tool()
async def tool_name(param_a: str, param_b: int = 0) -> dict:
    """<description shown to agent>"""
    ...
```

---

## Data Model

```python
from pydantic import BaseModel
from datetime import datetime

class <Entity>(BaseModel):
    id: int
    field_a: str
    field_b: datetime
    status: Literal["pending", "done", "failed"]
```

**DB schema** (if applicable):

```sql
CREATE TABLE <table> (
    id          SERIAL PRIMARY KEY,
    field_a     TEXT NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Error Handling

| Error Class | When raised | Exit code | Retry? |
|-------------|-------------|-----------|--------|
| `ConfigError` | Missing required env var | 1 | No |
| `ConnectionError` | DB/API unreachable | 2 | Yes (3x, backoff) |
| `ValidationError` | Invalid input schema | 3 | No |
| `<ToolError>` | <condition> | 4 | <behavior> |

All errors log to stderr with structured JSON: `{"level": "ERROR", "error": "...", "context": {...}}`.

---

## Testing

```bash
make test          # All tests
make test-unit     # Unit only (no AWS/DB)
make test-int      # Integration (requires .env)
make coverage      # HTML coverage report → htmlcov/
```

| Suite | What it covers | Fixtures needed |
|-------|----------------|-----------------|
| `test_core.py` | Pure business logic | None |
| `test_cli.py` | CLI command parsing | None |
| `test_integration.py` | End-to-end with real DB | `.env` with test DB |

> ⚠️ Integration tests must not touch production AWS resources.
> Use `AWS_ENDPOINT_URL=http://localhost:4566` for LocalStack.

---

## Deployment

### Local

```bash
make run
```

### Docker

```bash
docker build -t <tool-name>:local .
docker run --env-file .env <tool-name>:local
```

### EKS (Roman SOC)

```bash
# Dry run
make deploy-dry-run

# Deploy to dev
make deploy ENV=dev

# Deploy to prod
make deploy ENV=prod
```

### Lambda / EventBridge (if applicable)

```bash
make package   # Builds deployment.zip
make deploy-lambda ENV=dev
```

---

## Observability

| Signal | Where | What to watch |
|--------|-------|---------------|
| Logs | CloudWatch `/aws/<service>/<tool>` | ERROR count |
| Metrics | CloudWatch custom namespace | `ProcessedCount`, `ErrorCount` |
| Alerts | SNS → `<topic-arn>` | `ErrorCount > 0` for 5 min |

---

## Non-Goals (v1)

Explicit exclusions — do not implement these:

- <Feature A> — deferred to v2
- <Feature B> — out of scope for this tool
- <Feature C> — handled by <other system>

---

## Open Questions

Block the agent from guessing on these. Resolve before implementing.

- [ ] **Q1**: <question> — owner: <name>, due: <date>
- [ ] **Q2**: <question> — owner: <name>, due: <date>

---

## Agent Build Instructions

> This section is the authoritative build specification.
> A coding agent should implement this tool end-to-end using only this README.
> No clarifying questions needed — all ambiguity goes in Open Questions above.

### Build Order

| Phase | Deliverable | Done when |
|-------|-------------|-----------|
| 0 | Repo scaffold: `pyproject.toml`, `Makefile`, CI workflow | `make lint` passes on empty project |
| 1 | Data model + DB migrations | Unit tests pass for all schema operations |
| 2 | Core logic | `test_core.py` passes with fixture data |
| 3 | Interface layer (CLI / MCP tools) | Contract tests pass against spec above |
| 4 | Observability | Metrics emit to stdout in `--dry-run` mode |
| 5 | Deployment config | `make deploy-dry-run` exits 0 |

### File Map

| File | Purpose | Key symbols to define |
|------|---------|----------------------|
| `src/models.py` | Pydantic/SQLAlchemy models | <list class names> |
| `src/core.py` | Business logic | <list function names> |
| `src/cli.py` | Typer CLI | `app = typer.Typer()` |
| `src/config.py` | Settings from env | `class Config(BaseSettings)` |
| `tests/conftest.py` | Shared fixtures | <list fixture names> |
| `tests/test_core.py` | Unit tests | <list test names> |
| `Makefile` | Dev commands | `lint`, `test`, `deploy-dry-run` |
| `pyproject.toml` | Build + deps | `[project]`, `[tool.ruff]` |
| `Dockerfile` | Container build | Multi-stage, non-root user |
| `.env.example` | Env var template | All vars from Config Reference |

### Constraints

Hard rules — do not violate:

- Python 3.11+, typed signatures on all public functions
- No synchronous DB/network calls from async context
- All secrets via env vars — never hardcoded or committed
- `ruff` + `mypy --strict` must pass with zero warnings
- Tests must not hit real AWS endpoints (use moto or LocalStack)
- Log structured JSON to stderr; never `print()` in library code
- <Add project-specific constraints here>

### Acceptance Criteria

Agent self-check before declaring implementation complete:

- [ ] `make test` passes, coverage ≥ 80%
- [ ] `make lint` passes (ruff + mypy)
- [ ] `make deploy-dry-run` exits 0
- [ ] README Quick Start works on a clean machine with only `.env` configured
- [ ] All Open Questions above resolved or explicitly moved to v2
- [ ] No `TODO` / `FIXME` / `pass` left in production code paths

---

## Next Steps

Ordered task list — execute in sequence:

1. [ ] Resolve all Open Questions
2. [ ] Agent implements Phase 0 (scaffold)
3. [ ] Agent implements Phase 1 (data model)
4. [ ] Agent implements Phase 2 (core logic)
5. [ ] Agent implements Phase 3 (interface)
6. [ ] Agent implements Phase 4 (observability)
7. [ ] Agent implements Phase 5 (deployment)
8. [ ] Human review of generated code
9. [ ] Deploy to dev and run smoke test

---

## References

- <Confluence design doc URL>
- <Related repo or tool>
- <Relevant RFC or ADR>

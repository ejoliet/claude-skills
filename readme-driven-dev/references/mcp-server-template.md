# <server-name>

> <One-sentence description: what this MCP server exposes and to whom.>

![CI](https://github.com/<org>/<repo>/actions/workflows/ci.yml/badge.svg)
![Python](https://img.shields.io/badge/python-3.11+-blue)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## Purpose

**Problem**: <What is painful without this server?>
**Solution**: <What tools this server exposes and what they enable.>
**Scope**: Called by Claude Code / Claude Desktop via MCP. Runs locally or in Docker.

---

## Architecture

```
MCP Client (Claude Code / Claude Desktop)
        │  stdio / SSE transport
        ▼
  <server-name>  (FastMCP)
  ┌──────────────────────────────┐
  │  tools/                      │
  │    <tool_a>   → <upstream A> │
  │    <tool_b>   → <upstream B> │
  └──────────────────────────────┘
```

---

## Recommended Stack (researched)

| Layer | Chosen | Stars | Last Release | Why chosen | Rejected |
|-------|--------|-------|-------------|------------|---------|
| MCP framework | fastmcp | — | — | — | — |
| HTTP client | httpx | — | — | — | — |

---

## Repository Layout

```
<repo>/
├── src/
│   ├── server.py        # FastMCP app + tool registration
│   ├── tools/
│   │   └── <domain>.py  # Tool implementations
│   ├── client.py        # Shared httpx.AsyncClient
│   └── config.py        # BaseSettings
├── tests/
├── Makefile
├── pyproject.toml
├── Dockerfile
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## Installation

### Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Python | 3.11+ | |
| uv | latest | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| <Service> API token | — | [Register here](<link>) |

---

### Platform Integration

#### Claude Code
File: `~/.claude/mcp_servers.json`
```json
{
  "mcpServers": {
    "<server-name>": {
      "command": "uv",
      "args": ["run", "--directory", "/path/to/<repo>", "python", "-m", "src.server"],
      "env": { "<TOKEN_VAR>": "<your-token>" }
    }
  }
}
```
Then restart Claude Code. Run `/mcp` to verify the server appears.

---

#### Claude Desktop — macOS
File: `~/Library/Application Support/Claude/claude_desktop_config.json`
```json
{
  "mcpServers": {
    "<server-name>": {
      "command": "uv",
      "args": ["run", "--directory", "/path/to/<repo>", "python", "-m", "src.server"],
      "env": { "<TOKEN_VAR>": "<your-token>" }
    }
  }
}
```
Then: **Claude menu → Quit Claude → relaunch**. Check Settings → Developer → MCP Servers.

---

#### Claude Desktop — Windows
File: `%APPDATA%\Claude\claude_desktop_config.json`
```json
{
  "mcpServers": {
    "<server-name>": {
      "command": "uv",
      "args": ["run", "--directory", "C:\\path\\to\\<repo>", "python", "-m", "src.server"],
      "env": { "<TOKEN_VAR>": "<your-token>" }
    }
  }
}
```
> ⚠️ Use forward slashes or escape backslashes in JSON paths on Windows.

---

#### Docker — stdio via exec
```bash
docker build -t <server-name>:local .
docker run -d --name <server-name> --env-file .env <server-name>:local
```
`mcp_servers.json`:
```json
{
  "mcpServers": {
    "<server-name>": {
      "command": "docker",
      "args": ["exec", "-i", "<server-name>", "python", "-m", "src.server"]
    }
  }
}
```

---

#### Docker Compose — multi-service (e.g. with Context7)
```yaml
# docker-compose.yml
services:
  <server-name>:
    build: .
    environment:
      - <TOKEN_VAR>=${<TOKEN_VAR>}
    restart: unless-stopped

  context7:
    image: upstash/context7-mcp:latest
    restart: unless-stopped
```
```bash
docker compose up -d
```
`mcp_servers.json` (Docker exec mode):
```json
{
  "mcpServers": {
    "<server-name>": {
      "command": "docker",
      "args": ["exec", "-i", "<server-name>-1", "python", "-m", "src.server"]
    },
    "context7": {
      "command": "docker",
      "args": ["exec", "-i", "context7-1", "node", "dist/index.js"]
    }
  }
}
```

---

#### EKS / Kubernetes — SSE transport
```yaml
# helm-values.yaml snippet
env:
  - name: <SERVER_NAME>_TRANSPORT
    value: "sse"
  - name: <SERVER_NAME>_PORT
    value: "8080"
service:
  port: 8080
```
`mcp_servers.json` (remote SSE):
```json
{
  "mcpServers": {
    "<server-name>": {
      "url": "http://<k8s-service-host>:8080/sse",
      "env": { "<TOKEN_VAR>": "<your-token>" }
    }
  }
}
```

---

#### pip — no uv
```bash
pip install -e ".[dev]"
python -m src.server
```

---

## Quick Start (local, 3 steps)

```bash
git clone https://github.com/<org>/<repo>.git && cd <repo>
cp .env.example .env  # add your token
uv sync && uv run python -m src.server
```

---

## Usage Examples

> These show the natural language prompt you type to Claude, which tool fires, and a truncated realistic output.

### Example 1 — Simple lookup
**Prompt**: `"<simple one-line query>"`
→ Tool: `<tool_a>(param="<value>")`
```
<Truncated realistic output — 5–10 lines>
```

### Example 2 — Multi-step workflow
**Prompt**: `"<deeper analysis request>"`

Step-by-step:
1. `<tool_a>(...)` → get `<result_field>`
2. `<tool_b>(...)` → attempt primary path → returns `""` (silent failure case)
3. `<tool_c>(...)` → fallback → returns full content
4. Claude synthesizes: `"<plain-language explanation...>"`

> 💡 Always show the fallback path — silent failures are the #1 source of confusion.

### Example 3 — Batch / metrics use case
**Prompt**: `"<batch or aggregate request>"`
1. `<tool_search>(...)` → list of IDs
2. `<tool_metrics>(ids=[...])` → aggregated result
```
<Truncated output>
```

---

## Configuration Reference

| Variable | Type | Default | Required | Description |
|----------|------|---------|----------|-------------|
| `<TOKEN_VAR>` | str | — | ✅ | API token |
| `<BASE_URL>` | str | `<default>` | | Base URL override |
| `LOG_LEVEL` | str | `INFO` | | DEBUG/INFO/WARNING/ERROR |

---

## API / Interface Contract

```python
@mcp.tool()
async def <tool_a>(param_a: str, param_b: int = 10) -> str:
    """<Description shown to LLM. Be specific — drives tool selection.>"""

@mcp.tool()
async def <tool_b>(param_a: str) -> str:
    """<Description. Note any silent failure modes explicitly.>"""
```

---

## Claude Project Instructions

> Paste into the Claude Project Instructions field for this project.

```
You are a <role> with access to <server-name>.

### Tool routing

| Tool | When to call |
|------|-------------|
| <tool_a> | User asks about <X> |
| <tool_b> | User needs <Y>; or <tool_a> returns empty |

### Reasoning workflow

When asked to <common task>:
1. Call <tool_a> → get <result>
2. If result is empty → call <tool_b> as fallback
3. Synthesize in plain language

### Key facts

- <Silent failure mode to handle gracefully>
- <Rate limit or quota>
- <Query syntax tip>
```

---

## Error Handling

| Error Class | When raised | Retry? |
|-------------|-------------|--------|
| `AuthError` | 401 | No |
| `RateLimitError` | 429 | Yes (3×, exp. backoff) |
| `NotFoundError` | 404 | No |
| `UpstreamError` | 5xx | Yes (2×) |

Structured stderr: `{"level":"ERROR","tool":"...","error":"..."}`.

---

## Testing

```bash
make test        # All (no network — respx mocks)
make coverage    # HTML → htmlcov/
```

> ⚠️ Never hit real upstream APIs in tests. Mock all HTTP with `respx`.

---

## Non-Goals (v1)

- <Explicit exclusion>

---

## Open Questions

- [ ] **Q1**: <question>

---

## Agent Build Instructions

> Implement end-to-end using only this README. Resolve Open Questions first.

### Build Order

| Phase | Deliverable | Done when |
|-------|-------------|-----------|
| 0 | Scaffold: `pyproject.toml`, `Makefile`, CI | `make lint` passes |
| 1 | `config.py` + `client.py` | Tests pass with mocked responses |
| 2 | `tools/<domain>.py` | Tool tests pass |
| 3 | `server.py` wiring | `--list-tools` correct count + docstrings |
| 4 | Dockerfile + Compose + `.env.example` | `make deploy-dry-run` exits 0 |

### File Map

| File | Purpose | Key symbols |
|------|---------|-------------|
| `src/server.py` | FastMCP entry | `mcp = FastMCP("<server-name>")` |
| `src/config.py` | Env settings | `class Config(BaseSettings)` |
| `src/client.py` | HTTP client | `get_client() -> httpx.AsyncClient` |
| `src/tools/<domain>.py` | Tools | (list names) |
| `tests/conftest.py` | Fixtures | `mock_response`, `sample_input` |
| `Makefile` | Commands | `lint`, `test`, `deploy-dry-run` |
| `Dockerfile` | Container | Multi-stage, non-root |
| `docker-compose.yml` | Multi-service | All services wired |
| `.env.example` | Env template | All Config vars |

### Constraints

- Python 3.11+; typed signatures on all public functions
- No sync network calls from async context
- All secrets via env vars
- `ruff` + `mypy --strict` pass
- Tests never hit real APIs (`respx`)

### Acceptance Criteria

- [ ] `make test` passes, coverage ≥ 80%
- [ ] `make lint` passes
- [ ] `--list-tools` lists all tools with docstrings
- [ ] Silent failures return informative strings, not exceptions
- [ ] Usage Examples smoke-tested end-to-end in a real Claude project
- [ ] All platforms in Installation section verified
- [ ] Claude Project Instructions pasted and tested
- [ ] All Open Questions resolved or deferred to v2

---

## Next Steps

1. [ ] Resolve Open Questions
2. [ ] Agent Phase 0–4 (scaffold → tools → server → deploy)
3. [ ] `make test` + `--list-tools` smoke test
4. [ ] Pick platform from Installation section, wire `mcp_servers.json`
5. [ ] Paste Claude Project Instructions into Claude project
6. [ ] Run Usage Examples end-to-end to verify

---

## References

- <Upstream API docs>
- [FastMCP docs](https://gofastmcp.com)
- [respx — httpx mocking](https://lundberg.github.io/respx/)

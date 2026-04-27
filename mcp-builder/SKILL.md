---
name: mcp-builder
description: >
  Comprehensive guide for Model Context Protocol (MCP) server development using FastMCP (Python)
  or the MCP TypeScript SDK. Covers four-phase development: research & planning, implementation,
  review & refine, and evaluation. Use when building, extending, or testing any MCP server,
  defining tools/resources/prompts, or structuring multi-tool agent backends.
license: MIT
metadata:
  source: https://github.com/SZoloth/skill-pack
  skill-author: SZoloth (adapted for ejoliet)
---

# MCP Builder

Build production-quality Model Context Protocol servers using a four-phase methodology.

---

## When to Use This Skill

- Building a new MCP server (FastMCP, Python SDK, TypeScript SDK)
- Adding tools/resources/prompts to an existing MCP server
- Designing the tool schema or input/output contracts
- Testing or evaluating an MCP server against agent workflows
- Roman SOC file-exchange MCP, ADS MCP, or any IPAC FastMCP server work

---

## Four-Phase Development Process

### Phase 1: Deep Research & Planning

**Workflow-first design** — Consolidate related operations into unified tools that enable complete tasks rather than simply wrapping API endpoints one-to-one.

**Context optimization** — Agents have constrained context windows. Tools should:
- Return high-signal information with configurable detail levels
- Use human-readable identifiers (not internal IDs)
- Include only data the agent will act on

**Error message design** — Messages must guide agents toward correct usage:
- Include specific next steps ("Did you mean X?", "Required field Y is missing")
- Distinguish transient errors (retry) from permanent ones (fix schema)

**Pre-implementation checklist:**
- [ ] Study MCP protocol spec (tools, resources, prompts, sampling)
- [ ] Load framework docs via context7 (FastMCP or `@modelcontextprotocol/sdk`)
- [ ] Review target API docs exhaustively before writing a single tool
- [ ] Draft tool list: name, description, inputs, outputs, annotations
- [ ] Identify shared utilities needed (auth, pagination, error handling)

---

### Phase 2: Implementation

#### Project Setup

**Python (FastMCP):**
```python
from fastmcp import FastMCP
from pydantic import BaseModel

mcp = FastMCP("my-server")

class SearchInput(BaseModel):
    query: str
    limit: int = 10

@mcp.tool()
def search(input: SearchInput) -> list[dict]:
    """Search for items matching the query."""
    ...
```

**TypeScript (MCP SDK):**
```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { z } from "zod";

const server = new Server({ name: "my-server", version: "1.0.0" });

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [{ name: "search", description: "...", inputSchema: { ... } }]
}));
```

#### Infrastructure First

Build shared utilities before individual tools:
1. API client / HTTP helper with retry logic
2. Authentication management (token refresh, key rotation)
3. Error handling and response formatting
4. Pagination helpers

#### Tool Implementation Checklist

Each tool must have:
- [ ] Input schema with Pydantic (Python) or Zod (TypeScript)
- [ ] Comprehensive docstring (the agent reads this as the tool description)
- [ ] Proper MCP annotations: `readOnlyHint`, `destructiveHint`, `idempotentHint`
- [ ] Structured return type (not raw strings when possible)
- [ ] Error path returns useful `isError: true` content

#### Annotations Reference

| Annotation | When to use |
|---|---|
| `readOnlyHint=True` | Tool only reads data, never mutates |
| `destructiveHint=True` | Tool can permanently delete/overwrite |
| `idempotentHint=True` | Calling twice has same effect as once |
| `openWorldHint=False` | Tool operates only on known, bounded data |

---

### Phase 3: Review & Refine

Code quality review checklist:
- [ ] **DRY** — No duplicated logic across tools; extract shared helpers
- [ ] **Composable** — Tools can be chained by an agent without manual glue
- [ ] **Consistent** — Same field names, error shapes, and patterns across all tools
- [ ] **Error coverage** — Every external call has error handling
- [ ] **Docstrings** — Every tool description is agent-readable (clear, actionable)

**Testing note:** MCP servers are long-running processes. Running them directly in your main process will hang indefinitely. Use:
```bash
# Safe testing via MCP inspector
npx @modelcontextprotocol/inspector python -m my_mcp_server

# Or via FastMCP dev mode
fastmcp dev my_server.py
```

---

### Phase 4: Create Evaluations

Conclude development by writing **10 evaluation questions** that verify an LLM can use the server effectively:

- Questions must be realistic agent tasks (not unit test assertions)
- Answers must be verifiable and stable over time
- Cover both happy paths and error recovery
- Include multi-tool chained workflows

Example eval format:
```markdown
Q1: List all files delivered to SOC in the last 7 days with status PENDING.
Expected: Agent calls list_files(status="PENDING", days=7) and returns structured list.

Q2: What happens if you request a file that doesn't exist?
Expected: Agent receives isError=true with message guiding next steps.
```

---

## IPAC/Roman SOC Patterns

```python
# FastMCP server pattern used at IPAC
from fastmcp import FastMCP
import asyncpg

mcp = FastMCP("roman-soc-files")

@mcp.tool(annotations={"readOnlyHint": True})
async def list_soc_files(
    status: str | None = None,
    limit: int = 50
) -> list[dict]:
    """List files in the SOC exchange table. Filter by status (PENDING, DELIVERED, FAILED)."""
    async with asyncpg.connect(DSN) as conn:
        rows = await conn.fetch(
            "SELECT * FROM soc_files_table WHERE ($1::text IS NULL OR status=$1) LIMIT $2",
            status, limit
        )
    return [dict(r) for r in rows]
```

---

## Anti-Patterns to Avoid

| Anti-pattern | Instead |
|---|---|
| One tool per API endpoint | One tool per agent workflow step |
| Raw JSON string returns | Typed Pydantic/Zod response models |
| Silent failures | `isError: true` with actionable message |
| No pagination | Always add `limit`/`cursor` params |
| Generic tool names (`do_thing`) | Verb-noun names (`search_files`, `get_delivery_status`) |

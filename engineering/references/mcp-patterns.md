# MCP Patterns Reference

FastMCP boilerplate and conventions for Emmanuel's MCP servers.

---

## Server Scaffold

```python
# server.py
from fastmcp import FastMCP
from fastmcp.exceptions import ToolError
from pydantic import BaseModel, Field
from typing import Any
import structlog

log = structlog.get_logger()

mcp = FastMCP(
    name="my-mcp-server",
    version="1.0.0",
    description="What this server does.",
)

# ── Models ──────────────────────────────────────────────────────────────────

class SearchInput(BaseModel):
    query: str = Field(..., description="Search query string", min_length=1)
    limit: int = Field(10, ge=1, le=100, description="Max results to return")

# ── Service layer (no direct I/O here) ──────────────────────────────────────

async def _do_search(query: str, limit: int) -> list[dict[str, Any]]:
    """Business logic; raise ValueError on bad input."""
    ...

# ── Tools ───────────────────────────────────────────────────────────────────

@mcp.tool()
async def search_items(input: SearchInput) -> dict[str, Any]:
    """
    Search for items matching query.

    Args:
        input: Validated search parameters.

    Returns:
        Dict with `results` list and `total` count.

    Raises:
        ToolError: On downstream failure.
    """
    try:
        results = await _do_search(input.query, input.limit)
        log.info("search_complete", query=input.query, count=len(results))
        return {"results": results, "total": len(results)}
    except ValueError as e:
        raise ToolError(f"Invalid input: {e}") from e
    except Exception as e:
        log.exception("search_failed", error=str(e))
        raise ToolError(f"Search failed: {e}") from e


# ── Entry point ──────────────────────────────────────────────────────────────

if __name__ == "__main__":
    import asyncio
    asyncio.run(mcp.run())
```

---

## Database Tool Pattern (Aurora PG)

```python
# db.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase
import os

DATABASE_URL = os.environ["DATABASE_URL"]  # postgresql+asyncpg://user:pass@host/db

engine = create_async_engine(DATABASE_URL, pool_size=5, max_overflow=10)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

# In tool handler:
async with SessionLocal() as session:
    result = await session.execute(select(MyModel).where(...))
    rows = result.scalars().all()
```

---

## Error Response Contract

All tools return either:
```json
{ "data": ..., "error": null }
```
or raise `ToolError("Human-readable message")`.

Never return raw exceptions or stack traces to the model.

---

## Dockerfile (MCP Server)

```dockerfile
FROM python:3.11-slim AS builder
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

FROM python:3.11-slim
WORKDIR /app
RUN useradd -m appuser
COPY --from=builder /app/.venv /app/.venv
COPY --chown=appuser:appuser . .
USER appuser
ENV PATH="/app/.venv/bin:$PATH"
ENTRYPOINT ["python", "server.py"]
```

---

## Claude System Prompt Anti-Hallucination Pattern

For MCP servers backed by real infrastructure (like Roman file exchange), always include:

```
You have access to live data via the following tools. 
DO NOT invent file names, transfer IDs, or system states not returned by tools.
If a tool returns no results, say so — do not speculate.
All infrastructure details (hostnames, bucket names, DB tables) must come from tool responses only.
```

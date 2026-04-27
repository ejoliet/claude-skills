---
name: tdd
description: >
  Test-Driven Development with red-green-refactor loop using vertical slice tracer bullets.
  Tests verify behavior through public interfaces, not implementation details. Use when starting
  new features, fixing bugs with regression tests, or refactoring with safety net. Enforces
  planning → tracer bullet → incremental loop → refactor workflow.
license: MIT
metadata:
  source: https://github.com/SZoloth/skill-pack
  skill-author: SZoloth (adapted for ejoliet)
---

# TDD — Test-Driven Development

Red-green-refactor with vertical slice tracer bullets.

---

## When to Use This Skill

- Starting any new feature or function
- Fixing a bug (write the failing test first)
- Refactoring an existing module (ensure full test coverage before touching code)
- FastAPI endpoints, FastMCP tools, pipeline stages, or astronomy processing functions

---

## Core Philosophy

**Tests verify behavior through public interfaces, not implementation details.**

| Good tests | Bad tests |
|---|---|
| Integration-style, exercise real code paths | Coupled to implementation internals |
| Describe *what* the system does | Test private methods |
| Survive internal refactors | Mock internal collaborators |
| Use the same API a real caller would | Verify through back-channels |

---

## Anti-Pattern: Horizontal Slices

**DO NOT** write all tests first, then all implementation. This produces tests that verify *imagined* behavior, not *actual* behavior, and leads to 100% passing tests on broken code.

**Correct approach:** Vertical slices via tracer bullets.

```
One test → minimal implementation → next test → minimal implementation → ...
```

---

## Workflow

### 1. Planning (Get Approval Before Coding)

- Confirm interface changes needed
- List behaviors to test (not implementation steps — behaviors)
- Identify deep module opportunities (consolidate related logic)
- Design interfaces for testability (dependency injection, no hidden globals)
- Get user approval on the list before writing test #1

### 2. Tracer Bullet (First Slice)

Write **ONE** test for **ONE** system behavior:
- **RED**: Write the test → confirm it fails for the right reason
- **GREEN**: Write the minimal code to make it pass

### 3. Incremental Loop

For each remaining behavior on the approved list:
- **RED**: Write next test → confirm failure
- **GREEN**: Minimal code → pass
- Do not add logic not required by the current test

### 4. Refactor

After all tests are **GREEN**:
- Extract duplication
- Deepen modules (push logic down, simplify callers)
- Apply SOLID principles where natural
- Run tests after **every** refactor step

**Never refactor while RED.** Reach green first, always.

---

## Checklist Per Cycle

- [ ] Test describes behavior, not implementation
- [ ] Test uses public interface only
- [ ] Test would survive internal refactor without change
- [ ] Implementation is minimal for this test only
- [ ] No speculative features added

---

## Python Examples

### FastAPI endpoint (pytest + httpx)

```python
# test_files.py
from httpx import AsyncClient
import pytest

@pytest.mark.asyncio
async def test_list_files_returns_empty_when_no_files(client: AsyncClient):
    resp = await client.get("/files")
    assert resp.status_code == 200
    assert resp.json() == []

# Then implement the minimal /files endpoint to make this pass.
# Then write the next test: test_list_files_returns_delivered_files
```

### FastMCP tool (pytest + mcp test harness)

```python
# test_soc_tool.py
async def test_list_soc_files_filters_by_status(mcp_client):
    result = await mcp_client.call_tool("list_soc_files", {"status": "PENDING"})
    assert all(f["status"] == "PENDING" for f in result)
```

### Astronomy processing function

```python
# test_coord_transform.py
def test_galactic_to_icrs_roundtrip():
    from astropy.coordinates import Galactic, ICRS
    import astropy.units as u
    coord = Galactic(l=45*u.deg, b=30*u.deg)
    icrs = coord.icrs
    back = icrs.galactic
    assert abs(back.l.deg - 45) < 1e-6
```

---

## Tooling Defaults

| Context | Test runner | Assert library |
|---|---|---|
| Python / FastAPI | `pytest` + `pytest-asyncio` | plain `assert` |
| FastMCP servers | `pytest` + `fastmcp.testing` | plain `assert` |
| TypeScript / Node | `vitest` or `jest` | `expect()` |
| Astronomy pipelines | `pytest` + `astropy.tests` | plain `assert` |

---

## Scope Rules

- **Unit test**: One function, no I/O, no network, no DB
- **Integration test**: Real DB / real HTTP client, test container or fixture
- **E2E test**: Full stack against a running service

Prefer integration tests over unit tests for business logic. Prefer unit tests for pure computation (coord transforms, unit conversions, algorithms).

---
name: context7-docs-lookup
description: >
  Live documentation lookup for any named library or framework used in a coding
  task. Uses the Context7 MCP server to fetch up-to-date docs rather than
  relying on potentially stale training data. Trigger this skill on any coding
  task, system design, or technical question that involves a named library or
  framework — even when you think you already know the answer.
---

# Context7 Docs Lookup Skill

Fetches current, version-accurate documentation for libraries and frameworks
via the Context7 MCP server before generating code or architecture advice.

---

## When to Trigger

Trigger this skill for any coding task that involves a named dependency:

- FastAPI, SQLAlchemy, Pydantic, FastMCP
- boto3, botocore, awswrangler
- Apache Airflow, Helm, Kubernetes client
- Terraform, CDK, CloudFormation DSL
- astropy, pyvo, astroquery, asdf
- Any other library where the exact API surface matters

> ⚠️ Trigger even when you think you know the answer — training data may be
> stale. A wrong API call is worse than a slightly slower lookup.

---

## Workflow

### Step 1 — Resolve Library ID

Use the Context7 MCP `resolve-library-id` tool to get the canonical library
identifier from the library name:

```
resolve-library-id("fastmcp")
→ { "library_id": "/tadata/fastmcp", "name": "FastMCP", "versions": [...] }
```

### Step 2 — Fetch Relevant Docs

Use `query-docs` with the resolved ID and a focused topic query:

```
query-docs(
  library_id = "/tadata/fastmcp",
  query      = "tool decorator async error handling ToolError",
  tokens     = 8000
)
```

Tune `tokens` to scope:
- Simple syntax question: 2000–4000
- Architecture pattern: 6000–10000
- Migration / changelog: 8000–12000

### Step 3 — Apply to Response

Use the returned docs as the authoritative source for:
- Function signatures and parameter names
- Decorator syntax and required arguments
- Exception classes and error handling patterns
- Configuration options and defaults

Do not mix training-data recall with Context7 output if they conflict —
**Context7 wins**.

---

## Priority Library Mappings

| Library | Likely Context7 ID pattern | Priority topics |
|---------|---------------------------|-----------------|
| `fastmcp` | `/tadata/fastmcp` | tool decorator, ToolError, server init |
| `fastapi` | `/tiangolo/fastapi` | routing, dependencies, lifespan |
| `sqlalchemy` | `/sqlalchemy/sqlalchemy` | async session, mapped columns |
| `pydantic` | `/pydantic/pydantic` | model_validator, field types |
| `airflow` | `/apache/airflow` | DAG, operators, sensors |
| `boto3` | `/boto/boto3` | client vs resource, waiters |
| `pyvo` | `/astropy/pyvo` | TAP, SIA, registry search |
| `astroquery` | `/astropy/astroquery` | IRSA, MAST, query_region |
| `astropy` | `/astropy/astropy` | Table, WCS, FITS I/O, units |
| `helm` | `/helm/helm` | chart structure, values schema |
| `terraform` | `/hashicorp/terraform` | provider syntax, state |

> 💡 If `resolve-library-id` returns multiple matches, pick the one with the
> highest trust score or most recent update.

---

## Integration with Other Skills

When `context7-docs-lookup` co-triggers with another skill:

1. **Read the other skill first** to understand the task requirements.
2. **Run Context7 lookups** for any library the other skill references.
3. **Then generate code/architecture** using both the skill guidance and
   the live docs.

Example: "Build a FastMCP server for IRSA" triggers:
- `emmanuel-engineering` — architecture defaults, MCP patterns
- `context7-docs-lookup` — live FastMCP + pyvo + astroquery docs

---

## Failure Handling

If Context7 cannot resolve a library or returns empty docs:

1. Note the failure inline: `> ⚠️ Context7 lookup failed for <lib> — using training data. Verify against official docs.`
2. Fall back to training data but flag any uncertain API calls with a comment.
3. Always provide the official docs URL so the human can verify.

---

## References

- Context7 MCP server: installed as `context7@claude-plugins-official`
- Context7 docs: https://context7.com

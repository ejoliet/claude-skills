# RDD Examples

Two annotated examples showing the skill applied to real project types from
the IPAC/Roman SOC context.

---

## Example 1 — FastMCP Tool Server (minimal, focused)

Trigger phrase: *"RDD for a FastMCP server that wraps the Roman SOC file exchange DB"*

Key decisions the README must make explicit:

| Decision | What the README should specify |
|----------|-------------------------------|
| Auth | None for internal EKS deployment; `API_KEY` env var for external |
| Transport | `stdio` locally, `sse` in EKS |
| DB access | Read-only Aurora PostgreSQL via `asyncpg` connection pool |
| Tool names | Exact snake_case names in `## API / Interface Contract` |
| Error format | MCP `isError: true` with structured `content[0].text` JSON |

Critical sections for agent completeness:

```markdown
## API / Interface Contract

### MCP Tools

| Tool | Inputs | Output | Description |
|------|--------|--------|-------------|
| `list_transfers` | `status?: str`, `limit: int = 50` | `list[Transfer]` | Recent SOC↔SSC transfers |
| `get_transfer` | `transfer_id: int` | `Transfer` | Single record by ID |
| `transfer_stats` | `since_hours: int = 24` | `Stats` | Aggregate counts by status |
```

> 💡 The tool table IS the contract. The agent generates exactly these tools —
> no extras, no renames.

---

## Example 2 — Airflow DAG + Operator (pipeline tool)

Trigger phrase: *"RDD for an Airflow DAG that polls S3 and loads to Aurora"*

Key decisions:

| Decision | What to specify |
|----------|----------------|
| DAG schedule | Cron expression, not "every hour" |
| Idempotency | `logical_date` as dedup key in `soc_files_table` |
| Failure behavior | `retries=3`, `retry_delay=timedelta(minutes=5)`, alert on final failure |
| Backfill | Explicitly supported or not |

Critical sections:

```markdown
## API / Interface Contract

### DAG Parameters

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `s3_bucket` | `str` | `$S3_BUCKET` | Source bucket |
| `poll_prefix` | `str` | `transfers/` | S3 prefix to watch |
| `batch_size` | `int` | `100` | Max records per run |

### Task Graph

```
sense_s3 >> [load_new_files, skip_if_empty]
load_new_files >> update_status >> notify_success
[load_new_files, update_status] >> on_failure_callback
```

> ⚠️ Task IDs must match exactly — downstream sensors reference them by name.
```

---

## Anti-patterns to avoid

| Anti-pattern | Why it breaks agent builds |
|--------------|---------------------------|
| "Configure as needed" | Agent has no ground truth — will guess wrong |
| Function listed without signature | Agent will invent incompatible interface |
| "Standard error handling" | Agent will implement inconsistently across modules |
| Open Question left blank | Agent will make assumptions that conflict with prod |
| Phase 0 skipped | Agent starts coding without a compilable skeleton |

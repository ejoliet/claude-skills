---
name: agent-orchestration
description: >
  Multi-agent orchestration using Claude Code's TeammateTool and Task system. Covers spawning
  agents (in-process, tmux, iTerm2 backends), TeammateTool operations, task dependency graphs,
  inbox/message passing, and shutdown protocols. Use when coordinating parallel code reviews,
  building pipeline workflows, running swarm patterns, or any divide-and-conquer agent task.
license: MIT
metadata:
  source: https://github.com/SZoloth/skill-pack
  skill-author: SZoloth (adapted for ejoliet)
---

# Agent Orchestration

Master multi-agent orchestration using Claude Code's TeammateTool and Task system.

---

## When to Use This Skill

- Coordinating multiple specialized agents in parallel
- Parallel code review across multiple files/services
- Pipeline workflows with dependencies (e.g., ingest → transform → validate → publish)
- Running Roman SOC ingestion, Airflow pipeline validation, or transient broker tasks as swarms
- Any task that benefits from divide-and-conquer or fan-out patterns

---

## Primitives

| Primitive | What It Is | Location |
|---|---|---|
| **Agent** | A Claude instance that can use tools. You are an agent. | N/A (process) |
| **Team** | Named group of agents: one leader, multiple teammates | `~/.claude/teams/{name}/config.json` |
| **Teammate** | An agent that joined a team; has name, color, inbox | Listed in team config |
| **Leader** | The agent that created the team; receives messages, approves plans | First member in config |
| **Task** | Work item with subject, description, status, owner, dependencies | `~/.claude/tasks/{team}/N.json` |
| **Inbox** | JSON file where an agent receives messages from teammates | `~/.claude/teams/{name}/inboxes/{agent}.json` |
| **Message** | JSON object sent between agents (text or structured types) | Stored in inbox files |
| **Backend** | How teammates run: `in-process`, `tmux`, or `iterm2` | Auto-detected from environment |

---

## Spawn Backends

| Backend | When Used | Visibility |
|---|---|---|
| `in-process` | Same Node.js process, no terminal multiplexer | Invisible (sub-process) |
| `tmux` | tmux available in environment | Visible in separate panes |
| `iterm2` | iTerm2 with Python API | Visible in split panes |

---

## TeammateTool Operations

The 13 core operations:

| Operation | Purpose |
|---|---|
| `create_team` | Initialize a new team with leader agent |
| `join_team` | Spawn a new teammate in the team |
| `list_teams` | Show all active teams |
| `get_team` | Get team config and member list |
| `create_task` | Add a task to the team's task queue |
| `get_task` | Read task details and status |
| `list_tasks` | List all tasks for a team |
| `update_task` | Change task status, owner, or description |
| `send_message` | Send a message to a teammate's inbox |
| `read_inbox` | Read messages from your inbox |
| `request_shutdown` | Ask leader to shut down your process |
| `approve_shutdown` | Leader approves a shutdown request |
| `notify_idle` | Signal you have no more work |

---

## Orchestration Patterns

### Pattern 1: Parallel Fan-Out

```
Leader creates N tasks → spawns N teammates → each claims one task
→ each reports back → leader synthesizes
```

Use for: parallel file reviews, independent data processing, multi-archive queries

### Pattern 2: Pipeline with Dependencies

```
Task A (ingest)
  └─ Task B depends_on: [A] (transform)
       └─ Task C depends_on: [B] (validate)
            └─ Task D depends_on: [C] (publish)
```

Use for: ETL pipelines, calibration chains, multi-step report generation

### Pattern 3: Self-Organizing Queue

```
Leader creates task queue → teammates poll for unclaimed tasks
→ claim and execute → mark complete → pick next
```

Use for: unknown-length work queues, backfill jobs, catalog ingestion

---

## Message Types

```json
// Text message
{"type": "text", "from": "reviewer-1", "body": "Found 3 issues in files.py"}

// Shutdown request (teammate → leader)
{"type": "shutdown_request", "from": "worker-2", "reason": "Queue empty"}

// Idle notification
{"type": "idle_notification", "from": "worker-3"}

// Plan approval request
{"type": "plan_approval", "from": "worker-1", "plan": "..."}
```

---

## Minimal Orchestration Example

```python
# Leader creates a parallel review team
create_team("code-review")
join_team("code-review", "reviewer-1")
join_team("code-review", "reviewer-2")

create_task("code-review", "Review files.py", assignee="reviewer-1")
create_task("code-review", "Review models.py", assignee="reviewer-2")

# Each teammate runs autonomously, posts findings to inbox
# Leader reads inbox and synthesizes
```

---

## Shutdown Protocol

1. Teammate finishes work → calls `notify_idle`
2. Teammate calls `request_shutdown` with reason
3. Leader reads shutdown request from inbox
4. Leader calls `approve_shutdown` for that teammate
5. Teammate process exits cleanly

---

## Error Handling

- Always set a timeout on teammate tasks (default: 5 minutes per task)
- If a teammate goes silent, leader can reassign its tasks
- Failed tasks should be marked `FAILED` with error details, not silently dropped
- Leader should always have a fallback plan if < N teammates respond

---

## IPAC Use Cases

| Use Case | Pattern | Team Size |
|---|---|---|
| Roman SOC batch ingestion validation | Pipeline | 3–5 |
| Multi-archive VO query fan-out | Parallel fan-out | 4–8 |
| Airflow DAG review across pipelines | Parallel fan-out | 2–4 |
| Transient broker alert triage | Self-organizing queue | 3–6 |

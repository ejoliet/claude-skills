---
name: emmanuel-markdown
description: >
  Personal markdown layout guide for Emmanuel at IPAC Caltech. Apply this skill
  whenever Emmanuel asks for a markdown document, report, README, runbook, ADR
  (Architecture Decision Record), meeting notes, Confluence page, post-mortem,
  or any structured written deliverable. Trigger on phrases like "write a markdown",
  "create a doc", "document this", "write up", "draft a README", "make a runbook",
  "summarize this as a doc", or any request to produce a .md file.
  Do NOT use for inline chat answers, code-only responses, or slide decks.
---

# Emmanuel Markdown Skill

Layout and section guide for all markdown documents produced for Emmanuel.
Encodes his ADHD-friendly formatting preferences, work context, and document types.

---

## Core Formatting Rules

These apply to **every** markdown document, regardless of type:

| Rule | Requirement |
|------|-------------|
| **Hierarchy** | Max 3 heading levels (H1 title, H2 sections, H3 sub-sections) |
| **Paragraphs** | ≤ 3 sentences. Break early, break often. |
| **Lists** | Use bullets for unordered sets; numbered lists for steps only |
| **Tables** | Use for comparisons, options, env vars, trade-offs |
| **Bold** | Key terms, action items, warnings only — not decoration |
| **Code blocks** | Always fenced with language tag (` ```bash `, ` ```python `, etc.) |
| **Callouts** | Use `> ⚠️`, `> 💡`, `> ✅` blockquotes for warnings/tips/confirmations |
| **Length** | Omit sections that don't apply — never pad to look complete |

---

## Document Types & Section Templates

### 1. README / Project Overview

```
# <Project Name>

> One-sentence description of what this does and why it exists.

## Overview
[2–3 sentences: problem → solution → scope]

## Architecture
[Diagram or bullet list of components + data flow]

## Prerequisites
[Bullet list: tools, versions, AWS permissions, env vars]

## Quick Start
[Numbered steps: clone → configure → run]

## Configuration
[Table: env var | default | description]

## Usage
[Code examples for primary use cases]

## Development
[Local setup, test commands, linting]

## Deployment
[Phase-tagged steps or Helm/Terraform commands]

## Troubleshooting
[3–5 common failure modes + fixes]

## References
[Links to Confluence, related repos, design docs]
```

---

### 2. Architecture Decision Record (ADR)

```
# ADR-NNN: <Decision Title>

**Date**: YYYY-MM-DD
**Status**: Proposed | Accepted | Deprecated | Superseded by ADR-NNN
**Deciders**: [names or teams]

## Context
[What problem or situation forced this decision?]

## Options Considered

| Option | Pros | Cons | Best when |
|--------|------|------|-----------|
| A      | ...  | ...  | ...       |
| B      | ...  | ...  | ...       |

## Decision
**Chosen**: Option X

[1–2 sentence rationale]

## Consequences
- ✅ [Positive outcome]
- ⚠️ [Risk or trade-off to monitor]

## References
[Links, tickets, prior ADRs]
```

---

### 3. Runbook / Operational Procedure

```
# Runbook: <Procedure Name>

**System**: [service/component name]
**Severity**: P1 | P2 | P3
**Owner**: [team or person]
**Last reviewed**: YYYY-MM-DD

## Trigger
[When is this runbook invoked? Alarm name, threshold, symptom.]

## Impact
[What breaks? Who is affected?]

## Diagnosis Steps
1. Step one (with exact command)
2. Step two
3. ...

## Resolution Steps
1. Step one
2. Step two
3. Confirm resolution: [how to verify it's fixed]

## Escalation
[Who to page if unresolved after N minutes]

## Related
- Alarm: [CloudWatch alarm name / ARN]
- Dashboard: [Grafana URL]
- Ticket template: [Jira/Confluence link]
```

---

### 4. Technical Design / RFC

```
# <Feature or System Name> — Design Doc

**Author**: Emmanuel Joliet
**Date**: YYYY-MM-DD
**Status**: Draft | In Review | Approved

## Problem Statement
[1 paragraph: what's broken, missing, or needed?]

## Goals
- [Goal 1]
- [Goal 2]

## Non-Goals
- [Explicit exclusion 1]

## Proposed Design

### Architecture
[Component diagram or structured description]

### Data Model
[Schema tables or Pydantic models]

### API / Interface
[Endpoints, tool signatures, event shapes]

### Sequence Diagram (optional)
[Mermaid or ASCII flow]

## Trade-offs & Alternatives

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|

## Implementation Plan

| Phase | Description | Owner | ETA |
|-------|-------------|-------|-----|
| 0 | Prerequisites | | |
| 1 | Core model | | |
| ... | ... | | |

## Open Questions
- [ ] Question 1
- [ ] Question 2

## References
```

---

### 5. Meeting Notes / Sync Summary

```
# Meeting: <Topic>

**Date**: YYYY-MM-DD  **Time**: HH:MM PT
**Attendees**: [names]
**Facilitator**: [name]

## Agenda
1. Item 1
2. Item 2

## Decisions
- **Decision**: [what was decided]  
  **Rationale**: [why]

## Action Items

| # | Task | Owner | Due |
|---|------|-------|-----|
| 1 | ... | ... | ... |

## Notes
[Condensed discussion points — no transcription]

## Next Meeting
[Date / topic]
```

---

### 6. Post-Mortem / Incident Report

```
# Incident: <Short Title>

**Date**: YYYY-MM-DD
**Duration**: HH:MM – HH:MM PT
**Severity**: P1 | P2 | P3
**Status**: Resolved

## Summary
[2–3 sentences: what happened, impact, resolution]

## Timeline

| Time (PT) | Event |
|-----------|-------|
| HH:MM | Alert fired |
| HH:MM | ... |
| HH:MM | Resolved |

## Root Cause
[Specific technical cause — no blame]

## Impact
- Services affected:
- Users affected:
- Data loss: None | [describe]

## Resolution
[Steps taken to fix]

## Action Items

| # | Task | Owner | Due | Status |
|---|------|-------|-----|--------|
| 1 | ... | ... | ... | Open |

## Lessons Learned
- ✅ What worked
- ⚠️ What to improve
```

---

## Style Constants

### IPAC / Caltech Context

When producing docs for work use, assume:
- **AWS account**: `765894972596`, region `us-east-1`
- **Author**: Emmanuel Joliet (`ejoliet`)
- **Confluence space**: `~ejoliet`
- **Primary systems**: Roman SOC, SPHEREx, Euclid, NEOWISE pipelines on EKS
- Add `> ⚠️ Internal — IPAC/Caltech use only` callout at top of sensitive runbooks

### Tone
- Direct, no filler phrases ("In this document we will explore…")
- Active voice
- Engineering-literate audience — don't over-explain AWS/K8s basics

### Next Actions Block

End every document with a `## Next Steps` or `## Action Items` section if there are open tasks — never leave a doc without a clear "what happens next."

---

## Quick Decision Guide

| Request phrasing | Document type |
|------------------|---------------|
| "README for X" | README |
| "We decided to use X instead of Y" | ADR |
| "How do we fix X when it breaks" | Runbook |
| "Design doc / RFC for X" | Technical Design |
| "Notes from the X meeting" | Meeting Notes |
| "What went wrong with X incident" | Post-Mortem |
| Anything else | README (default) |

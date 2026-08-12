---
name: prompt-audit
description: >
  Act as a prompt engineering reviewer for Emmanuel. Use this skill whenever he
  pastes or references a prompt intended for an LLM and asks to audit, improve,
  rewrite, validate, or propose a better version of it. Trigger on phrases like:
  "audit this prompt", "improve this prompt", "rewrite this prompt", "make this
  instruction better", "propose a better prompt", "before answering, suggest a
  better version of my prompt", "what prompt should I use for this?", or requests
  to review a system prompt, agent prompt, CLAUDE.md instruction block, or skill
  description for quality. Do NOT trigger for normal questions, or for reviewing
  documents/instructions that are not prompts, unless he explicitly asks for a
  prompt audit.
---

# Prompt Audit

Review a prompt the way a prompt engineer would: find ambiguity, conflicts, and gaps, then produce a better version.

## Evaluate every prompt for

1. **Clarity** — is the task easy to understand?
2. **Specificity** — is the expected output defined?
3. **Missing context** — what information would improve the result?
4. **Ambiguity** — what could be interpreted in multiple ways?
5. **Conflicts** — contradictory instructions?
6. **Scope creep** — trying to do too much?
7. **Success criteria** — is it clear what a good answer looks like?
8. **Tool needs** — does it require search, files, codebase inspection, current docs, or external sources?
9. **Safety/security** — could it expose secrets, credentials, account identifiers, or private infrastructure details?
10. **Output format** — structure, length, and style specified?

## Response format

Use this structure. Omit sections that don't apply — keep simple audits short.

### Prompt Audit

**What the prompt is trying to do**
Restate the intended task in plain language.

**Assumptions**
Assumptions the prompt appears to make.

**Weaknesses**
Ambiguity, missing context, conflicts, outdated wording, unclear output format, vague success criteria.

**Questions to improve the prompt**
Only the most important missing questions. Max 3 unless the prompt is highly complex.

**Better Prompt**
A rewritten prompt, ready to use.

**Why this version is better**
The most important improvements, briefly.

**Optional Stronger Version**
Only if useful — a more rigorous version for higher-stakes or technical work.

**Validation Scenario**
One short test scenario showing how the improved prompt should behave.

**Expected Output Shape**
What the answer from the improved prompt should look like.

**Prompt Tester Notes**
Only if testing is explicitly requested: simulate how the improved prompt performs; identify remaining ambiguity or failure modes.

## Rules

- Preserve Emmanuel's intent. Do not add unrelated requirements.
- Remove or flag conflicting instructions.
- Prefer clear imperative instructions.
- Make the output format explicit.
- Keep the improved prompt practical, not over-engineered.
- Separate committed/project-safe instructions from local-only or sensitive ones.
- Warn if the prompt could expose secrets, credentials, account identifiers, or private infrastructure.
- If the prompt cannot be improved confidently, say what information is missing.

## Rigor scaling

- Simple prompt: short audit, skip optional sections.
- Prompt affecting code, infrastructure, security, documentation, or agents: full rigor, all sections.

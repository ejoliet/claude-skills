# GitHub Copilot Instructions — claude-skills

This repository stores Claude Code skills. Keep changes focused, minimal, and consistent with existing skill structure.

## Always read first

- `CLAUDE.md`
- `AGENTS.md`

## Repository conventions

- Each skill lives in `<skill-name>/SKILL.md`.
- `SKILL.md` must begin with valid YAML frontmatter (at least `name` and `description`).
- Keep `CLAUDE.md` and `skills.json` in sync when adding, renaming, or removing skills.
- Use concise, trigger-oriented wording for skill descriptions and trigger lists.
- Prefer surgical edits; avoid unrelated refactors.

## When adding or updating a skill

1. Add or update `<skill-name>/SKILL.md`.
2. Add or update the skill entry in `skills.json`.
3. Update the Skills Index and trigger rules in `CLAUDE.md`.
4. If applicable, add `references/` files under the skill directory.

## Boundaries

- Do not introduce credentials, tokens, or private data.
- Do not create broad project-wide rewrites unless explicitly requested.

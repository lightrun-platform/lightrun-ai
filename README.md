# Lightrun Skills

Small, task-focused skills for Lightrun workflows.

## What Is In This Repo

- Runtime debugging skills
- Skill authoring and maintenance skills
- Shared conventions for deterministic skill behavior

## Skill Layout

Each skill should include:

- `SKILL.md` (behavior and workflow)
- `agents/openai.yaml` (optional UI/dependency metadata)
- optional `assets/`, `scripts/`, or `references/`

## Conventions

- Keep skills deterministic and easy to audit.
- Use exact MCP tool names (no paraphrasing).
- For runtime skills, require Lightrun MCP + OAuth in the skill guidance.
- Keep outputs concise and action-oriented.

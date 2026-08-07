# Lightrun AI

A collection of reusable skills for investigating runtime issues with Lightrun.

## What is in this repository

- `skills/` — skill folders (`SKILL.md`, plus optional assets/scripts)
- `.cursor-plugin/` — Cursor plugin metadata
- `.claude-plugin/` — Claude plugin metadata
- `.cursor/mcp.json` — Cursor MCP server configuration used by the plugin metadata

## Available skills

- `lightrun-live-runtime-debugging` — deterministic live-runtime investigation workflow with preflight checks, evidence capture, and handoff output.
- `lightrun-error-remediation-automation` — error remediation automation workflow with root cause analysis, mitigation strategies, and automated fixes.
- `lightrun-ask-prod` — live production runtime question answering for current values, durations, hit counts, and distributions.
- `lightrun-runtime-context` — diagnosis for slow endpoints, latency or SLO regressions, timeouts, deadlines, and hangs.
- `lightrun-runtime-aware-pr-review` — pull request review workflow that combines live runtime samples with diff-aware risk analysis.

## Demos

- [Codex live debugging demo](https://www.youtube.com/watch?v=bIaa5_MeKlg)
- [Claude live debugging demo](https://www.youtube.com/watch?v=2RKvQTBs3pk)

## Installation

### Cursor

1. Open Cursor.
2. Add plugin from repository URL (or local path).
3. Use this repository URL: `https://github.com/lightrun-platform/lightrun-ai`

### Claude

This repository includes:

- `.claude-plugin/plugin.json`
- `.claude-plugin/marketplace.json`

Use your Claude plugin/marketplace flow to install from this repository.

### Codex

Codex loads skills from `.agents/skills` locations.

1. Copy or symlink skill folders from `skills/` into one of:
   - `$REPO_ROOT/.agents/skills`
   - `$HOME/.agents/skills`
2. Restart/reload Codex if needed.

## Requirements

- Access to Lightrun MCP server
- Completed MCP OAuth authorization for your environment

## Contributing

1. Add a new folder under `skills/<skill-name>/`.
2. Add `SKILL.md` with YAML frontmatter (`name`, `description`).
3. Add optional assets/scripts if needed.
4. Update marketplace/plugin metadata when adding publishable skills.

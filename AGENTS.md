# Agent Instructions

This repository contains Lightrun AI assistant assets: reusable skills, MCP server metadata, client plugin metadata, and setup examples. Treat it as a documentation and configuration repository, not as application source code.

## Repository Layout

- `skills/` contains publishable skill folders. Each skill must have a `SKILL.md`; optional `assets/`, `agents/`, and helper files are scoped to that skill.
- `skills/*/agents/openai.yaml` describes OpenAI-facing skill metadata and the Lightrun MCP dependency.
- `lightrun-mcp-server/` contains MCP server documentation, examples, license, and marketplace submission copy.
- `server.json` is the root MCP registry metadata. Keep its `repository.subfolder` aligned with `lightrun-mcp-server`.
- `.cursor/`, `.cursor-plugin/`, and `.claude-plugin/` contain client/plugin metadata.

## Editing Guidelines

- Keep docs and metadata consistent across README files, plugin metadata, `server.json`, and examples when changing names, URLs, descriptions, or versions.
- Preserve the public MCP endpoint `https://app.lightrun.com/mcp` unless the task explicitly targets a different deployment.
- Do not invent MCP capabilities or tool names. Skill workflows should refer to currently exposed Lightrun MCP tools and use exact tool identifiers when documenting required calls.
- Keep skill instructions deterministic and evidence-first. Runtime investigation skills must preserve preflight before evidence capture, concrete source selection, action result checks, and cleanup/handoff rules.
- Do not store credentials, OAuth tokens, API keys, customer data, or captured runtime values in this repository.
- Prefer small, focused documentation edits over broad rewrites. Maintain ASCII text unless an existing file already uses non-ASCII content for a clear reason.

## Lightrun Skill Invariants

- Runtime capture must be gated by source discovery/preflight and completed MCP authentication.
- Evidence collection should be tied to hypotheses and executable code locations.
- Async runtime actions need explicit lifecycle handling: create, status check, retrieve available data, retain or cancel, and report cleanup state.
- Agentic remediation must not deliver a final fix proposal while a required runtime action is still pending, running, or has unread results.
- If runtime data from Lightrun is used in reports, dashboards, or user-facing findings, attribute Lightrun as the data source.

## Validation

There is no application build for this repository. For metadata changes, validate JSON files that were touched, for example:

```powershell
python -m json.tool server.json
python -m json.tool .cursor\mcp.json
python -m json.tool .cursor-plugin\plugin.json
python -m json.tool .cursor-plugin\marketplace.json
python -m json.tool .claude-plugin\plugin.json
python -m json.tool .claude-plugin\marketplace.json
```

For Markdown or skill changes, review the rendered structure and check that relative asset paths still resolve from the file that references them.

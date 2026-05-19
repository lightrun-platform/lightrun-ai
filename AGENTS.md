# AGENTS.md

## Cursor Cloud specific instructions

This repository is a **configuration-and-documentation-only** project — there is no source code to build, no dependencies to install, no tests to run, and no local services to start. The entire product consists of:

- **MCP plugin metadata** (`.cursor-plugin/`, `.claude-plugin/`, `.cursor/mcp.json`, `server.json`) that configures AI clients to connect to the remote Lightrun MCP server at `https://app.lightrun.com/mcp`.
- **Skills** (`skills/`) — structured Markdown workflows (with YAML frontmatter) consumed by AI assistants.
- **Documentation and examples** (`lightrun-mcp-server/`, `README.md`).

### Key facts

- There is no `package.json`, `requirements.txt`, `Makefile`, `Dockerfile`, or any build/install step.
- The update script is intentionally empty (`echo 'No dependencies to install'`) since there are no dependencies.
- The only "runnable" component is the remote MCP server (`https://app.lightrun.com/mcp`), which requires a Lightrun account and OAuth authorization.
- HTTP 401 from the MCP endpoint is expected without authentication — it confirms the server is reachable.

### Validation

To validate the repository integrity, run the JSON/YAML checks described in the README or manually verify:

1. All JSON files parse without errors: `server.json`, `.cursor/mcp.json`, `.cursor-plugin/plugin.json`, `.cursor-plugin/marketplace.json`, `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`.
2. `skills/lightrun-live-runtime-debugging/agents/openai.yaml` parses correctly.
3. `skills/lightrun-live-runtime-debugging/SKILL.md` has valid YAML frontmatter with `name`, `description`, and `version` fields.
4. Cross-references between config files are consistent (e.g., MCP URL in `.cursor/mcp.json` matches `server.json`).

### Lint / test / build

There are no lint, test, or build commands for this repository. Contributions are validated by reviewing the JSON/YAML/Markdown structure manually or with a JSON/YAML parser.

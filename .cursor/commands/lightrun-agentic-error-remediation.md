Use the Lightrun agentic error remediation skill.

Before planning, using runtime tools, proposing a fix, or editing code, read and follow:

`skills/lightrun-agentic-error-remediation/SKILL.md`

Use this command when the requested outcome includes runtime investigation plus a fix proposal, pull request, or local fallback patch.

Use the Lightrun MCP server configured in `.cursor/mcp.json`.

Preserve the skill workflow:

- Run runtime source preflight before runtime evidence capture.
- Verify or create persistent investigation state before scheduling async runtime actions.
- Start with hypotheses and tie each runtime action to a concrete signal.
- Check every required runtime action once per investigation run.
- Do not deliver a final diagnosis, PR, or local fix while a required runtime action is pending, running, or has unread results.
- Retrieve available data before cleanup.
- Cancel or explicitly retain investigation-owned runtime actions before final handoff.

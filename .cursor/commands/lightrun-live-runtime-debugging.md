Use the Lightrun live runtime debugging skill.

Before planning or using tools, read and follow:

`skills/lightrun-live-runtime-debugging/SKILL.md`

Use this command for diagnosis-only investigations involving live runtime behavior, production or staging debugging, runtime source selection, snapshots, execution counts, execution duration, numeric runtime metrics, or call stacks.

Use the Lightrun MCP server configured in `.cursor/mcp.json`.

Preserve the skill workflow:

- Run runtime source preflight before runtime evidence capture.
- Start with hypotheses and tie each runtime action to a decision it can change.
- Use executable source locations for runtime actions.
- Check async action status/results before diagnosis.
- Retrieve available data before cleanup.
- Cancel or explicitly retain investigation-owned runtime actions before final handoff.

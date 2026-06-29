## Phase 4 - Execute Snapshots

**Skip condition:** Skip this phase entirely when Phase 2 marked every verification area as `covered`. Otherwise, run live snapshots **only for locations Phase 2 marked `uncovered`** (from the Phase 3 plan) - do not re-instrument areas already satisfied by runtime profile hits.

Use the Lightrun MCP tools available to capture expression values at each uncovered location. Discover the available tools from the MCP server and adapt to whatever snapshot API is exposed.

### Step 0 - Register correlation key

Before creating any snapshots, check whether the MCP exposes a correlation tool (for example a tool for registering an investigation, session, or correlation key).
- **If such a tool exists**: call it with the PR URL as the correlation key.
- **If no dedicated correlation tool exists**: pass the PR URL as `correlationKey` (or equivalent parameter) directly on every snapshot creation call.

This links all snapshots created during this run to the PR for traceability.

### Fixed parameters - always apply regardless of tool version

Resolve the runtime target from the environment and currently available Lightrun tooling.

Always set:
- `correlationKey`: PR URL

Choose targeting values using the narrowest valid selector supported by the exposed tool schema, for example:
- `agentPoolName`
- `customSourceName`
- `tagNames`
- `agentNames`

Prefer values discovered from Lightrun source-discovery tools. If multiple valid targets exist and the correct one is not clear from the PR context, ask the user to confirm the target environment or service.

Use tool defaults for wait time and hit limits unless the environment, tool schema, or review goal requires explicit values.

### Variable parameters - from Phase 3 plan (uncovered locations only)

- `file`: source-root relative path
- `line`: line number
- `condition`: boolean expression, or omit if null
- `expressions`: list of 1-10 expressions

### Async tool pattern (if MCP exposes create + status + retrieve tools)

1. Create the snapshot action - store the returned action identifier
2. Perform bounded in-session status checks for a short but meaningful window.
3. Retrieve expression values whenever new hits arrive.
4. If in-session results are sufficient for the changed logic, continue the review in the same run.
5. If in-session results are still unavailable or insufficient, emit the Sampling Request below and stop active review pending follow-up traffic or a later resume.

### Sync tool pattern (if MCP exposes a single blocking tool)

1. Call the tool - it blocks until hits are captured or timeout is reached.
2. If the returned results are sufficient for the changed logic, continue the review in the same run.
3. If the returned results are unavailable or insufficient, emit the Sampling Request below and stop active review pending follow-up traffic or a later resume.

### Sampling Request

Use this message only when in-session collection did not produce enough runtime evidence to continue the review in the same run.

Emit a chat message containing:
- Which **uncovered** code locations are being sampled (file:line for each live snapshot in this phase)
- The specific action the PR owner must perform in the production system to trigger each code path - use the "How to trigger" from the Sampling Plan
- The active trigger window for the current runtime actions. Use the tool default window unless a different window was intentionally chosen.

Do not list areas already `covered` by Phase 2 runtime profile hits.

Example format:

> **Runtime sampling active - action required within the active capture window**
>
> Lightrun snapshots are waiting for hits at:
> - `com/example/MyClass.java:180` - [trigger instruction from plan]
> - `com/example/MyClass.java:207` - [trigger instruction from plan]
>
> Please trigger the relevant flow in the production system now.

### Cleanup

After retrieving results (or after a no-hit retry cycle), cancel or delete any snapshot actions that are still active using the appropriate MCP tool (for example cancel or delete by action identifier). Do not leave snapshots running in production after the investigation is complete.

### No-hit handling

If no hits are captured for an uncovered location:
- Do not guess or fabricate samples
- Check: non-executable line, code-version mismatch, condition too narrow
- **First**: try a higher-traffic line in the same method, or loosen / remove the condition
- **Then**: retry that location with an adjusted plan using a small retry budget. Use up to **3 snapshot attempts per uncovered location** as the recommended default unless there is a clear reason to do more.

Other verification areas keep their existing samples - runtime profile hits from Phase 2 and successful live captures from earlier attempts are not discarded when one location retries.

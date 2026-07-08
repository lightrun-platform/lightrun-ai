## Phase 4 — Execute Snapshots

**Skip condition:** Skip this phase entirely when Phase 2 marked every verification area as `covered`. Otherwise, run live snapshots **only for locations Phase 2 marked `uncovered`** (from the Phase 3 plan) — do not re-instrument areas already satisfied by runtime profile hits.

Use the Lightrun MCP tools available to capture expression values at each uncovered location. Discover the available tools from the MCP server and adapt to whatever snapshot API is exposed.

### Step 0 — Register correlation key
Before creating any snapshots, check whether the MCP exposes a correlation tool (e.g. a tool for registering an investigation, session, or correlation key).
- **If such a tool exists**: call it with the PR URL as the correlation key.
- **If no dedicated correlation tool exists**: pass the PR URL as `correlationKey` (or equivalent parameter) directly on every snapshot creation call.

This links all snapshots created during this run to the PR for traceability.

### Fixed parameters — always apply regardless of tool version

Resolve the runtime target from the environment and currently available Lightrun tooling.

Always set:
- `correlationKey`: PR URL

Choose stable shared targeting values discovered from Lightrun source-discovery tools.

Prefer:
- `customSourceName`
- `tagNames`

Avoid `agentNames` unless there is a clear reason to target a specific instance.

If multiple valid shared targets exist and the correct one is not clear from the PR context, ask the user to confirm the target environment or service.

Unless there is a clear reason to do otherwise, use explicit snapshot limits that match this review workflow:
- `totalMaxHits`: `5`
- `maxWaitTimeSeconds`: `600`

### Variable parameters — from Phase 3 plan (uncovered locations only)
- `file`: source-root relative path
- `line`: line number
- `condition`: boolean expression, or omit if null
- `expressions`: list of 1–10 expressions

### Async tool pattern (if MCP exposes create + status + retrieve tools)

**Coverage gate (apply before steps 4–5):**
- **Sufficient** = every verification location (uncovered in Phase 2 and/or planned in Phase 3) has at least one hit from Phase 2 profile data or Phase 4 live capture.
- **Insufficient** = any verification location still has zero hits after the bounded in-session poll below.

Partial hits on one location do not satisfy Phase 4 while other locations remain at zero.

1. Create the snapshot action — store the returned action identifier.
2. Perform bounded in-session status checks for up to 60–120 seconds to catch already-active traffic. This short poll is not a substitute for the active capture window on zero-hit locations.
3. Retrieve expression values whenever new hits arrive.
4. If **every** verification location has at least one hit, continue to Phase 5 in the same run.
5. If **any** verification location still has zero hits, emit the Sampling Request below and **stop**. Do not continue to Phase 5 or emit the Output Format in this run.
6. **On resume only** — when the user later resumes the review or invokes it again, first check the previously created correlated runtime actions for hits.
7. **On resume only** — if usable hits are now available for previously zero-hit locations, continue to Phase 5 with them. If zero-hit locations remain after re-checking correlated actions and the active capture window has elapsed, continue to Phase 5 with the evidence state as-is.

### Sync tool pattern (if MCP exposes a single blocking tool)

1. Call the tool - it blocks until hits are captured or the wait time elapses.
2. If the returned results are sufficient for the changed logic, continue the review.
3. If the returned results are unavailable or insufficient, continue the review with the evidence state as-is.

### Sampling Request

Emit this message when **any** uncovered verification location still has zero hits after the bounded in-session poll in the async tool pattern — even when other locations already have hits.

Partial coverage does **not** satisfy Phase 4. Do not proceed to Phase 5 or emit the Output Format in the same run.

After emitting this message, stop the current run.

Emit a chat message containing:
- Which **zero-hit** uncovered code locations are being sampled (file:line for each live async snapshot still waiting for hits)
- The specific action the user must perform in the target system to trigger each code path — use the "How to trigger" from the Sampling Plan
- The active trigger window for the current runtime actions. Use the tool default window unless a different window was intentionally chosen.

Do not list areas already `covered` by Phase 2 runtime profile hits or locations that already have Phase 4 hits.

Example format:

> **Runtime sampling active — action required within the active capture window**
>
> Lightrun snapshots are waiting for hits at:
> - `com/example/MyClass.java:180` — [trigger instruction from plan]
> - `com/example/MyClass.java:207` — [trigger instruction from plan]
>
> Please trigger the relevant flow in the target system now.

### Cleanup
After retrieving results (or after a no-hit retry cycle), cancel or delete any snapshot actions that are still active using the appropriate MCP tool (e.g. cancel or delete by action identifier). Do not leave snapshots running in production after the investigation is complete.

### No-hit handling
If no hits are captured for an uncovered location:
- Do not guess or fabricate samples
- Check: non-executable line, code-version mismatch, condition too narrow
- **First**: try a higher-traffic line in the same method, or loosen / remove the condition
- **Then**: retry that location with an adjusted plan (up to **3 snapshot attempts per uncovered location**)

Other verification areas keep their existing samples — runtime profile hits from Phase 2 and successful live captures from earlier attempts are not discarded when one location retries.

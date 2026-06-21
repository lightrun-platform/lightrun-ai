---
name: lightrun-feature-validation
description: >-
  Use Lightrun to capture real runtime data from production in order to design
  or validate a new feature — before writing or merging code. Works across
  Java, .NET, Node.js, and Python 3.11.
version: 0.1.0
---

# Goal

Capture real production data to answer a specific question about a new feature — either to understand what the system actually does today (design), or to confirm that a new behavior matches expectations (validate) — without redeployment and without synthetic data.

# When to Use This Skill

Use this skill when:
- You are **designing** a new feature and need to understand what real runtime data looks like before writing code.
- You are **validating** a feature in progress and need to confirm the actual runtime behavior matches the expected behavior.
- You want to go **left of the PR** — catching gaps or wrong assumptions before code review or merge.

This skill is NOT for debugging a broken feature or incident response. For that, use `lightrun-live-data-sampling`.

# Scope

- In scope: question intake, codebase exploration to find a valid observation point, expression construction, data capture, comparison against expected behavior, local file output.
- Out of scope: code changes, bug fixes, incident response, shared/remote storage.

# Preconditions

- User has access to the target service source code locally.
- Lightrun MCP server is installed and authenticated.
- OAuth authorization for Lightrun MCP is completed before runtime capture.
- Supported runtimes: Java, .NET, Node.js, Python 3.11 (not later versions).

# MCP Tool Discovery

Follow [MCP tool discovery](references/mcp-tool-discovery.md). Read MCP tool schemas and descriptions at run time; do not assume fixed tool names or client prefixes.

# MCP Preflight

Complete source discovery per [MCP tool discovery](references/mcp-tool-discovery.md) before any runtime evidence capture.

- Pass and fail criteria: see [Source discovery (preflight)](references/mcp-tool-discovery.md#source-discovery-preflight).
- Missing-MCP recovery: see [Missing-MCP recovery](references/mcp-tool-discovery.md#missing-mcp-recovery).

# Missing-MCP Recovery

Use [Missing-MCP recovery](references/mcp-tool-discovery.md#missing-mcp-recovery). Continue only after preflight pass criteria are met.

# Runtime Tool Selection Strategy

- At run start, inspect currently exposed Lightrun runtime tools and their descriptions before selecting a capture path.
- Choose the smallest useful evidence capability for the feature question:
  - expression/value capture for object, field, request, response, and branch-state questions;
  - execution count for "does this run / how often" questions;
  - numeric metric or distribution capture for runtime value ranges;
  - execution duration for latency questions.
- Record the selected tool identifier exactly as exposed by MCP.
- Before each action, state what design or validation decision this action can change.
- Re-check currently exposed runtime tools when resuming a later run, and adapt the capture path if available capabilities changed.

# Question Intake

Before any codebase exploration or runtime action, collect the following. Ask only for what is missing.

Required:
- **The question to answer**: what does the user need to learn or confirm? Examples:
  - "I want to understand what fields are present in the request before I design the new enrichment layer."
  - "I want to confirm that the new scoring logic produces the expected value for premium users."
- **Observation target**: what object, field, or value should be captured to answer the question?
- **Condition / scope**: under what runtime conditions should the capture fire? (e.g. specific user segment, partner, content type, environment)
- **Environment / pod**: which environment to observe (prod, staging, specific region or pod).

Optional (infer if not provided):
- **Expected value** (for validation mode): what value or structure does the user expect to see? If provided, compare captured data against it and report match/mismatch explicitly.
- **Output path**: where to save the file locally. Default: `~/Downloads/<ServiceName>/<YYYY-MM-DD>/`.
- **Max samples**: how many hits to capture. Default: 1.

If the question is absent, ask: "What do you need to learn or confirm from the running system? For example: what fields exist, what value is computed, or whether a condition holds at runtime."

# Mode Detection

Determine the mode based on the user's question:

- **Design mode**: the user is exploring unknown runtime behavior to inform how a feature should be built. There is no expected value yet. Output is a captured snapshot the user can use as a reference.
- **Validation mode**: the user has an expected value or behavior and wants to confirm the system matches it. Output includes a match/mismatch verdict.

State the mode explicitly before proceeding.

# Runtime Detection

After preflight, detect the target service runtime from source file extensions or build files:
- `.java` / `pom.xml` / `build.gradle` -> Java
- `.cs` / `.csproj` -> .NET
- `.js` / `.ts` / `package.json` -> Node.js
- `.py` / `pyproject.toml` / `requirements.txt` -> Python

State the detected runtime explicitly. Apply the matching expression constraint rules for all subsequent instrumentation steps.

# Observation Point Selection

A valid observation point must satisfy all of the following:
- The target object or field is available as a **typed local variable** at that line.
- The line is **executable** (not a declaration, interface, or abstract method).
- The line is on the **actual code path** for the specified condition.
- The expression depth required to access target fields stays within the runtime limit.

Search strategy:
1. Trace the call path from the service entry point toward the condition scope.
2. Identify where the observation target is available as a typed local variable.
3. Prefer locations just **after** the target object is fully populated.
4. If the target is a byte array (serialized), prefer capturing at the point of serialization completion.

# Expression Constraint Rules by Runtime

## Java

Forbidden in expressions:
- Lambdas and method references (`.stream()`, `.map()`, `->`, `::`)
- The `new` keyword
- `toString()` on large objects or collections
- `Base64.getEncoder().encodeToString(...)`
- `objectMapper.writeValueAsString(...)`
- `new String(bytes)`

Generic erasure workarounds:
- Use explicit casts when a collection returns `Object`: `((TargetType)list.get(0))`

Depth limits:
- Default depth ~3 levels. Use indexed access for deeper fields: `((Deal)dealList.get(0)).getWhitelistSeats()`

Large payload extraction (byte arrays):
- Use chunked capture with `java.util.Arrays.copyOfRange(byteArray, N, N+1024)` at N = 0, 1024, 2048 (add more if needed).
- Each chunk exposes its own `$base64` field.
- Use gzip-tolerant decompression for compressed payloads.

## Python (3.11 only)

Forbidden: `import`, generators, list comprehensions, lambdas, multi-line expressions.
Allowed: attribute access, index access, simple comparisons.

## Node.js

Forbidden: `require()`, arrow functions, `JSON.stringify()` on large objects, template literals.
Allowed: property access, index access, simple comparisons.

## .NET

Forbidden: LINQ expressions, `new` keyword, multi-statement expressions.
Allowed: property access, index access, simple casts.

# Data Extraction and Comparison

After a hit is captured:

1. Retrieve captured expression values from the Lightrun MCP tool result.
2. Identify format: base64 chunked (Java byte arrays) or plain object.
3. Reassemble chunked base64 if needed: decode, concatenate, decompress, parse as JSON.
4. **Design mode**: save the captured data as a reference snapshot. Summarize what was found and highlight fields relevant to the feature being designed.
5. **Validation mode**: compare captured data against the expected value provided during intake. Report explicitly: MATCH or MISMATCH. If mismatch, show the diff.
6. Save to output path with a descriptive filename.

# Output File Naming

Format: `<environment>_<mode>_<feature-slug>_<date>.json`

Examples:
- `prod_design_enrichment-layer-input_2025-06-01.json`
- `staging_validate_scoring-premium-user_2025-06-01.json`

Rules:
- Use `design` or `validate` in the filename to reflect the mode.
- Derive `<feature-slug>` from the feature or question in natural language, lowercased, hyphenated.
- Create the output directory if it does not exist.

Default output path: `~/Downloads/<ServiceName>/<YYYY-MM-DD>/`

# Async Activation Gate

Use a same-run capture capability discovered from the current MCP tool list first. Switch to async when:
- Two consecutive synchronous attempts return no hits on a correctly targeted location.
- The user cannot reproduce the traffic condition in the current session.

When switching to async:
- Create the async action using the exact exposed MCP identifier and record the action ID.
- Provide exact reproduction instructions and the active collection window.
- Pause until the next session; resume by checking existing action IDs before creating new ones.

# Runtime Action Cleanup

Before ending the session:
- Cancel any Lightrun actions that are no longer needed.
- Retain async actions only if a concrete next reproduction step depends on them.
- Report cleanup status: cancelled / retained / already terminal.

# Flow

1. Collect question, observation target, condition, and environment. Ask for missing fields.
2. Detect mode: design or validation.
3. Inspect currently exposed Lightrun runtime tools and select the capture capability that fits the question.
4. Complete source discovery preflight using read-only discovery capabilities. Select source target.
5. Detect runtime from source files.
6. Explore codebase: trace call path, find valid observation point.
7. Construct condition expression using language-specific constraint rules.
8. Place Lightrun action at the selected location using the exact MCP-exposed tool identifier.
9. Ask user to trigger traffic matching the condition (or confirm it flows naturally).
10. If no hits, check whether traffic is region/pod-specific. Suggest switching pod before changing instrumentation.
11. Apply async gate if synchronous capture misses twice across pods.
12. On hit: retrieve data, reassemble if chunked, decompress if needed.
13. Design mode: summarize findings as a reference snapshot.
    Validation mode: compare against expected value, report MATCH or MISMATCH with diff.
14. Save to output path with descriptive filename.
15. Run cleanup gate: cancel unneeded actions.
16. Report: file path, mode, verdict (validation) or summary (design), any remaining unknowns.

# Output Contract

- **Mode declared**: design or validation
- **Preflight pass**: selected source target + source-selection reasoning + first planned action with exact MCP-exposed identifier
- **Preflight fail**: blocker category + exact remediation + retry condition
- **Observation point selected**: file, line number, variable name, runtime, constraint notes
- **Condition constructed**: the exact expression string placed on the action
- **Reproduction required** (async): active action IDs + reproduction instructions + collection window
- **Capture success (design)**: file path + summary of captured fields relevant to the feature
- **Capture success (validation)**: file path + MATCH or MISMATCH verdict + diff if mismatch
- **Cleanup summary**: cancelled / retained / terminal action IDs

# Quality Checklist

- [ ] Question and mode (design/validation) established before any codebase exploration.
- [ ] Expected value collected if validation mode.
- [ ] Source discovery completed before any runtime evidence tool.
- [ ] Discovery followed tool-description usage flow and pagination when needed.
- [ ] Preflight produced `agentPoolName` plus selector required by the first evidence tool.
- [ ] Selected runtime tool identifier matches the MCP-exposed name exactly.
- [ ] Runtime detected from source files and stated explicitly.
- [ ] Observation point validated: typed local variable, executable line, correct code path.
- [ ] Expression constructed using only allowed constructs for the detected runtime.
- [ ] Generic erasure handled with explicit casts (Java).
- [ ] toString() not used in conditions or large-object expressions (Java).
- [ ] Lambdas and streams not used in any expression.
- [ ] Chunked base64 capture used when payload exceeds 1024 bytes (Java byte arrays).
- [ ] Gzip-tolerant decompression used when capturing compressed payloads.
- [ ] Design mode: captured data summarized with feature-relevant highlights.
- [ ] Validation mode: explicit MATCH or MISMATCH verdict with diff.
- [ ] Output file named with mode indicator and feature slug.
- [ ] Output directory created if it does not exist.
- [ ] If no hits in current pod/region, recommend switching pod before changing instrumentation.
- [ ] Async gate evaluated after two consecutive no-hit synchronous attempts.
- [ ] Existing async action IDs checked before creating new ones on resume.
- [ ] Cleanup gate executed before final response.
- [ ] Final output includes file path, mode, and verdict or summary.

---
name: lightrun-ask-prod
description: >-
  Answer questions about production system behavior — captured variable values,
  execution durations, hit counts, and value distributions — using suitable
  existing Runtime History first, then live instrumentation with Lightrun MCP
  tools when needed. Use when the question requires runtime evidence rather
  than static code analysis (e.g. "show recent requests to this endpoint",
  "show the runtime distribution for this
  operation", "what values appear for this expression in production?", "which
  branch runs for customer X?"). Do not use for incident diagnosis,
  pull-request review, code changes, setup, or deployment. Route latency,
  timeout, deadline, SLO, or hang diagnosis to lightrun-slow-execution-diagnosis when
  available; route other diagnosis, or fallback when that skill is unavailable,
  to lightrun-live-runtime-debugging when available. If neither diagnosis skill
  is available, report the scope gap.
---

# Usage telemetry

When this skill is invoked, it may report one usage event through the Lightrun-owned `skill_start` MCP tool. This helps Lightrun measure skill adoption. The request supplies the skill name `lightrun-ask-prod`.

Call the tool once only when it is verified as belonging to the Lightrun MCP provider. Do not call similarly named tools exposed by other providers. If the tool is unavailable or the provider cannot be verified, skip telemetry and continue normally.

# Ask Prod

Answer production questions from suitable captured Runtime History, falling back to live measurement when needed.

## MCP Tool Discovery

Follow [MCP tool discovery](references/mcp-tool-discovery.md). Read MCP tool schemas and descriptions at run time; do not assume fixed tool names or client prefixes.

## Prerequisites

- Lightrun MCP installed and active
- Access to relevant Runtime History for historical answers; history reads do not require active agents or source-discovery preflight
- Access to relevant source code when locating code, interpreting captured behavior, or validating code-version compatibility
- For live actions: an executable source location and at least one matching active agent pool connected to Lightrun

---

## Flow

### 1. Understand the question

Read the user's question and identify:
- What value, behavior, or measurement is being asked about
- Which service or component is likely involved
- Whether the question requires a single data point or a combination of capabilities
- What runtime source scope is needed: one agent, multiple agents, a tag, a custom source, a region, or a customer-specific traffic path
- Whether the answer requires a short collection window or a longer observation window
- What capture time range and code version would make existing evidence suitable, especially for questions about current state

### 2. Identify the right capability

Select the Lightrun capability that best fits the question from the currently exposed MCP tools. More than one may be needed.

| If the question asks... | Use... |
|---|---|
| What values, request examples, or branch/stack behavior were observed? | Runtime History; snapshot/expression capture if insufficient |
| What is the current value of X? | Runtime History only if sufficiently current; otherwise snapshot/expression capture |
| How long does operation X take? | Execution duration |
| How often does line X run? | Execution count |
| What range of values does X take over time? | Distribution / numeric metric |

For a new duration, execution-count, or numeric-metric measurement that stored snapshots cannot supply, skip irrelevant history calls and proceed to the live flow. Historical hit counts and deduplicated values are not execution counts or measured production distributions.

### 3. Check suitable Runtime History

When stored snapshots can answer the question, use the exposed Runtime History read tools before creating a live action. Follow the [history query guidance](references/mcp-tool-discovery.md#runtime-history-reads); do not gate these reads on active-agent/source preflight.

1. Query the narrowest relevant code, time/version, and runtime-source scope supported by the schema, requesting only the evidence needed to answer the question.
2. Evaluate suitability using the shared history guidance.
3. If the evidence is suitable, answer directly using step 7 and create no live action. Otherwise explain the specific gap and continue to step 4.

Empty or insufficient history triggers the existing live flow; it does not prove that code did not execute or that a value did not occur.

### 4. Locate the relevant code for live measurement

Before creating a live action, identify an executable **file path** and **line number** tied to the requested measurement.

Useful signals by question type:
- **Current values** (e.g. cache size, selected instance state): find the field or variable on the relevant object, at the point it is read or returned
- **Durations**: identify the start and end lines of the relevant code section
- **Request/response examples**: find where the request is parsed or the response is constructed
- **Branch behavior**: locate the conditional and the variables that determine which path runs

Select the strongest code candidate from available evidence. Ask the user only when viable candidates remain indistinguishable. If no executable location can be identified, do not capture; return the blocker, required remediation, and retry condition.

### 5. Select the agent pool and agents for live measurement

Complete source discovery using read-only discovery tools (see [Source discovery (preflight)](references/mcp-tool-discovery.md#source-discovery-preflight)). Follow tool-description usage flow and pagination when the pool or agent list is large.

Apply this selection logic:
- Select the strongest runtime source candidate from available metadata for the service and question
- Use a specific agent when the question is about one known instance
- Use multiple agents when the answer should compare behavior across selected instances
- Use a tag or custom source when the answer should cover a service, environment, region, tenant group, or another shared runtime scope
- Prefer sources actively serving traffic relevant to the question (e.g. for a US-specific question, prefer sources handling US requests if identifiable)
- Ask the user only when viable source candidates remain indistinguishable
- If source preflight fails, do not capture; return the blocker, required remediation, and retry condition

> Do not present a single-instance result as a global production answer. When full coverage is not available, describe the observed scope explicitly.

### 6. Collect live runtime data

Call the appropriate Lightrun tool with the executable location, selected source scope, and observation window.

Use runtime actions for evidence collection:
- Use the default duration only when it covers expected signal latency and fits task or session bounds; otherwise use an explicit bounded window and state why
- Before creating an action, check for a duplicate owned measurement. Treat only actions created or explicitly inherited for this request as owned; cancel only owned actions that are stale, duplicate, superseded, mistargeted, or unnecessary
- Retain an owned action only for a concrete pending observation, recording its reason and expected window. After repeated no-data or low-information results, revalidate or change the source, code location, signal, or window
- Follow the exposed schema's status and result-availability semantics: new results → retrieve them; non-terminal without new results → wait within the bounded window and return the resume condition; terminal → report the blocker or empty result

### 7. Analyse and answer

Answer the measurement directly in plain language using the returned data:
- If a distribution was collected, summarise the range, typical value, and any notable outliers
- If multiple capabilities were used, synthesise the results into a single coherent answer
- For count-style questions across multiple instances, aggregate only the selected sources and state the source coverage
- Include the runtime sources covered, collection window, and limits of the result
- Distinguish existing history from newly captured live evidence. Cite supporting hit IDs when available and state capture times, code version, sample scope, and completeness limits; do not present past captures as current state without sufficient evidence

For each answer or handoff, include established state needed to understand or resume the measurement: measurement question, executable code location, selected source scope, observation window, observed facts and coverage limits, and owned action disposition when applicable. Omit unavailable fields and workflow narration.

---

## Error Handling

| Situation | Action |
|---|---|
| No agent pools found when live measurement is needed | Inform the user; ask them to verify the service is running and connected to Lightrun; this does not block an answer from sufficient history |
| Line cannot be instrumented | Try an adjacent line; if still unavailable, explain the limitation to the user |

---

## Schema-Driven Example

The call below is illustrative; its literal identifiers are not the Lightrun API. Take real tool names and parameters from the exposed MCP schema.

- **Question:** "How long does `priceOrder` take in the selected production service?"
- **Selection:** Choose execution duration at `src/pricing.ts:84-91`; source preflight confirms `pricing-prod-us`; use a bounded 60-second window.
- **Illustrative call:** `measure_duration({"file":"src/pricing.ts","start_line":84,"end_line":91,"source":"pricing-prod-us","window_seconds":60})`
- **Validation:** The schema-defined status exposes new results; retrieve them and verify the expected duration fields.
- **Representative result:** `{"sample_count":250,"median_ms":42,"upper_ms":87}`
- **Answer:** "`priceOrder` took a median 42 ms across 250 samples, with the reported upper bound at 87 ms, at `src/pricing.ts:84-91` for `pricing-prod-us` during a 60-second window. Coverage is limited to that source and window."

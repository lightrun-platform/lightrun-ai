---
name: lightrun-ask-prod
description: >-
  Answer questions about live production system behavior — current variable
  values, execution durations, hit counts, and value distributions — by
  instrumenting running services with Lightrun MCP tools. Use when the question
  requires live runtime data rather than static code analysis (e.g. "show
  recent requests to this endpoint", "show the runtime distribution for this
  operation", "what values appear for this expression in production?", "which
  branch runs for customer X?").
---

# Ask Prod

Query live production runtime to answer questions about system behavior using Lightrun's observability tools.

## Prerequisites

- Lightrun MCP installed and active
- Access to the relevant service's source code
- At least one active agent pool connected to Lightrun

---

## Flow

### 1. Understand the question

Read the user's question and identify:
- What value, behavior, or measurement is being asked about
- Which service or component is likely involved
- Whether the question requires a single data point or a combination of capabilities
- What runtime source scope is needed: one agent, multiple agents, a tag, a custom source, a region, or a customer-specific traffic path
- Whether the answer requires a short collection window or a longer observation window

### 2. Identify the right capability

Select the Lightrun capability that best fits the question. More than one may be needed.

| If the question asks... | Use... |
|---|---|
| What is the current value of X? | Snapshot expression capture (`snapshot_create`, `snapshot_status`, and `snapshot_get_values`) |
| How long does operation X take? | Execution duration (`get_runtime_execution_duration`) |
| How often does line X run? | Execution count (`get_runtime_execution_count`) |
| What range of values does X take over time? | Distribution / numeric metric (`get_runtime_numeric_metric`) |

### 3. Locate the relevant code

Before calling any tool, identify the specific **file path** and **line number** that will yield the needed information.

Useful signals by question type:
- **Current values** (e.g. cache size, selected instance state): find the field or variable on the relevant object, at the point it is read or returned
- **Durations**: identify the start and end lines of the relevant code section
- **Request/response examples**: find where the request is parsed or the response is constructed
- **Branch behavior**: locate the conditional and the variables that determine which path runs

> If the code location is ambiguous, ask the user to clarify before proceeding. Do not guess.

### 4. Select the agent pool and agents

Call `get_runtime_sources` to retrieve all agent pools, each with their agents, tags, and custom sources in one response.

Apply this selection logic:
- Select the runtime source scope that corresponds to the service and question
- Use a specific agent when the question is about one known instance
- Use multiple agents when the answer should compare behavior across selected instances
- Use a tag or custom source when the answer should cover a service, environment, region, tenant group, or another shared runtime scope
- If multiple pools or source scopes are plausible, present the list to the user and ask which to use
- Prefer sources actively serving traffic relevant to the question (e.g. for a US-specific question, prefer sources handling US requests if identifiable)
- If the right source scope is still ambiguous, present the options to the user and ask which to use

> Do not present a single-instance result as a global production answer. When full coverage is not available, describe the observed scope explicitly.

### 5. Collect runtime data

Call the appropriate Lightrun tool with:
- The file path and line number from step 3
- The selected source scope from step 4
- An observation window that matches the question

Use runtime actions for evidence collection:
- If the user expects the code path to run quickly, wait briefly for results in the current session
- If the expected signal needs a longer or uncertain observation window, keep the action active and explain when to check the result
- Track action IDs and check action status before creating duplicate actions for the same question
- Cancel actions that are no longer needed, or explain why an action remains active

### 6. Analyse and answer

Interpret the returned data in the context of the user's question:
- Translate raw values into a plain-language answer
- If a distribution was collected, summarise the range, typical value, and any notable outliers
- If multiple capabilities were used, synthesise the results into a single coherent answer
- For count-style questions across multiple instances, aggregate only the selected sources and state the source coverage
- Include the runtime sources covered, collection window, and limits of the result
- If the data is ambiguous or insufficient, say so clearly and suggest what additional instrumentation might help

---

## Error Handling

| Situation | Action |
|---|---|
| No agent pools found | Inform the user; ask them to verify the service is running and connected to Lightrun |
| Multiple plausible agent pools | Present the list to the user and ask which to use |
| Multiple plausible agents within a pool | Use the agents, tags, and custom sources returned by `get_runtime_sources` to narrow down; if still ambiguous, ask the user |
| A single source cannot answer a fleet-level question | Target a tag, custom source, or multiple agents that match the requested scope; if full coverage is unavailable, frame the answer in terms of selected sources |
| Tool returns no data | Keep the action active for a longer observation window; if still empty, consider whether the code path is being actively exercised |
| Longer observation window needed | State the observation window, action ID, selected source scope, signal being collected, and condition for checking results later |
| Line cannot be instrumented | Try an adjacent line; if still unavailable, explain the limitation to the user |
| Code location unclear | Ask the user before proceeding — do not guess |

---

## Example Questions This Skill Can Answer

- Show the current value of `cacheSize` in the selected production instances.
- How many users are logged in across the selected production instances?
- Show details of a few recent requests sent to `POST /api/v1/users`.
- Show details of a few recent responses from the external sales tax service.
- Is this cache key currently in use?
- Show the runtime distribution for trade requests coming in now.
- Let me know if this list is empty in production during the next 24 hours.
- What values are showing up in production for this expression when `varX` is true?
- Which branch of `calculateDiscount` runs for customer X?

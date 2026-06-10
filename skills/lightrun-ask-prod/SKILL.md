---
name: lightrun-ask-prod
description: >-
  Answer questions about live production system behavior — current variable
  values, execution durations, hit counts, and value distributions — by
  instrumenting running services with Lightrun MCP tools. Use when the question
  requires live runtime data rather than static code analysis (e.g. "how many
  connections are in the pool?", "how long does X take?", "is this list ever
  empty in production?", "which branch runs for customer X?").
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
- **Current values** (e.g. pool size, logged-in users): find the field or variable on the relevant object, at the point it is read or returned
- **Durations**: identify the start and end lines of the relevant code section
- **Request/response examples**: find where the request is parsed or the response is constructed
- **Branch behavior**: locate the conditional and the variables that determine which path runs

> If the code location is ambiguous, ask the user to clarify before proceeding. Do not guess.

### 4. Select the agent pool and agents

Call `get_runtime_sources` to retrieve all agent pools, each with their agents, tags, and custom sources in one response.

Apply this selection logic:
- Select the pool and agents that correspond to the service in question
- If multiple pools are plausible, present the list to the user and ask which to use
- Prefer agents actively serving traffic relevant to the question (e.g. for a US-specific question, prefer agents handling US requests if identifiable)
- If the right agents are still ambiguous, present the options to the user and ask which to use

### 5. Collect runtime data

Call the appropriate Lightrun tool with:
- The file path and line number from step 3
- The selected agent(s) from step 4
- A sampling window — **default to 60 seconds**; extend to 5 minutes if the operation is infrequent or the first attempt returns no data

### 6. Analyse and answer

Interpret the returned data in the context of the user's question:
- Translate raw values into a plain-language answer
- If a distribution was collected, summarise the range, typical value, and any notable outliers
- If multiple capabilities were used, synthesise the results into a single coherent answer
- If the data is ambiguous or insufficient, say so clearly and suggest what additional instrumentation might help

---

## Error Handling

| Situation | Action |
|---|---|
| No agent pools found | Inform the user; ask them to verify the service is running and connected to Lightrun |
| Multiple plausible agent pools | Present the list to the user and ask which to use |
| Multiple plausible agents within a pool | Use the agents, tags, and custom sources returned by `get_runtime_sources` to narrow down; if still ambiguous, ask the user |
| Tool returns no data | Retry with a longer sampling window; if still empty, consider whether the code path is being actively exercised |
| Line cannot be instrumented | Try an adjacent line; if still unavailable, explain the limitation to the user |
| Code location unclear | Ask the user before proceeding — do not guess |

---

## Example Questions This Skill Can Answer

- How many available connections are in the DB connection pool?
- How many users are currently logged into service X?
- Give me an example of an HTTP request to our `POST /api/v1/users` endpoint
- Give me an example of the response JSON from the external sales tax service
- Is this cache key currently in use?
- How long does it take to process an average trade request?
- Is this list ever empty in production?
- What values can this expression take when `varX` is true?
- Which branch runs for customer X?

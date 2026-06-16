# Agent Migration & Reliability Upgrade Roadmap

---

# PHASE 0 — Quick Reliability Fixes (Do First, ~2 Hours)

## 1. Remove Artificial Response Truncation

### File
`orchestrator.py`

### Function
`_handle_real_estate_question`

### Location
~Lines 977–992

### Required Changes

#### Delete Rule 6

Remove the hard constraint:

> "Maximum 3 sentences"

This rule unnecessarily degrades answer quality and prevents the AI from providing complete responses.

#### Rewrite Rule 2

Current:

> "If the sources don't contain the answer, say 'I don't have specific data'."

Replace with:

> "Prefer information from the provided sources. If the sources are incomplete or do not directly answer the question, provide your best general answer based on available knowledge and clearly indicate that it is a general response."

---

## 2. Apply Identical Prompt Fix to Streaming Path

### File
`orchestrator_streaming.py`

### Location
Fallback prompt (~line 750)

### Required Changes

Apply the exact same modifications:

- Remove Rule 6 ("max 3 sentences")
- Replace Rule 2 with the generalized-answer behavior

---

## 3. Remove Dead-End Error Responses

### File
`orchestrator.py`

### Locations

- ~911–914
- ~927–931
- ~1008–1011
- ~1024–1029

### Current Behavior

Returns messages such as:

```text
Please try again.
```

### Required Behavior

Never return a dead-end response.

Instead:

1. Invoke the general-purpose LLM.
2. Generate the best available answer using model knowledge.
3. Include a soft disclaimer if data retrieval failed.

Example:

```text
I couldn't retrieve live market data right now, but generally speaking...
```

---

## 4. Remove Dead-End Streaming Errors

### File
`orchestrator_streaming.py`

### Locations

- ~768
- ~800–811

### Required Changes

Apply the same fallback behavior:

- No bare errors
- No "Please try again"
- Always return a useful response

---

# PHASE 1 — Agent Skeleton (Feature Flagged)

## Goal

Introduce an agent-based architecture while keeping the current orchestrator fully operational.

---

## 1. Create Agent Loop

### New File

```text
app/ai/agent.py
```

### Responsibilities

Implement a tool-calling loop:

```text
Model
  ↓
Tool Call
  ↓
Execute Tool
  ↓
Return Tool Result
  ↓
Model
```

Repeat until:

- Final answer generated
- Maximum ~5 hops reached

If exhausted:

- Generate fallback answer

---

## 2. Create Tool Registry

### New File

```text
app/ai/tools.py
```

### Responsibilities

- Tool JSON schemas
- Tool descriptions
- Tool dispatcher
- run_tool() implementation

---

## 3. Add Gemini Client

### New File

```text
app/ai/llm_client.py
```

### Environment Variable

```env
GEMINI_API_KEY=
```

### Config Updates

File:

```text
app/core/config.py
```

Add:

```python
GEMINI_API_KEY
GEMINI_MODEL
```

---

## 4. Feature Flag Routing

### File

`orchestrator.py`

### Function

`master_ai_agent`

### Location

~Line 563

### Required Change

Add feature flag:

```python
if USE_AGENT:
    return await agent_loop(...)
else:
    return existing_orchestrator(...)
```

Purpose:

- Side-by-side comparison
- Safe rollout
- Easy rollback

---

# PHASE 2 — Wrap Existing Logic as Tools

## Goal

Reuse existing code.

Do not rewrite business logic.

---

## Tool: answer_market_question

### Backend

Reuse:

```text
_handle_real_estate_question
```

Components:

- Tavily search
- Existing summarization pipeline

---

## Tool: get_property_listings

### Backend

Reuse:

```text
tavily_property_service.search_properties_stream
```

---

## Tool: get_market_prices

### Backend

Reuse:

```text
price_prediction_service
```

---

## Tool: calculate_affordability

### Backend

Reuse:

```text
_execute_analysis_calculation
```

### Requirement

Deterministic Python only.

No LLM calculations.

---

## Tool: calculate_yield

### Backend

Reuse:

```text
_execute_analysis_calculation
```

### Requirement

Deterministic Python only.

No LLM calculations.

---

## Tool: calculate_flip

### Backend

Reuse:

```text
_execute_analysis_calculation
```

### Requirement

Deterministic Python only.

No LLM calculations.

---

## Tool: search_platform_docs

### Backend

Reuse:

```text
guide_service
```

Source:

```text
_handle_platform_question
```

RAG remains unchanged.

---

## Tool: resolve_location

### New Tool

Purpose:

Resolve any global location.

---

## Resolver Refactor

### File

```text
resolver.py
```

### Current Problem

Location matching depends on:

```text
OFFICIAL_LOCATIONS
```

with only 7 hardcoded locations and embedding matching.

This creates the Dubai-only limitation.

### Required Change

Replace with geocoding API:

Options:

- Google Places
- Mapbox
- Nominatim

### Outcome

Location resolution becomes:

```text
Global
Dynamic
Provider-backed
```

This becomes the backend implementation of:

```text
resolve_location
```

---

# PHASE 3 — Retire Dead Weight & Harden System

## 1. Remove Query Rewriter Layer

### File

`orchestrator.py`

### Remove

Import:

```python
rewrite_query_with_context
```

(Line 19)

Call:

```python
rewrite_query_with_context(...)
```

(Line 577)

### Result

Delete:

```text
rewriter.py
```

Reason:

Modern agent performs context resolution natively.

---

## 2. Remove Intent Router Layer

### File

`orchestrator.py`

### Remove

Import:

```python
determine_intent
```

(Line 20)

Call:

```python
determine_intent(...)
```

(Line 578)

Dispatch block:

```python
if intent == ...
```

(Lines 581–630)

### Result

Delete:

```text
router.py
```

Reason:

Agent performs intent selection through tool choice.

---

## 3. Add Reliability Layer

### File

```text
app/ai/llm_client.py
```

### Add

#### Retry Logic

For:

- Rate limits
- Transient failures
- Network issues

#### Exponential Backoff

Example:

```text
1s
2s
4s
8s
```

#### Provider Fallback

On:

```text
429
Rate Limit
Quota Errors
```

Fallback to secondary provider.

---

## 4. Evaluation Framework

### New Directory

```text
tests/eval/
```

### Requirements

Build a regression suite of:

```text
~100 Real User Queries
```

Track:

- Tool selected
- Tool sequence
- Latency
- Final answer
- Pass/Fail

### Observability

Use:

- Langfuse
- Phoenix

for trace-level monitoring.

---

# PHASE 4 — Cutover

## Production Migration

Flip the feature flag:

```python
USE_AGENT = True
```

for 100% traffic.

Monitor:

- Evaluation suite
- Production logs
- Tool usage patterns
- Latency

---

## Post-Cutover Optimization

Iteratively refine:

```text
app/ai/tools.py
```

especially:

- Tool descriptions
- Tool examples
- Tool selection criteria

whenever the model selects the wrong tool.

---

# Non-Negotiables

## 1. Financial Calculators Must Remain Deterministic

Never allow the LLM to perform:

- Affordability calculations
- Rental yield calculations
- Flip analysis calculations

All financial math must remain deterministic Python.

```text
LLM = Reasoning
Python = Calculation
```

---

## 2. No Error Path May Dead-End

The system must never return:

```text
Please try again.
Something went wrong.
Error retrieving data.
```

Every failure path must still provide:

- A useful answer
- General guidance
- Best-effort reasoning
- A soft disclaimer when required

The user should always receive value, even when upstream services fail.

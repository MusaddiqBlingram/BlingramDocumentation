# SYSTEM SPECIFICATION: Support AI State Machine

**To:** Backend Engineering (Fahad)  
**Subject:** Integration requirements for the upgraded AI Support Orchestrator

---

# 1. Architectural Overview

We have completely re-engineered the Support AI. It is no longer a standard conversational chatbot. It is a Deterministic State Machine (FSM).

Its primary function is to ingest unstructured user complaints, sanitize them, validate them against strict rules, and output structured JSON payloads to trigger database actions.

## The Pipeline

### Rewriter AI

Intercepts the user's message, looks at the recent chat history, and resolves any pronouns or shorthand into a standalone sentence. It has built-in safety trips to drop profanity.

### The Validation Gate

The main Support AI evaluates the sanitized string. If the user says "hi", "thanks", or asks non-technical questions, the AI refuses to execute and returns a standard conversational string.

### Dynamic Tool Pruning

If the input is a valid technical issue, the AI executes a tool. The tool it is allowed to use is dynamically pruned based on the current database state.

---

# 2. The State Contract (`active_ticket_id`)

This is the most critical integration point. The AI is stateless by default. It does not inherently know if the user already opened a ticket 30 seconds ago. The backend must inject this state.

In `app/services/supportchat_streaming.py`, the core function signature is:

```python
async def process_support_message_streaming(
    self,
    message: str,
    session_id: Optional[str],
    user: Optional[User] = None,
    active_ticket_id: Optional[str] = None  # <-- INJECTION POINT
):
```

## How it works

### If `active_ticket_id` is `None`

The AI is only given the `submit_bug_report` tool.

It will create a new ticket in the database.

### If `active_ticket_id` has a value (e.g., `"TKT-123"`)

The AI is physically blocked from creating a new ticket.

It is only given the `append_to_ticket` tool to add notes to the existing ticket.

## Your Action Item

You must design how the API tracks this across a user's session.

You have two options:

### Frontend Passed

The React frontend holds the `ticket_id` in state after the first creation, and passes it back in the request body for subsequent messages.

### Backend Redis/Postgres

When the AI creates a ticket, you save:

```text
session_id -> active_ticket_id
```

in Redis.

On the next incoming message, you pull it from Redis and pass it into the function.

---

# 3. Required Database Methods

The AI outputs structured JSON to trigger methods in `SupportService`.

You need to ensure the following methods exist and handle the SQLAlchemy layer.

## A. create_bug_report (Already exists, ensure payload matches)

```python
# Triggered when a new issue is detected
bug_payload = BugReportCreate(
    title="...",
    description="..."
)

bug_report = await self.support_service.create_bug_report(
    payload=bug_payload,
    current_user=user
)
```

## B. append_to_bug_report (NEEDS TO BE BUILT)

When the AI detects a follow-up issue, it triggers an append action.

You must build this method in:

```text
app/services/support_service.py
```

### Invocation

```python
# In supportchat_streaming.py:
elif tool_name == "append_to_ticket":
    args = json.loads(tool_arguments)

    # FAHAD: Build this method to add a comment or append text to the description
    await self.support_service.append_to_bug_report(
        ticket_id=active_ticket_id,
        additional_info=args.get("additional_info")
    )
```

---

# 4. Restore the Core LLM Client

The `app/core/llm.py` file was accidentally deleted during local prototyping.

Please recreate it.

This uses a Singleton pattern so FastAPI doesn't open 500 connections to Groq under load.

## File: `app/core/llm.py`

```python
import logging
import os
from groq import AsyncGroq

logger = logging.getLogger(__name__)

class GroqClientWrapper:
    def __init__(self):
        api_key = os.environ.get("GROQ_API_KEY")
        self.client = AsyncGroq(api_key=api_key)
        self.model = os.environ.get(
            "GROQ_MODEL",
            "llama-3.3-70b-versatile"
        )

    async def generate(
        self,
        messages,
        tools=None,
        temperature=0.1
    ):
        return await self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            tools=tools,
            temperature=temperature,
        )

    async def generate_stream(
        self,
        messages,
        tools=None,
        temperature=0.1
    ):
        stream = await self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            tools=tools,
            temperature=temperature,
            stream=True
        )

        async for chunk in stream:
            if chunk.choices:
                yield chunk.choices[0].delta

_llm_client_instance = None

def get_llm_client() -> GroqClientWrapper:
    global _llm_client_instance

    if _llm_client_instance is None:
        _llm_client_instance = GroqClientWrapper()

    return _llm_client_instance
```

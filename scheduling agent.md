🛠️ Scheduling System Logic Fixes
1. Intent Mapping Optimization
File: app/ai/scheduling_orchestrator.py

Broadened Triggers: Added "what" and "when" keywords to the LIST_SLOTS intent to capture natural queries like "What are the slots?" or "When can I see the place?"

Aggressive Booking Catch: Added "visit" and "viewing" to REQUEST_BOOKING to ensure the agent moves to the booking flow immediately when the user shows intent.

Direct Date/Time Recognition: Modified the orchestrator to trigger the booking flow immediately if the natural language parser detects a date or time in the user's message, bypassing generic chat.

2. Hidden Metadata Injection (Context Grounding)
File: app/services/scheduling_chat_service.py

The Problem: The LLM would occasionally "forget" which specific slot ID it was discussing in a multi-turn conversation.

The Fix: Updated get_conversation_context to inject a System Hidden Context into the assistant's history.

Mechanism: If a slot_id exists in the message metadata, it is appended to the message content as a system hint:

[SYSTEM HIDDEN CONTEXT: The active slot_id for this interaction is 'UUID']

Result: This ensures the LLM always has the correct database IDs ready for tool calling without the user seeing the "ugly" UUIDs in the chat UI.

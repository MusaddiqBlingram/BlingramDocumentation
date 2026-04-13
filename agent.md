# 🧠 Architecture Overview: The B2B Agent Persona (Bellokey Copilot)

**To:** Fahad (Frontend Lead)  
**From:** Backend Engineering  
**Subject:** Implementation Details for the AGENT Persona  

---

## 1. Executive Summary

Bellokey’s AI Engine now operates on a **split-cognitive architecture**. Depending on who logs in, the Orchestrator routes the query to a completely different "brain."

The **Agent Persona** is our B2B Copilot designed exclusively for licensed Real Estate Brokers. It is not designed to help families find homes; it is designed to act as a **high-speed quantitative analyst** for brokers to manage leads, generate Comparative Market Analyses (CMAs), and close deals faster.

---

## 2. Core Capabilities & Use Cases

When `viewer_role == "AGENT"`, the AI is unlocked to perform enterprise-level real estate tasks:

- **Instant Market Analysis:**  
  Calculates and summarizes:
  - Average price-per-square-foot  
  - Days on Market (DOM)  
  - Inventory gaps  

- **Lead Matching:**  
  Takes raw constraints (e.g., *"Find 4-bed villas in Marina under 5M"*) and instantly searches the database to match CRM leads.

- **Multi-Tier Search Fallback:**  
  If internal inventory is insufficient:
  1. Internal Database  
  2. External Cache  
  3. Live Web (Tavily → PropertyFinder / Bayut)

- **Market Intelligence:**  
  Aggregates:
  - News  
  - Market trends  
  - Internal metrics  

  Example: *"What are the price trends in Downtown Dubai for Q3?"*

---

## 3. Cognitive & Linguistic Rules (The Vibe)

The LLM is heavily restricted to behave like a **senior analyst**, not a chatbot.

- **Zero Fluff:**  
  No explanations of basic concepts (e.g., mortgages)

- **Hyper-Concise:**  
  Maximum **3–4 sentences per response**

- **Data-First:**  
  Defaults to **bullet points** for metrics

- **No AI Apologies:**  
  Never says:
  - "As an AI language model..."
  - "I’d be happy to help..."

---

## 4. Frontend Integration Requirements (Crucial for UI)

### A. The "Invisible" Property List

When a broker searches:

- **Backend Payload:**
```json
{
  "intent": "SEARCH",
  "properties": [...],
  "message": "Found 12 matches. The average price in this batch is 1,850 AED/sqft."
}
```

- **Frontend Action:**
  - Do **NOT** display properties in chat text  
  - Render `properties[]` as **visual Property Cards**  
  - Display `message` as a short analytical summary  

---

### B. State Transitions (Missing Data)

If input is vague (e.g., *"Find me villas for my client"*):  

- **Backend Behavior:**
  - No search executed  
  - Transitions to **data collection state**

- **Response Example:**
```
"What is the budget and preferred sub-community?"
```

- **Frontend Action:**
  - Render text  
  - Capture user input  
  - Send next message → Orchestrator  

---

## 5. Security & Operational Firewalls (Anti-Jailbreak)

The Agent Persona is protected by a strict **Domain Lock**.

---

### Total Exclusivity

If user asks for anything outside real estate:

- Python code  
- Recipes  
- Trivia  
- Politics  

➡️ The system **hard-blocks** the request.

---

### The Pivot (Standard Response)

```
"I am a real estate analytics tool. I cannot assist with that. How can I help you manage your listings or client searches today?"
```

- No apology  
- No deviation  

---

### Compliance Lock

The AI is strictly blocked from:

- Legal contract advice  
- Licensing compliance data  

---

## 🚀 Status

- Routing logic ✅  
- Async processing ✅  
- Search fallbacks ✅  

All systems are fully operational in `orchestrator.py`.

**The endpoint is ready for frontend integration.**

# Architecture Decision Record: AI Recommendation Engine Refactor


---

## Executive Summary

The AI engine has transitioned from a **probabilistic chatbot** to a **deterministic, fault-tolerant state machine**.

To combat LLM unpredictability—such as **schema hallucinations** and **invalid web links**—we have implemented a **"Defense in Depth" strategy**.

The AI is now encapsulated within:

- Validation shields  
- Data coercion layers  
- A 3-tier fallback system  

This ensures the frontend receives only **high-integrity, structured data**.

---

## 1. The Extraction Layer (`extractor.py`)

### The Problem: Semantic Ambiguity & Malformed JSON

Previously:

- The system relied on **implicit LLM understanding** of real estate nuances (e.g., rent vs. buy).
- LLM "laziness" often returned **strings instead of arrays**  
  - Example: `"apartment"` instead of `["apartment"]`
- This triggered **500 Internal Server Errors** during Pydantic validation.

---

### The Engineering Fix: Strict Contracts & Data Coercion

#### Explicit Schema Definition

- Introduced `transaction_type` into the **SearchFilters Pydantic model**.
- The LLM is strictly instructed to classify intent as:
  - `"buy"`
  - `"rent"`

**Example Rule:**

- `"per month"` → classified as `"rent"`

This eliminates **database mismatches**.

---

#### The Auto-Correction Shield

Before Pydantic validation, the payload passes through a **recursive coercion layer** that:

- Strips arbitrary wrappers  
  - Example: removes nested `{"properties": {...}}`
- Detects and converts:
  - Single strings → arrays  
    - `property_type`
    - `location`

---

## 2. The Orchestration & Routing Layer (`orchestrator.py`)

### The Problem: Single Points of Failure & Memory Bloat

- The system **failed completely** if internal queries returned no results.
- Heavy persona prompts were **imported globally**, wasting memory for users who didn’t need them.

---

### The Engineering Fix: 3-Tier Fallback & Lazy Loading

---

### A. The Routing Pipeline

When `intent == "SEARCH"`, the system executes a **cascading fallback**:

#### Tier 1 (Internal)

- High-fidelity **PostgreSQL Property table**

#### Tier 2 (External Cache)

- `ExternalProperty` table  
- Contains scraped data **< 7 days old**

#### Tier 3 (Live Web)

- `TavilyPropertyService`  
- Performs **real-time scraping**

---

### B. Memory Optimization

- Persona modules:
  - `investor_logic.py`
  - `user_logic.py`

Use **Just-in-Time (JIT) imports** inside execution blocks.

**Result:**

- No memory allocated for investor tools during a normal user session  
- Reduced RAM footprint  
- Improved scalability  

---

## 3. The Validation Airlock (Tavily Integration)

### The Problem: Garbage Web Data

Live searches often returned:

- Irrelevant URLs (Reddit, forums)  
- Broken listings (e.g., `0 AED`)  

This caused **frontend UI crashes**.

---

### The Engineering Fix: Strict URL & Data Filtering

The **"Airlock" pattern** enforces a rigid filter.

---

### Allowed Real Estate Domains

| Domain              | Status       |
|--------------------|-------------|
| propertyfinder.ae  | Whitelisted |
| bayut.com          | Whitelisted |
| dubizzle.com       | Whitelisted |
| All others         | Dropped     |

---

### Validation Rules

#### Domain Strictness

- Any non-whitelisted URL → **instantly discarded**

#### Price Integrity

- Listings with:
  - `price <= 0`
  - Missing currency  

→ **dropped**

---

### Prompt Hardening

- The LLM is **forbidden** from outputting raw URLs in chat text
- It generates only **conversational responses**

The backend attaches:

- `property_items` (JSON)

The frontend renders:

- Clean UI cards  
- Structured listing displays  

---

## 4. DevOps Action Required

The logic is verified via:

```
test_ai.py (with db_connection=None)
```

---

### Required Environment Variables

#### DATABASE_URL

- Must point to the **staging PostgreSQL instance**

#### TAVILY_API_KEY

- Required for **Tier 3 live scraping**

---

### Deployment Target

Once variables are configured:

- `/api/v1/chat/` endpoint is cleared for  
  **full-scale integration testing**

---

## Next Step

Would you like me to draft the **Nginx** or **Docker configuration** required to securely inject these environment variables during the staging build?

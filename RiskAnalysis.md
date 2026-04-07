📑 Module Specification: Risk Analysis Engine

1. Executive Summary

The risk_analysis module is a mission-critical component of the Bellokey platform. It provides a standardized, real-time risk profile for any global real estate asset. Unlike traditional static models, this engine uses a Hybrid Intelligence approach:

Tavily provides the "Eyes" (Live Web Data).  
Llama 3.1 provides the "Brain" (Qualitative Extraction).  
Python provides the "Guardrails" (Deterministic Logic).  

2. The Three-Tier Pipeline Architecture

The engine operates as a linear, asynchronous pipeline to minimize latency and maximize data accuracy.

Tier 1: Real-Time Scoped Search (Tavily)

The system bypasses the LLM's outdated training data by executing a targeted, deep-web search.

Strategy: It bundles three distinct macro-economic queries into a single API call to save credits and reduce I/O wait times.

Target Data: Central bank rates (Country), Market volatility (City), and Local zoning/climate risks (Neighborhood).

Tier 2: Semantic Translation (Groq / Llama 3.1)

The raw text returned from Tier 1 is unstructured chaos. The LLM acts as a high-speed parser.

Model: llama-3.1-8b-instant

Configuration: temperature=0.0 and response_format={"type": "json_object"}.

Goal: Coerce human language (news reports, PDF snippets) into integer severity scores (0–100) and semantic labels (Low, Medium, High).

Tier 3: Deterministic Logic & Business Overrides (Python)

AI is probabilistic and cannot be trusted with final business logic.

Vacancy Math: Calculated via a pure Python function:

$Vacancy \% = 100 - Occupancy \%$

The "Cash Is King" Override: If a property is flagged as is_property_financed=False, the system ruthlessly overwrites any AI-generated interest rate risk to 0.

Standardization: The overall_score is a simple arithmetic mean of all four factors, ensuring the frontend always gets a predictable integer.

3. Technical Data Contract

Input Parameters (build_risk_analysis_payload)

Variable | Type | Requirement | Description
---------|------|-------------|------------
property_id | str | Required | UUID of the property.
city_name | str | Required | Metropolitan area for volatility check.
neighborhood_name | str | Required | Specific district for regulatory/climate check.
country_name | str | Required | Sovereign nation for interest rate check.
occupancy_rate_pct | float | Required | Used for the deterministic vacancy formula.
is_property_financed | bool | Required | Determines if interest rates apply to the user.

Output Schema (JSON)

```json
{
  "context": {
    "property_id": "prop_test_101",
    "status": "success"
  },
  "risk_analysis": {
    "overall_score": 42,
    "factors": [
      {
        "name": "Market Volatility",
        "severity_score": 70,
        "severity_label": "High"
      }
      /* ... 3 other factors ... */
    ]
  }
}
```

4. Engineering Standards & Guardrails

🛠️ Defensive Naming

We utilize Self-Documenting Variable Names. Instead of score, we use overall_average_risk_score. This prevents "Scope Creep" where a developer might accidentally use a sub-score as a total.

⚡ Concurrency

The entire module is built using asyncio. This allows the dashboard_orchestrator to fire the Risk Engine and the Exit Strategy Engine in parallel, reducing total API response time from ~8 seconds to ~4 seconds.

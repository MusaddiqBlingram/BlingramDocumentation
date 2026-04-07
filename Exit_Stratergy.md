

# 📑 Technical Specification: Exit Strategy Intelligence Engine

## 1. Executive Summary

The `exit_strategy` module is a **high-precision forecasting engine**. It calculates the **Total Yield** of a property by combining real-time market sentiment with time-series compounding math.

It is designed to answer the investor's most critical question:

> "When is the mathematically optimal moment to sell?"

---

## 2. The Forecasting & Compounding Pipeline

The engine utilizes a **"Bimodal Architecture"**—it uses:

- AI for **Market Intelligence**
- Pure Python for **Financial Compounding**

---

### Stage 1: Market Cycle Discovery (Tavily)

**Goal:** Identify the **Macro-Trend** of the specific city.

**Query Vector:**

- Searches for **10-year historical appreciation averages**
- Retrieves **3-year local forecasts**

**Insight Extraction:**

- Detects **"Black Swan" events**, such as:
  - New zoning laws  
  - Major infrastructure (e.g., new Metro lines)  
  - Tax changes  

These signals help trigger a **"Sell"** or **"Hold"** recommendation.

---

### Stage 2: Scenario Modeling (Llama 3.1 & Groq)

The LLM converts qualitative news into quantitative **Growth Coefficients**.

#### The Three Scenarios:

1. **Bull Run**  
   - Best-case annualized growth  
   - **Capped at 25%** to prevent hallucinations  

2. **Stable Market**  
   - Likely case based on historical averages  

3. **Market Correction**  
   - Stress-test scenario  
   - Usually negative or near-zero  

---

#### The Exit Window

The AI identifies **specific quarters** (e.g., `"Q3 2027"`) where supply-demand curves suggest a market peak.

---

### Stage 3: The Compounding Engine (Python)

AI is notoriously bad at **multi-step math**, so all financial calculations are handled in deterministic Python.

#### Future Value Formula

$$
FV = PV \times (1 + r)^t
$$

---

#### Total Profit Logic

The engine evaluates:

- **Compounded Property Value**
- **Accumulated Rental Income (over 5 years)**

---

#### Yield Result

$$
\text{Projected Profit} =
(\text{Compounded Property Value} + \text{Total Rent}) - \text{Purchase Price}
$$

---

## 3. Data Contract & Schema Specification

### A. Functional Signature

```python
async def build_exit_strategy_payload(
    property_id: str,
    city_name: str,
    purchase_price_local: float,
    monthly_rent_local: float,
    currency_code: str = "USD"
) -> Dict[str, Any]
```

---

### B. Output Object Definition (The "UI Contract")

The frontend uses this data to render the **Scenario Comparison Table** and the **Optimal Exit badge**.

| Key                 | Type  | Constraints                     | Purpose                                      |
|--------------------|------|--------------------------------|----------------------------------------------|
| optimal_timing      | dict | Includes window and tags       | Tells the user **When and Why**              |
| appreciation_pct    | float| Annualized rate                | Shows the "Aggressiveness" of the model      |
| projected_profit    | int  | Rounded local currency         | Final **Net Gain** after 5 years             |
| currency            | str  | ISO Code (INR, AED, USD)       | Used for frontend currency formatting        |

---

## 4. Engineering Standards

### 📏 Guardrails: The "Logic Hierarchy"

To prevent the AI from giving nonsensical advice, we enforce a **Mathematical Hierarchy in Python**.

---

#### The Correction Flip

If:

- AI predicts **Correction > Stable**

Then:

- Python **forces Correction rate to be 5% lower** than Stable  

---

#### The Hallucination Cap

If:

- AI predicts **annual growth > 25%**

Then:

- Python **clips it to 18%**  

This avoids confusion between **long-term vs short-term growth rates**.

---

### ⚡ Global-Ready Logic

#### Currency Agnostic Design

By using variables like:

```python
purchase_price_local
```

The system works seamlessly for:

- ₹1.5 Cr villa in Bangalore  
- $1.5M penthouse in Las Vegas  

---

#### Separation of Concerns

- **Backend:** Sends raw numeric values  
- **Frontend (React):** Handles:
  - Currency symbols (`₹`, `$`, `AED`)  
  - Number formatting (commas, decimals)  

This keeps the backend **clean, universal, and scalable**.

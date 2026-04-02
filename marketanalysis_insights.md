# Backend Engineering Specification: Market Intelligence Aggregation

## 📌 Document Purpose
This document defines the requirements for the backend database layer to aggregate raw real estate data. This data serves as the primary input for the **Python Insights Engine**.

## 🏗 Architectural Rule
* **Responsibility:** The backend is strictly responsible for **data retrieval** and **spatial aggregation**.
* **Separation of Concerns:** All business logic, percentage change calculations (MoM/YoY), and natural language generation are handled by the Python middle-layer. 
* **Constraint:** Do not perform percentage math in the SQL/Database layer.

---

## 1. The Spatial Boundary (The Core Filter)
Real estate markets are hyper-local. All metrics must be filtered geographically before calculation.

* **Logic:** Extract `latitude` and `longitude` for the `property_id`.
* **Execution:** All subsequent queries must be constrained by a **bounding box** or **radial distance** (e.g., 3km radius) centered on those coordinates.

---

## 2. Market Velocity: Days on Market (DOM)
Determines the "speed" of the market to identify Buyer vs. Seller trends.

* **Target Variables:** `average_days_on_market_current_month`, `average_days_on_market_previous_month`
* **Calculation:** 1. Filter for `status = SOLD` within the Spatial Boundary.
    2. Group by current month and previous month.
    3. Calculate $DOM = sale\_date - listing\_date$.
    4. Return the mathematical mean of these differences.
* **⚠️ Engineering Traps:** * **Exclude Active Listings:** Only use completed sales; active listings have incomplete DOM.
    * **Trim Outliers:** Exclude properties where $DOM = 0$ or $DOM > 365$.

---

## 3. Supply Constraints: Seasonal Inventory
Compares current supply against historical seasonal norms.

* **Target Variables:** `total_active_listings_current_month`, `historical_average_active_listings_for_this_month`
* **Calculation:**
    * **Current:** Count properties where `status = ACTIVE` within the boundary.
    * **Historical:** Query the count of `ACTIVE` listings for this specific calendar month over the previous 3 to 5 years (e.g., if current is Oct 2026, average Oct 2025, 2024, and 2023).
* **⚠️ Engineering Traps:** Never compare October to September. You must compare October to the historical "October Average" to account for seasonality.

---

## 4. Affordability: Median Income vs. Home Price
Detects if local residents are being priced out of the market.

* **Target Variables:** `average_home_price_in_market_area_current_month_usd`, `median_household_annual_income_for_market_area_usd`
* **Calculation:**
    * **Home Price:** Median/Average sale price in the Spatial Boundary for the current month.
    * **Income Data:** Integrate with 3rd-party demographic APIs (Census, Tax Aggregates) using the property’s **Zip Code** or **Geohash**.
* **⚠️ Engineering Traps:** You **MUST** use **Median Household Income**. Mean (Average) income is easily skewed by high-net-worth outliers.

---

## 5. API Contract (Output Requirements)
The backend must return a flat JSON structure. In the event of missing data or zero-division errors, return `null` or `0.0` to allow the Python engine to handle degradation.

### Required JSON Payload
```json
{
  "target_property_unique_identifier": "string",
  "property_valuation_current_month_usd": "float",
  "property_valuation_previous_month_usd": "float",
  "average_price_per_square_foot_current_month_usd": "float",
  "average_price_per_square_foot_previous_month_usd": "float",
  "average_home_price_in_market_area_current_month_usd": "float",
  "average_home_price_in_market_area_previous_month_usd": "float",
  "average_days_on_market_current_month": "integer",
  "average_days_on_market_previous_month": "integer",
  "total_active_listings_current_month": "integer",
  "historical_average_active_listings_for_this_month": "integer",
  "median_household_annual_income_for_market_area_usd": "float",
  "historical_twelve_month_price_trend_data": [
      {"month": "string", "average_price": "float"}
  ]
}

# Architecture Spec: Bellokey Distressed Deals Engine

## 1. System Philosophy

This system is a **Native Anomaly Detection & Recommendation Engine**. It does not rely on external web scraping or heavy predictive ML. It evaluates properties submitted directly by Bellokey users/agents in real-time, utilizing deterministic mathematical gates and NLP to identify high-probability distressed assets. Once verified, these assets receive an algorithmic ranking boost in the main user feed.

---

## 2. The Detection Pipeline

When a property is created or updated in the database, it must pass through one of three specific logic gates to be flagged with the `is_distressed = True` state.

---

### Route A: The Qualitative Trigger (NLP)

**Target:** Seller psychology and desperation.

**Logic:**  
An in-memory Regex scan evaluates the description string.

**Trigger Condition:**  
If the description contains predefined crisis vocabulary (e.g., `"urgent"`, `"distressed"`, `"leaving country"`, `"cash buyer only"`).

**Result:**  
`is_distressed = True`

---

### Route B: The Quantitative Trigger (New Listings)

**Target:** Immediate aggressive underpricing on market entry.

**Logic:**  
Compares the new listing's price per square foot against the cached mean of its immediate spatial baseline (the neighborhood or specific building).

**Trigger Condition:**  
The property must be priced at least **10% below the neighborhood average.**

`Price_new <= 0.90 * Mean_neighborhood`

**Result:**  
`is_distressed = True`

---

### Route C: The Anti-Exploit Trigger (Existing Listing Price Cuts)

**Target:** Catching genuine price drops while completely blocking **"Fake Discount" exploits** by manipulative agents.

**Logic:**  
A **Two-Key Verification system**. A price drop is only valid if it crosses the baseline of market reality.

**Trigger Condition:**

**Key 1 (Velocity):**  
The price must drop by **≥ 10%**.


frac{Price_{old} - Price_{new}}{Price_{old}} \ge 0.10

**Key 2 (Floor):**  
The new discounted price must strictly cross below the neighborhood average.

**Key 1 (Velocity):**  
The price must drop by **≥ 10%**.

`(Price_old - Price_new) / Price_old >= 0.10`

**Result:**  
Both keys must return **True** to set `is_distressed = True`. If **Key 2 fails**, the drop is ignored as an exploit.

---

## 3. The Algorithmic Output (Feed Integration)

Properties that successfully achieve the `is_distressed = True` state are intercepted before standard feed indexing.

**Action:**  
A **visibility multiplier** is applied to their ranking score, forcing them to the top of the Bellokey smart feed or populating a dedicated **"Urgent Deals" UI carousel**.

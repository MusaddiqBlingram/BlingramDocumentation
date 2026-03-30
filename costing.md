# CONFIDENTIAL MEMO: Bello AI Unit Economics & COGS

## Executive Summary
This document outlines the **worst-case Cost of Goods Sold (COGS)** to support a single active user on Bello AI for one month. 

Because we are currently operating without an internal property database, 100% of user queries must be routed through **Tavily** to scrape live real estate portals (MagicBricks, 99acres, etc.) in real-time. The AI (**Groq**) then processes this raw web data to generate a response.

---

## Baseline Assumptions
* **Usage:** 500 queries per user, per month.
* **Architecture:** 100% reliance on live external web scraping (Tavily). Zero internal database hits.
* **Payload & Entropy:** Heavy data extraction. Accounts for a **~20% CAPTCHA/blocking failure rate**, forcing the system to consume duplicate API credits for retries.

---

## Total Monthly Cost Per User

| Infrastructure Layer | Monthly Cost (per user) |
| :--- | :--- |
| **Data Retrieval** (Tavily API + Retries) | $9.00 |
| **AI Processing** (Groq API + Error Handling) | $0.40 |
| **Infrastructure Buffer** (Network/Bandwidth) | $1.10 |
| **TOTAL MONTHLY COGS** | **$10.50** |

---

## Business Impact & Margin Analysis
At a worst-case baseline cost of **$10.50 per user per month**, our gross margins are heavily exposed to third-party API limits and external website defenses.

### Example Scale Impact:
* **100 Active Users** = $1,050 / month API overhead
* **1,000 Active Users** = $10,500 / month API overhead
* **10,000 Active Users** = $105,000 / month API overhead

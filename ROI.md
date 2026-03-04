# Bellokey System Architecture: The "Airlock" & Dual-Oracle ROI Engine

### Phase 1: The Airlock (State Management)
Before we build the sensors or the math engine, we build the "Airlock." This is the database layer that permanently separates the chaotic outside world from your delicate internal ROI calculator.

* **Provision a Lightweight Cache:** Set up a very small, cheap database on your cloud provider (like Redis or a single table in PostgreSQL).
* **Define the Schema:** Create exactly one record that holds two numbers:
    * `macro_stress_multiplier` (Default: 1.0)
    * `geopolitical_sentiment_score` (Default: 1.0)
* **The Rule:** Your main Bellokey application is never allowed to talk to the internet to get news or stock data. It is only allowed to read from this internal Airlock.

---

### Phase 2: Oracle Alpha - The Quantitative Sensor (Macro Data)
This is the automated pipeline that measures global financial panic.

* **The Trigger:** Set up a Serverless Cron Job (e.g., AWS EventBridge + Lambda) scheduled to fire every 6 hours.
* **The Fetch:** When it wakes up, it pings the free FRED API to request the exact current value of the VIX (the Volatility Index / Fear Gauge).
* **The Logic Transformation:** * If VIX is normal (< 20): Set `macro_stress_multiplier` = 1.0 (No stress).
    * If VIX is elevated (20-30): Set `macro_stress_multiplier` = 0.9 (Slight market contraction).
    * If VIX is spiking (> 30): Set `macro_stress_multiplier` = 0.7 (Severe crash/liquidity freeze).
* **The Write:** The Cron Job writes this single multiplier into your Airlock Database and goes back to sleep.

---

### Phase 3: Oracle Beta - The Qualitative Sensor (Path 1: Outsourced NLP)
This is the pipeline that reads the news to detect wars, sanctions, or local crises in Dubai, using the free compute you selected.

* **The Trigger:** A second Serverless Cron Job, also waking up every 6 hours.
* **The RSS Scraper:** The script uses a basic feed parser to download the latest XML headlines from regional news sources (e.g., Al Jazeera, Reuters Middle East).
* **The API Handoff:** The script bundles the top 10 headlines related to "Dubai", "UAE", or "Real Estate" and sends them via an HTTP POST request to the Hugging Face Serverless Inference API.
* **The Remote Compute:** Hugging Face's supercomputers process the text through a heavy transformer model and return a simple JSON response (e.g., 85% Negative, 15% Positive).
* **The Logic Transformation:**
    * If sentiment is overwhelmingly Negative (e.g., words like "attack", "war", "crash" dominate): Set `geopolitical_sentiment_score` = 0.6 (High probability of a shock).
    * If neutral/positive: Set it to 1.0.
* **The Write:** The script writes this score to the Airlock Database and shuts down.

---

### Phase 4: The Physics Engine (The Monte Carlo Simulator)
This is where the magic happens when a user actually clicks "Calculate ROI" on a Bellokey property listing.

* **Read the Baseline:** The backend pulls the static data of the property: Current Price, Expected Rent, and Standard Expected Appreciation (e.g., 5% a year).
* **Read the Oracles:** The backend instantly reads the two current multipliers from the Airlock Database.
* **The Loop (1,000 Iterations):** The backend initiates a loop that will run 1,000 times in a fraction of a second.
* **The Dice Roll (Probability Injection):** Inside each loop, it projects the 5-year growth. However, it uses a random number generator influenced by your Oracle multipliers.
    * **Example:** If the `geopolitical_sentiment_score` in the database is 0.6 (Bad News), the system forces 40% of the 1,000 loops to simulate a sudden 20% drop in property value in Year 2.
    * If the database scores are normal (1.0), only 5% of the loops simulate a random market correction.
* **The Math:** For every single loop, it calculates the standard Discounted Cash Flow (DCF) formula to find the final ROI percentage for that specific timeline.

---

### Phase 5: The Output Abstraction (The Fan Chart)
You now have an array of 1,000 different possible ROI percentages. You do not show 1,000 numbers to the user; you abstract the complexity.

* **Percentile Aggregation:** You sort the 1,000 results from worst to best.
* **The 5th Percentile:** This is your "Worst Case Scenario" (The bottom edge of the fan chart).
* **The 50th Percentile:** This is the "Most Likely Scenario" (The middle line).
* **The 95th Percentile:** This is the "Best Case Scenario" (The top edge).
* **The UI Render:** You send these three data arrays to the frontend (using a charting library).
* **The "Smart" Alert:** If the 5th Percentile drops below 0% (meaning the simulated war/crash wiped out the rental gains), the UI displays a specific warning: *"Elevated Macro Risk Detected: Stress tests indicate potential negative equity under current geopolitical conditions."*

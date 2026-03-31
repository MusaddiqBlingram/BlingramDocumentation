# 📐 The First Principle: Recommendation is Geometry

> **Bello AI is not searching a database; it is navigating a Latent Space.**

Imagine a universe with **768 dimensions**. Every property is a single coordinate point in that universe. Every user is also a single coordinate point in that exact same universe. 

The entire goal of our application is to measure the physical distance between the User’s point and the Properties' points. **The shorter the distance, the better the recommendation.**

To make this geometry happen in under **100 milliseconds**, the architecture is split into two distinct physical pipelines: 
1. **The Offline Sieve** (Data Ingestion)
2. **The Online Funnel** (Data Retrieval)

---

## 🛑 PART 1: The Offline Sieve (Data Ingestion)

This pipeline runs in the background when an agent creates a listing. It does not affect the user's app speed.

> **The Problem:** Real estate agents write garbage. They write *"Stunning luxury 2BHK"* when it's a rotting shack. If we put that raw text into a mathematical vector space, the math breaks. The word "luxury" acts like a black hole, pulling the property toward rich users who will hate it.

**The Solution:**

* 🛡️ **The Bouncer:** We pass the agent's text through a **Fast LLM**. The LLM acts as a ruthless filter. It strips out adjectives and opinions, extracting only cold, hard facts (e.g., *"east-facing"*, *"marble floors"*, *"A-Khata"*).
* 🗄️ **The JSONB Vault:** These clean facts are stored in a Postgres `JSONB` column with a `GIN` index. This allows us to do blazing-fast subset queries later.
* 🧮 **The Mathematician:** We take those flat facts and feed them to Google's **SigLIP model**. SigLIP compresses the concepts into a 768-dimensional vector (an array of floats). This is saved to a `Vector(768)` column in Postgres.

*The property is now permanently plotted on the map. The AI's job is done.*

---

## ⚡ PART 2: The Online Funnel (The User Execution Loop)

This is the real-time engine. A user opens the app, and we have roughly **100 milliseconds** to put the perfect 10 houses on their screen. We achieve this through a 4-Stage Funnel.

### Stage 0: The Origin Point (Memory)
Before we can search, we need to know where the user is standing on the map.

* **Implicit Signals:** When a user stares at a house for 10 seconds, a background worker uses an **Exponential Moving Average (EMA)** to drag the user's vector 15% closer to that house's vector.
* **Explicit Signals:** If the user types *"Farmhouse"* in the search bar, we instantly convert *"Farmhouse"* to a vector and drag the user's profile 40% toward it.

**🎯 The Result:** The user's vector represents their current "Vibe". This is the exact coordinate we hand to the database.

### Stage 1: Candidate Generation (The Scatter)
We cannot scan 100,000 properties—it would take seconds. Instead, we fire four database queries at the exact same millisecond using **Asynchronous I/O multiplexing**:

| Strategy | Technology | Purpose |
| :--- | :--- | :--- |
| **The Map** | PostGIS | Grabs everything within a 5km radius using a spatial index. |
| **The Vibe** | pgvector (HNSW) | Uses the user's vector to traverse a graph, finding the 300 visually/thematically closest properties. |
| **The Facts** | pgvector (HNSW) | Finds the 300 properties that semantically match the user's historical feature demands. |
| **The Chaos** | Randomizer | Grabs 100 completely random, highly-verified properties to prevent "echo chambers" (Overfitting). |

**📊 Funnel Status:** We pulled ~1,000 properties into Python RAM in **30ms**.

### Stage 2: Hard Filtering (The Bouncer)
> **⚠️ Architectural Warning:** Do not put rules like `WHERE price < 8000000` directly into the Stage 1 vector queries. This causes "Graph Disconnection," breaking the vector index and making queries 10x slower.

Instead, we pull the 1,000 properties into the server's RAM (L1/L2 cache). Python loops through 1,000 objects in **<1 millisecond**:
* If the price is too high: `continue` *(Drop it)*
* If it's a 1BHK and they need a 3BHK: `continue` *(Drop it)*
* If it lacks "A-Khata" in the JSONB tags: `continue` *(Drop it)*

**📊 Funnel Status:** We threw away 800 garbage properties. We have exactly **200 viable, affordable properties** sitting in RAM.

### Stage 3: Ranking & Slotting (The Sniper)
We have 200 perfect houses. Which one is #1? Which is #10? 
We run a **Heuristic Scoring Equation**. Every property gets graded from `0.0` to `1.0` based on:

1. **Vector Similarity (40%):** How perfectly does it match the user's vibe?
2. **Proximity Decay (30%):** Closer is exponentially better.
3. **Quality (20%):** Does it have 5+ photos and a verified badge?
4. **Freshness (10%):** Is it new on the market?

We sort the list descending by this score.

#### 🧬 The Stratified Injection (Slotting)
We don't just show the top 10. We use a psychological slotting strategy:
* We take the **Top 8 "Safe" properties** (Exploitation).
* We take the **Top 2 "Chaos" properties** from Stage 1 (Exploration).
* We surgically inject the Chaos properties at **Index #3** and **Index #7**.

This guarantees the user sees exactly what they asked for, while safely probing their subconscious to see if their tastes are changing.

### 🔄 Stage 4: The Telemetry Loop (Closing the Circuit)
The funnel does not end when the user sees the properties. If we do not capture their reaction, the AI goes blind. Once the Top 10 feed is rendered, the system shifts into **Listening Mode**:

* **The Rejection:** If they immediately scroll past the 2 injected "Chaos" properties, the frontend fires an asynchronous ping.
* **The Interest:** If they stop and look through the photos of a specific property for >5 seconds, the frontend fires a `dwell_time` payload.
* **The Adjustment:** A background worker (RabbitMQ/Redis) catches these payloads and quietly executes the Exponential Moving Average (EMA) math.

**🎯 The Result:** The user's Latent Space coordinate is physically moved in the database. The next time they pull to refresh, Stage 0 starts from a completely new mathematical origin point. The system has successfully learned.

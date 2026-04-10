# Technical Documentation: AI-Powered Smart Listing Enhancement

---

## 1. Overview

This module implements a Computer Vision pipeline using **Gemini 2.5 Flash** to automatically generate real estate marketing content from property photos. It extracts a quality score, descriptive highlights, and actionable marketing strategies.

---

## 2. Core Architecture

- **Model:** gemini-2.5-flash  
- **Input:** Multi-image payload (Maximum 10 images)  
- **Logic:** Zero-shot visual reasoning with a strictly enforced JSON schema output  
- **Reliability:** Implemented Exponential Backoff (via tenacity) to handle 503/429 API rate limits or server spikes  

---

## 3. Data Contract (JSON Schema)

The backend must return this exact structure to ensure the React Native frontend hydrates correctly:

```json
{
  "description": "2-paragraph compelling marketing copy.",
  "quality_score": 85,
  "ai_assistance_overview": {
    "highlights": [
      "Spacious 2-bedroom apartment with panoramic views",
      "Modern kitchen with professional stainless appliances"
    ],
    "strategies": [
      {
        "id": "strat_1",
        "label": "Emphasize location benefits",
        "recommended": true
      },
      {
        "id": "strat_2",
        "label": "Target high-net-worth buyers",
        "recommended": false
      }
    ]
  }
}
```

---

## 4. Implementation Details

- **Prompt Engineering:**  
  The prompt includes a "Muzzle" to prevent the AI from generating overly long labels. Labels are strictly limited to **4–6 words** for UI compatibility.

- **Environment Agnostic:**  
  The code does not use `.env` files in production. It expects `GEMINI_API_KEY` to be injected via the system environment/deployment pipeline.

- **Dependencies:**  
  - `google-genai` (Modern SDK)  
  - `tenacity` (Fault tolerance)  

---

## 5. Frontend Integration Mapping

- **Highlights:**  
  Maps to the **"AI Assistance Overview"** list with blue checkmarks.

- **Strategies:**  
  Maps to the **"Strategies" toggle section**. The `recommended` boolean determines the default state of the switch.

- **Quality Score:**  
  Maps to the **"Impact Analysis" circular progress ring**.

---

## 6. Development Checklist

- [x] Migrated from deprecated `google-generativeai` to `google-genai`  
- [x] Purged `print()` statements and unused `glob` imports  
- [x] Resolved namespace import issues in VS Code  
- [x] Branch created: `feat/ai-smart-listing`  

---

## 7. Architectural Decision: Image-Based vs. Video-Based Inference

For the initial launch of Smart Listing, we have intentionally restricted processing to **Static Images only**.

---

### A. Cost & Token Efficiency

- **Images:**  
  A 10-image payload consumes a fixed, predictable amount of tokens (approx. `$3,000$ to $5,000$` tokens total).

- **Videos:**  
  Gemini processes video at `$1$ frame per second`. A `$60$-second` walk-through video results in `$60$` unique images being analyzed.

  ➝ This increases costs by **~600% per listing** without delivering proportional value.

---

### B. Data Density & Noise

- **Redundancy:**  
  Real estate videos contain large amounts of repeated data (e.g., `$5$ seconds` showing the same kitchen cabinet).

- **Signal Quality:**  
  Images represent **curated highlights**, giving a better signal-to-noise ratio.

- **Latency:**  
  - 10 images → `$3–5$ seconds`  
  - Video → Requires upload + staging + long-running inference  

  ➝ Degrades the **"Instant AI"** experience.

---

### C. Quality Scoring Reliability

The `quality_score` metric depends on:

- Lighting  
- Composition  
- Sharpness  

Video footage often introduces:

- Motion blur  
- Camera shake  

➝ This would **artificially reduce the score**.

Using high-quality static images ensures the AI evaluates the property at its **absolute best representation**.

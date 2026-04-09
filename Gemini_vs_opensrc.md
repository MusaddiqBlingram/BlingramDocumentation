# EXECUTIVE BRIEF: AI Visual Models for Video-to-Text Automation

---

## 1. The Strategic Decision: Renting vs. Owning

To automate our property listing descriptions from user videos, we need a Visual AI model. We have two ways to build this:

**Rent It (Managed API):**  
We pay a vendor like Google (Gemini) per use. They handle the massive server infrastructure and ensure it never crashes under load.

**Own It (Open-Source):**  
We download a free model and host it ourselves on our own cloud servers (like AWS). We pay zero licensing fees to an AI company, but we pay heavily for the specialized servers required to keep it running 24/7.

---

## 2. The Top Open-Source Options (If We "Own It")

If company policy requires us to own the data entirely and host the model internally, the landscape in 2026 is dominated by three main models:

**Qwen2.5-VL:**  
Currently the absolute best all-around open-source visual model. It is incredibly smart, handles complex instructions perfectly, and is built to process sequences of video frames natively.

**InternVL 2.5:**  
A massive, powerful model built for pure accuracy. It closely rivals paid models like GPT-4, but it is heavy and requires incredibly expensive, top-tier servers to run without lagging.

**LLaVA-Video:**  
The industry standard for open-source efficiency. It is slightly less capable than Qwen, but it is highly optimized and much cheaper to host on mid-tier servers.

---

## 3. How Implementation Works (The Operations Reality)

Whether we choose open-source or a paid API, the flow of data looks like this:

**Step 1:**  
The user uploads a video from their phone to our cloud storage.

**Step 2:**  
We cannot feed raw video into most open-source AI. We must build a custom "middle-man" server that:

- Chops the video into a flipbook of images (e.g., 30 pictures)  
- Extracts the spoken audio  

**Step 3:**  
The AI analyzes the flipbook and the audio transcript, then writes the requested property highlights.

---

## 4. The Head-to-Head Comparison: Open-Source vs. Gemini Flash

| Business Factor       | Open-Source (e.g., Qwen2.5-VL) | Managed API (Gemini Flash) |
|----------------------|--------------------------------|----------------------------|
| Upfront Engineering  | High. We must build the video-chopping system and manage complex cloud server deployments. | Low. We send the raw video file directly to Google; they handle the chopping and processing internally. |
| Ongoing Maintenance  | High. We have to monitor server health, update software, and fix server crashes. | Zero. Google handles all maintenance and guarantees uptime. |
| Cost Structure       | High fixed costs. We pay thousands per month to keep specialized AI servers turned on, even if nobody is using the app at 3 AM. | Pay-per-use. We pay fractions of a cent only when a user actually clicks "Enhance Video." |
| Data Privacy         | Absolute. The video data never leaves our internal company servers. | High, but reliant on Google's enterprise privacy agreements. |
| Speed to Market      | Weeks or Months. | Days. |

---

## Conclusion & Recommendation

Building our own open-source infrastructure is a heavy distraction from building our core product.

Unless we have strict legal reasons to keep video data on our own servers, the standard industry move is to start with **Gemini Flash**.

It allows us to:

- Launch immediately  
- Scale automatically during traffic spikes  
- Keep costs tied directly to user activity  

Rather than paying flat, high server rental fees regardless of usage.

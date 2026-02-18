# 🚫 Content Moderation Policy

## 🚫 INSTANT BLOCKS (Zero Tolerance)

If AWS detects **any** of the following with **>60% confidence**, the content is **instantly blocked**:

- Gore  
- Viscera  
- Graphic violence  
- Sexual violence  
- Hate symbols  
- Nazi  
- Drugs (Category)  
- Drug products  
- Pills  
- Explicit nudity  
  - *Note: Ignored if AWS also tags it as **Non-Explicit***  

---

## 🧩 CONDITIONAL BLOCKS (Context & Logic Based)

These items are **not blocked on their own**, but are blocked **only if specific conditions are met**.

---

### 1️⃣ Child Safety Risk  
**Blocked if `Average Age < 18`**

If **any** of the following are detected, the system performs **face analysis**.  
If a **minor is detected**, the content is **blocked**.

- Suggestive content  
- Partial nudity  
- Revealing clothes  
- Swimwear  
- Underwear  
- Sexual activity  
- Kissing  
- Non-explicit nudity  

---

### 2️⃣ Animal Abuse  
**Blocked if combined with violence**

If AWS detects **an animal AND violence in the same frame**, the content is blocked.

**Animals:**
- Animal  
- Dog  
- Cat  
- Horse  
- Bird  
- Pet  

**Combined with:**
- Violence  
- Physical violence  
- Graphic violence  

---

### 3️⃣ Drug Paraphernalia  
**Blocked via background context scan**

If AWS detects **any of the following objects**, the content is blocked:

- Syringe  
- Needle  
- Bong  
- Hookah  
- Glass pipe  
- Crack pipe  

---

## ⚠️ HUMAN REVIEW (Flagged, Not Blocked)

If the following are detected **and no higher-severity violations exist**, the content **passes** but is **flagged for moderation review**:

- Tobacco  
- Smoking  
- Alcohol  
- Drinking  

---

## ✅ Summary

| Category | Action |
|--------|--------|
| Instant Block | Immediately blocked |
| Conditional Block | Blocked based on logic & context |
| Human Review | Allowed but flagged |


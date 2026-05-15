# ICP Scoring Model
### Healthcare GTM Engine — Composite Lead Scoring (0–100)

> The scoring model is implemented as a Code node in n8n workflow `02-lead-scoring-routing.json`. This document explains the design rationale and calibration behind each dimension.

---

## Score Summary

| Dimension | Max Score | Weight % | Why It Matters |
|---|---|---|---|
| Pain Signal Strength | 25 | 25% | Active pain = active buyer |
| Intent Recency | 20 | 20% | Timing is everything in outbound |
| Organization Size | 20 | 20% | 10–200 providers = ideal ICP |
| Specialty Match | 15 | 15% | Prior-auth burden varies by specialty |
| EHR Vendor | 10 | 10% | Legacy EHR = integration friction = pain |
| Geographic Fit | 10 | 10% | Healthcare density = cluster targeting |
| **TOTAL** | **100** | **100%** | |

---

## Routing Tiers

| Tier | Score Range | Action | Rationale |
|---|---|---|---|
| 🔴 HOT | ≥ 80 | Priority sequence + Slack alert | Immediate outreach, <24h |
| 🟡 WARM | 50–79 | HubSpot 14-day nurture | Good fit, build awareness |
| 🟠 COOL | 20–49 | Monthly drip | Keep warm, watch for new signals |
| ⚪ COLD | < 20 | Archive | Don't waste sequence slots |

---

## Dimension Deep-Dives

### 1. Pain Signal Strength (25 pts max)

The most heavily weighted dimension. A lead with active pain now is worth more than a perfect-fit lead with no signal.

```javascript
let painScore = 0;
const signals = lead.pain_signals || [];

if (signals.includes('job_posting_admin'))  painScore += 8;  // Hiring admin = drowning in manual work
if (signals.includes('negative_reviews'))   painScore += 6;  // Public complaints = visible pain
if (signals.includes('high_denial_rate'))   painScore += 7;  // Denial rate data = quantified problem
if (signals.includes('staff_turnover'))     painScore += 4;  // Turnover signal = process problems

breakdown.pain = Math.min(painScore, 25);
```

**Why job postings score highest:** A practice posting for a Prior Authorization Coordinator is actively spending $50–70K/year to hire someone to do what automation can do. They've acknowledged the problem and allocated budget — they just haven't found the right solution yet.

---

### 2. Intent Recency (20 pts max)

Signal timing is the single biggest lever in outbound. An org that expanded 6 months ago has probably already solved their problem. An org that posted an admin job 4 days ago is actively feeling it.

```javascript
const daysSinceSignal = parseInt(lead.days_since_signal) || 999;

if (daysSinceSignal <= 7)   breakdown.recency = 20;  // This week — hot window
if (daysSinceSignal <= 30)  breakdown.recency = 12;  // This month — still warm
if (daysSinceSignal <= 90)  breakdown.recency = 5;   // This quarter — cooling
else                         breakdown.recency = 0;   // Stale signal
```

> **The 7-day window:** Research on outbound timing consistently shows that 60–70% of deals close within 30 days of the triggering event. Catching an org within the first week of a buying signal dramatically increases reply rates.

---

### 3. Organization Size (20 pts max)

Not all provider counts are equal. The ICP sweet spot for healthcare admin automation:

```javascript
const providers = parseInt(lead.num_providers) || 0;

if (providers >= 10 && providers <= 200) breakdown.org_size = 20;  // Sweet spot
if (providers >= 5  && providers < 10)  breakdown.org_size = 12;  // Small but real
if (providers > 200)                     breakdown.org_size = 15;  // Large, but longer sales cycle
else                                     breakdown.org_size = 5;
```

**Why 10–200 is ideal:**
- **Too small (<5 providers):** Not enough admin volume to justify; often one person wears all hats
- **Sweet spot (10–200):** Enough admin complexity to feel the pain, fast enough decision-making to close quickly
- **Enterprise (200+):** Real budget, but 12–18 month sales cycles, procurement committees, and IT gatekeeping

---

### 4. Specialty Match (15 pts max)

Prior authorization burden varies dramatically by specialty. High-burden specialties = highest pain = most motivated buyers.

```javascript
const highAdminSpecialties = ['radiology', 'cardiology', 'orthopedics', 'oncology', 'gastroenterology'];

if (highAdminSpecialties.some(s => specialty.includes(s)))        breakdown.specialty = 15;
else if (specialty.includes('primary') || specialty.includes('family')) breakdown.specialty = 10;
else if (specialty.includes('urgent') || specialty.includes('multi'))   breakdown.specialty = 12;
else                                                                      breakdown.specialty = 6;
```

**Specialty prior-auth burden ranking (MGMA data):**
1. Radiology — nearly every procedure requires auth
2. Cardiology — high-cost procedures, frequent denials
3. Orthopedics — elective surgeries, payer scrutiny
4. Oncology — complex protocols, constant reauthorization
5. Gastroenterology — high procedure volume, scope approvals

---

### 5. EHR Vendor (10 pts max)

EHR choice reveals both technical environment and likely pain level. Legacy systems = more manual workarounds = more admin burden.

```javascript
if (ehr.includes('epic') || ehr.includes('cerner'))                   breakdown.ehr = 10;
else if (ehr.includes('athena') || ehr.includes('eclinical'))         breakdown.ehr = 8;
else if (ehr.includes('allscripts') || ehr.includes('meditech'))      breakdown.ehr = 7;
else                                                                    breakdown.ehr = 4;
```

**EHR Vendor Signal Interpretation:**
- **Epic/Cerner:** Large enterprise installation, often has API access, but admin workflows built around legacy processes. Integration-ready.
- **athenahealth/eClinical:** Mid-market, more modern, but still high auth volume
- **Allscripts/Meditech:** Older, harder to integrate, more manual gap-filling
- **Unknown/Other:** Either very small or using niche software — needs more research

---

### 6. Geographic Fit (10 pts max)

Focus on high-density states first for cluster efficiency — more accounts = more referenceability = faster trust building.

```javascript
const highDensityStates = ['CA', 'TX', 'FL', 'NY', 'PA', 'IL', 'OH', 'GA', 'NC', 'MI'];
breakdown.geo = highDensityStates.includes(state) ? 10 : 5;
```

**Why these states:** Top 10 US states by number of physician practices (AMA data). Cluster targeting in a single state allows for case study leverage: "We work with 3 other orthopedic groups in Texas, here's what they saw."

---

## Model Calibration Notes

This model was calibrated for **Interactly.AI's specific GTM motion** at the pre-seed stage:
- Small sales team (1–2 founders doing outbound)
- Limited Clay enrichment credits (need signal filtering)
- Outcome-based pricing model (ROI math matters from Day 1)
- US healthcare focus (HIPAA compliance as default barrier)

**Recalibrate when:**
- Moving upmarket to 200+ provider health systems
- Expanding to insurance/payer side (different ICP entirely)
- After 90 days of data — tune weights based on what actually converts

---

## Full Code Implementation

The complete scoring function lives in the `ICP Scoring Engine` Code node inside:
[`workflows/n8n/02-lead-scoring-routing.json`](../workflows/n8n/02-lead-scoring-routing.json)

Look for node ID: `score-lead`

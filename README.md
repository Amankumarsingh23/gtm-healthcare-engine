# 🏥 GTM Automation Engine — Healthcare Edition
### Built for Interactly.AI | by Aman kumar singh, IIT Kanpur

> **An end-to-end GTM automation pipeline that detects buying signals from US healthcare orgs, enriches them via Clay waterfall, scores them with a weighted ICP model, and routes them into personalized multi-channel outbound sequences — fully automated, pre-seed budget ($435–700/mo).**

---

## 📐 System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     GTM AUTOMATION ENGINE — LAYER MAP                  │
├──────────────┬──────────────┬──────────────┬───────────────────────────┤
│  LAYER 1     │  LAYER 2     │  LAYER 3     │  LAYER 4                  │
│  Signal      │  Enrichment  │  Scoring     │  Personalization          │
│  Detection   │  (Clay)      │  & Routing   │  (Claude API)             │
│              │              │              │                           │
│  Google News │  Firmograph- │  ICP Model   │  Prompt Chain:            │
│  RSS Feeds   │  ics         │  (0–100)     │  Pain → Angle →           │
│  (3 streams) │  Technograph-│              │  Subject → Body →         │
│              │  ics         │  HOT ≥ 80    │  CTA                      │
│  Job Boards  │  Contact     │  WARM 50–79  │                           │
│  LinkedIn    │  Mapping     │  COOL 20–49  │  Healthcare-              │
│  Bombora     │  Pain Indic- │  COLD < 20   │  specific tone            │
│              │  ators       │              │  & triggers               │
└──────┬───────┴──────┬───────┴──────┬───────┴───────────┬───────────────┘
       │              │              │                   │
       ▼              ▼              ▼                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  LAYER 5: ORCHESTRATION (n8n)                                           │
│  Webhook triggers → Code nodes → Switch routing → HTTP integrations     │
└──────────────┬──────────────────────────────────────────────────────────┘
               │
       ┌───────┴──────────────────────┐
       ▼                              ▼
┌─────────────────┐          ┌────────────────────┐
│  LAYER 6:       │          │  LAYER 7:           │
│  OUTBOUND       │          │  ATTRIBUTION        │
│  EXECUTION      │          │  & FEEDBACK         │
│                 │          │                     │
│  Smartlead      │          │  Google Sheets      │
│  (email seq.)   │          │  Signal → Reply     │
│  Slack alerts   │          │  Tracking           │
│  HubSpot CRM    │          │  HubSpot pipeline   │
│  LinkedIn DMs   │          │  Win/loss analysis  │
└─────────────────┘          └────────────────────┘
```

---

## 🔄 Workflows

### Workflow 1 — Signal Detection Pipeline
**File:** [`workflows/n8n/01-signal-detection-pipeline.json`](workflows/n8n/01-signal-detection-pipeline.json)

Monitors 3 parallel RSS streams every 6 hours:
- **Pain signals** — `"prior authorization" OR "revenue cycle" hiring`
- **Expansion signals** — `"new clinic opening" OR "medical group expansion"`
- **EHR migration signals** — `"Epic implementation" OR "Cerner contract"`

Then: deduplicates → classifies → strength-filters → pushes to Clay → logs to Sheets → Slack alert.

```
Schedule (6h) ──┬──► Google News RSS [Pain]      ─┐
                ├──► Google News RSS [Expansion]  ─┼──► Parse & Classify
                └──► Google News RSS [EHR]        ─┘         │
                                                              ▼
                                                   Filter: Medium+ only
                                                         │
                                          ┌──────────────┼──────────────┐
                                          ▼              ▼              ▼
                                     Clay Enrich    Google Sheets   Slack Alert
```

**Import:** Copy JSON → n8n → Import Workflow → Set env vars (see [Setup Guide](docs/setup-guide.md))

---

### Workflow 2 — Lead Scoring & Smart Routing
**File:** [`workflows/n8n/02-lead-scoring-routing.json`](workflows/n8n/02-lead-scoring-routing.json)

Triggered by Clay webhook after enrichment completes. Runs composite ICP scoring and routes to the right action.

```
Clay Webhook ──► ICP Scoring Engine ──► Route by Tier
                      │                      │
                      │            ┌─────────┼──────────┬──────────┐
                      │            ▼         ▼          ▼          ▼
                      │          HOT        WARM       COOL       COLD
                      │        (≥80)      (50-79)    (20-49)    (<20)
                      │          │         │          │          │
                      │       Smartlead  HubSpot   G.Sheets  Archive
                      │       Slack DM  Nurture    Monthly
                      │
                      └──► Webhook Response (score + tier)
```

---

## 🧮 ICP Scoring Model

**Total: 100 points** | Routing thresholds: HOT ≥ 80 | WARM 50–79 | COOL 20–49 | COLD < 20

| Dimension | Max Points | Rationale |
|---|---|---|
| **Pain Signal Strength** | 25 | Admin job postings, denial rate complaints, staff turnover = active pain |
| **Intent Recency** | 20 | Signal <7 days = 20pts, <30 days = 12pts, <90 days = 5pts |
| **Organization Size** | 20 | 10–200 providers = ideal ICP (max admin burden, fast decision) |
| **Specialty Match** | 15 | Radiology/Cardiology/Ortho = highest prior-auth burden |
| **EHR Vendor** | 10 | Epic/Cerner = legacy pain, integration-ready |
| **Geographic Fit** | 10 | Top 10 US healthcare density states |

> **Design decision:** Pain + Recency = 45 points (nearly half the score). An org hiring admin staff right now is worth more than a perfect-fit org showing no buying signal. Recency is a forcing function for timing.

Full model code: [`scoring/icp-model.md`](scoring/icp-model.md)

---

## 🤖 Prompt Engineering — Claude API Personalization

**File:** [`prompts/personalization-prompt.md`](prompts/personalization-prompt.md)

Each outbound email is generated by Claude using a 5-stage chain:

```
Stage 1: PAIN IDENTIFICATION
  Input: signal_type + specialty + ehr_vendor
  Output: primary pain hypothesis + secondary angle

Stage 2: OPENING HOOK
  Input: pain hypothesis + headline (from signal)
  Output: 1-sentence opener referencing their specific situation

Stage 3: SUBJECT LINE
  Input: opener + org_name + signal_type
  Output: 3 subject line variants (curiosity / direct / benchmark)

Stage 4: EMAIL BODY
  Input: all above + num_providers + contact_role
  Output: 4-sentence body (hook → problem → proof → CTA)

Stage 5: FOLLOW-UP SEQUENCE
  Input: email body + contact_role
  Output: Day 3 bump + Day 7 value add + Day 14 breakup
```

See the full prompt chain with examples → [`prompts/personalization-prompt.md`](prompts/personalization-prompt.md)

---

## 📁 Repository Structure

```
gtm-healthcare-engine/
│
├── workflows/n8n/
│   ├── 01-signal-detection-pipeline.json   # RSS monitoring + Clay trigger
│   └── 02-lead-scoring-routing.json        # ICP scoring + HubSpot/Smartlead routing
│
├── prompts/
│   ├── personalization-prompt.md           # 5-stage Claude API prompt chain
│   └── objection-handler-prompt.md         # Real-time objection deflection prompts
│
├── scoring/
│   └── icp-model.md                        # Full scoring model with rationale
│
├── docs/
│   ├── architecture.md                     # Full system architecture breakdown
│   └── setup-guide.md                      # Step-by-step env var + import guide
│
└── assets/
    └── architecture-diagram.svg            # Visual system map
```

---

## 🛠️ Tech Stack & Monthly Cost

| Tool | Purpose | Cost/mo |
|---|---|---|
| **n8n** (self-hosted) | Workflow orchestration | $0 (self-hosted) or $20 cloud |
| **Clay** | Waterfall enrichment | $149 (Starter) |
| **Smartlead** | Email sequencing | $59 |
| **Claude API** | Email personalization | ~$30–50 (usage) |
| **HubSpot** | CRM | $0 (free tier) |
| **Google Sheets** | Signal logging | $0 |
| **Slack** | Alerts | $0 |
| **Total** | | **~$238–278/mo** |

> Scales to $435–700/mo as volume grows (Clay higher tier, Smartlead seats).

---

## ⚙️ Quick Setup

1. **Import n8n workflows** — drag JSONs from `workflows/n8n/` into n8n
2. **Set environment variables** — see [`docs/setup-guide.md`](docs/setup-guide.md)
3. **Configure Clay table** — create a webhook source, paste `CLAY_WEBHOOK_ID` into n8n env
4. **Connect Smartlead campaigns** — create HOT and WARM campaigns, paste IDs into env
5. **Activate Schedule Trigger** — workflow runs every 6h automatically

Full setup guide with screenshots → [`docs/setup-guide.md`](docs/setup-guide.md)

---

## 📊 Expected KPIs (12-Week Targets)

| Metric | Week 4 | Week 8 | Week 12 |
|---|---|---|---|
| Signals/week | 50+ | 100+ | 150+ |
| HOT leads/week | 5–8 | 10–15 | 15–25 |
| Email open rate | 35% | 42% | 48% |
| Reply rate | 8% | 12% | 15% |
| Demos booked/mo | 2–3 | 5–8 | 10–15 |

---

## 👤 About

**Aman kumar singh** | IIT Kanpur | GTM Automation & AI Engineer candidate @ Interactly.AI

Built for the US healthcare GTM motion — understanding that the real unlock isn't more emails, it's **signal timing × ICP precision × personalization at scale**.

> *The best outbound email is one that arrives the same week the prospect realizes they have a problem.*

---

*This repository represents my submission for the GTM Automation & AI Engineer role at Interactly.AI.*

# Setup Guide
### GTM Healthcare Engine — Environment Variables & Import Instructions

---

## Prerequisites

| Tool | Account Needed | Cost |
|---|---|---|
| n8n | Self-hosted (Docker) or n8n.cloud | Free / $20/mo |
| Clay | clay.com account | $149/mo Starter |
| Smartlead | smartlead.ai account | $59/mo |
| HubSpot | Free CRM account | $0 |
| Google Sheets | Google account | $0 |
| Slack | Workspace | $0 |
| Anthropic | api.anthropic.com | Pay-as-you-go |

---

## Step 1: Set Environment Variables in n8n

Go to **n8n Settings → Variables** and create:

```env
# Clay Integration
CLAY_API_KEY=clay_sk_...
CLAY_WEBHOOK_ID=your-clay-webhook-id

# Smartlead
SMARTLEAD_API_KEY=sl_...
SMARTLEAD_HOT_CAMPAIGN=campaign-id-for-hot-leads
SMARTLEAD_WARM_CAMPAIGN=campaign-id-for-warm-leads

# HubSpot
HUBSPOT_API_KEY=pat-na1-...

# Slack
SLACK_SIGNALS_CHANNEL=C0XXXXXXXXX   # Channel ID, not name
SLACK_HOT_LEADS=C0YYYYYYYYY

# Google Sheets
GSHEET_SIGNAL_LOG=1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms  # Sheet ID from URL

# Anthropic (for personalization stage)
ANTHROPIC_API_KEY=sk-ant-...
```

---

## Step 2: Import n8n Workflows

### Workflow 1 — Signal Detection Pipeline
1. Open n8n → **Workflows → New**
2. Click **⋮ menu → Import from file**
3. Select `workflows/n8n/01-signal-detection-pipeline.json`
4. Review nodes — all env vars should auto-resolve
5. **Activate** the workflow (toggle top right)

### Workflow 2 — Lead Scoring & Routing
1. Open n8n → **Workflows → New**
2. Import `workflows/n8n/02-lead-scoring-routing.json`
3. Copy the **Webhook URL** from the `Webhook: Clay Enrichment Complete` node
4. Paste this URL into your Clay table as the "Send to webhook" action
5. **Activate** the workflow

---

## Step 3: Configure Clay Table

1. Create a new Clay table: **"Healthcare Signal Inbound"**
2. Add webhook source: **Sources → Webhook → Create**
3. Copy webhook URL → paste into n8n env as `CLAY_WEBHOOK_ID` (extract the ID from the URL)
4. Add enrichment columns:
   - **Firmographics:** Clearbit Company, Apollo.io
   - **Technographics:** BuiltWith (EHR detection)
   - **Contacts:** Apollo, Hunter, LinkedIn
   - **Pain signals:** Custom formula column (see below)

**Pain Signal Formula (Clay):**
```
IF(CONTAINS(job_postings_text, "prior auth"), "job_posting_admin", "")
+ IF(google_reviews_avg < 3.5, "negative_reviews", "")
+ IF(staff_turnover_signal = TRUE, "staff_turnover", "")
```

5. Add action: **"Send to Webhook"** → your n8n Workflow 2 webhook URL

---

## Step 4: Configure Smartlead Campaigns

### HOT Campaign (Priority Sequence)
- **Name:** `Healthcare GTM — HOT`
- **Sending schedule:** Mon–Fri, 7am–6pm local
- **Daily limit:** 50 emails
- **Sequence:** 5 steps (Day 1, 3, 7, 14, 21)
- **Subject line variable:** `{{custom_subject}}` (populated from Claude API)

### WARM Campaign (Nurture)
- **Name:** `Healthcare GTM — WARM`
- **Sending schedule:** Mon–Fri, 8am–5pm local
- **Daily limit:** 100 emails
- **Sequence:** 3 steps (Day 1, 7, 14)

After creating both campaigns:
1. Copy each Campaign ID from the URL (`smartlead.ai/campaigns/{ID}`)
2. Set as `SMARTLEAD_HOT_CAMPAIGN` and `SMARTLEAD_WARM_CAMPAIGN` env vars

---

## Step 5: Google Sheets Setup

Create a Google Sheet with two tabs:

**Tab 1: "Signals"** — columns:
```
signal_id | org_name | signal_type | signal_strength | headline | source_url | detected_at | published_at | status
```

**Tab 2: "Scored Leads"** — columns:
```
org_name | contact_email | icp_score | icp_tier | icp_action | signal_type | specialty | num_providers | ehr_vendor | score_breakdown | scored_at
```

Copy the Sheet ID from the URL:
`docs.google.com/spreadsheets/d/{SHEET_ID}/edit`

Set as `GSHEET_SIGNAL_LOG` env var.

---

## Step 6: Slack Configuration

1. Create two channels: `#gtm-signals` and `#hot-leads`
2. Go to **api.slack.com/apps → Create New App**
3. Add **Bot Token Scopes:** `chat:write`, `channels:read`
4. Install app to workspace
5. Copy Bot Token → use in n8n Slack credential
6. Get Channel IDs: right-click channel → **Copy → Copy Channel ID**
7. Set as `SLACK_SIGNALS_CHANNEL` and `SLACK_HOT_LEADS` env vars

---

## Testing the Full Pipeline

### Manual Test (Workflow 1):
1. Open Signal Detection workflow
2. Click **"Execute Workflow"** (runs once manually)
3. Check: Google Sheets "Signals" tab should have new rows
4. Check: Slack `#gtm-signals` should have alert messages
5. Check: Clay table should have new records

### Manual Test (Workflow 2):
1. Copy the Webhook URL from n8n
2. Use this curl to simulate a Clay payload:

```bash
curl -X POST "YOUR_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{
    "org_name": "Summit Orthopedics",
    "contact_email": "test@summitortho.com",
    "contact_first_name": "Jennifer",
    "contact_last_name": "Walsh",
    "contact_role": "Practice Manager",
    "specialty": "orthopedics",
    "num_providers": "45",
    "ehr_vendor": "epic",
    "state": "TX",
    "pain_signals": ["job_posting_admin", "negative_reviews"],
    "days_since_signal": "5",
    "signal_type": "pain_signal",
    "headline": "Summit Orthopedics hiring Prior Authorization Coordinator"
  }'
```

Expected response:
```json
{ "status": "processed", "score": 85, "tier": "HOT" }
```

---

## Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| Clay webhook not firing | Wrong webhook URL in Clay | Re-copy URL from n8n webhook node |
| Smartlead leads not enrolling | Campaign ID wrong or campaign paused | Check campaign status in Smartlead |
| Slack not posting | Bot not in channel | Invite bot: `/invite @YourBotName` |
| Scoring returns 0 | Lead missing fields | Check Clay enrichment ran successfully |
| Google Sheets error | Sheet ID wrong or no access | Re-share sheet with service account email |

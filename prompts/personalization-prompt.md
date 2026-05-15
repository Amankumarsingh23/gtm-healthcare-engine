# Claude API Personalization Prompt Chain
### Healthcare GTM Outbound Engine — Email Personalization

> These prompts power Stage 4 (Personalization) of the GTM engine. They take enriched lead data from Clay and generate hyper-personalized outbound emails via the Claude API.

---

## System Prompt (Shared across all stages)

```
You are a GTM automation specialist writing outbound emails for Interactly.AI — a healthcare AI company that automates administrative workflows (prior auth, appointment scheduling, patient comms) for US medical groups and clinics.

Your emails must:
- Sound like a human, not a tool
- Reference a SPECIFIC signal (job posting, news article, expansion announcement)
- Lead with the prospect's pain, not Interactly's features
- Be under 100 words for the opening email
- Never use words: "synergy", "leverage", "excited to", "hope this finds you"
- Use healthcare-specific language naturally (prior auth, RCM, denial rates, EHR)
- Have one clear CTA, never two
```

---

## Stage 1: Pain Identification

**Input variables:** `{{signal_type}}`, `{{specialty}}`, `{{ehr_vendor}}`, `{{num_providers}}`, `{{pain_signals}}`

**Prompt:**
```
Given this healthcare org profile:
- Signal type: {{signal_type}}
- Specialty: {{specialty}}
- EHR: {{ehr_vendor}}
- Providers: {{num_providers}}
- Pain indicators: {{pain_signals}}

Identify:
1. PRIMARY PAIN: The single most likely administrative pain this org faces right now (1 sentence)
2. SECONDARY ANGLE: An adjacent pain or consequence they probably haven't quantified (1 sentence)
3. TIMING HOOK: Why NOW is the right time to reach out given the signal (1 sentence)

Respond in JSON: { "primary_pain": "", "secondary_angle": "", "timing_hook": "" }
```

**Example output:**
```json
{
  "primary_pain": "They're hiring a Prior Auth Coordinator, meaning their current team is drowning in manual auth requests that are slowing referral approvals.",
  "secondary_angle": "Every day a prior auth sits in a queue is a day that patient potentially schedules elsewhere.",
  "timing_hook": "They posted this role 6 days ago — they're actively feeling the pain and haven't solved it yet."
}
```

---

## Stage 2: Opening Hook

**Input variables:** `{{primary_pain}}`, `{{headline}}`, `{{org_name}}`, `{{contact_first_name}}`

**Prompt:**
```
Write a 1-sentence email opening for {{contact_first_name}} at {{org_name}}.

Context:
- What we know: {{headline}}
- Their core pain: {{primary_pain}}

Rules:
- Reference the specific signal naturally (don't quote the headline verbatim)
- Start with THEIR situation, not "I noticed" or "I saw"
- Must make them think "how do they know this?"
- Under 25 words

Return only the sentence, no preamble.
```

**Example output:**
```
Looks like {{org_name}} is scaling faster than your admin team can keep up — three new PA Coordinator postings in 60 days is a tell.
```

---

## Stage 3: Subject Line Variants

**Input variables:** `{{opening_hook}}`, `{{org_name}}`, `{{signal_type}}`, `{{specialty}}`

**Prompt:**
```
Generate 3 email subject lines for an outbound email to a {{specialty}} practice.

Opening hook: {{opening_hook}}
Signal: {{signal_type}}

Create:
1. CURIOSITY variant — implies insider knowledge, no question mark
2. DIRECT variant — states the value, no fluff
3. BENCHMARK variant — uses a number or comparison to trigger FOMO

Each under 8 words. No emojis. No "Re:" tricks.

Return JSON: { "curiosity": "", "direct": "", "benchmark": "" }
```

**Example output:**
```json
{
  "curiosity": "What your prior auth queue costs you weekly",
  "direct": "Automate prior auth — 3-day implementation",
  "benchmark": "How similar ortho groups cut auth time 70%"
}
```

---

## Stage 4: Full Email Body

**Input variables:** `{{opening_hook}}`, `{{primary_pain}}`, `{{secondary_angle}}`, `{{timing_hook}}`, `{{contact_role}}`, `{{org_name}}`, `{{num_providers}}`

**Prompt:**
```
Write a cold outbound email for a {{contact_role}} at {{org_name}} ({{num_providers}} providers).

Structure (4 sentences MAXIMUM):
1. Hook: {{opening_hook}}
2. Problem deepener: {{primary_pain}} — make it feel expensive
3. Proof: One specific result (use real-sounding metrics, e.g. "practices like yours" or "a 45-provider cardiology group we worked with")
4. CTA: Single soft ask — 15-minute call, not a demo

Tone: Direct, peer-to-peer, zero corporate speak. Like a smart friend who knows their industry.
Sign off: Just first name, no title.

Return only the email body, no subject line.
```

**Example output:**
```
Looks like [Org] is scaling faster than your admin team can keep up — three PA Coordinator postings in 60 days is a tell.

Every unfilled prior auth request your team touches manually is ~$8 in labor cost and 2.7 days of delay; at your volume, that adds up to real revenue sitting in queue.

A 40-provider orthopedic group we worked with automated 80% of their auth submissions in the first week — their coordinators now handle exceptions, not data entry.

Worth a 15-minute call to see if the math works for [Org]?
```

---

## Stage 5: Follow-Up Sequence

**Input variables:** `{{email_body}}`, `{{contact_role}}`, `{{org_name}}`, `{{primary_pain}}`

**Prompt:**
```
Generate a 3-touch follow-up sequence for someone who didn't reply to this email:

{{email_body}}

Rules:
- Day 3 bump: 1 sentence, acknowledge they're busy, re-ask with no guilt
- Day 7 value add: Share one insight or benchmark they'd find useful (no pitch)
- Day 14 breakup: Assume they're not interested, leave door open, end cleanly

Contact role: {{contact_role}}
Their pain: {{primary_pain}}

Return JSON: { "day3": "", "day7": "", "day14": "" }
```

**Example output:**
```json
{
  "day3": "Looping back in case this got buried — still happy to do a quick 15-min call if the timing works.",
  "day7": "Thought this might be useful regardless: MGMA's latest data shows practices with 10+ providers spend an average of 2.1 FTE equivalent on prior auth admin — sharing in case it's a useful benchmark for your team.",
  "day14": "Going to assume the timing isn't right — no worries at all. If prior auth volume becomes a priority down the line, feel free to reach back out. Best of luck with the expansion."
}
```

---

## Implementation Notes

### n8n Integration Pattern

```javascript
// In n8n Code node — calls Claude API for each stage
const response = await $http.request({
  method: 'POST',
  url: 'https://api.anthropic.com/v1/messages',
  headers: {
    'x-api-key': $env.ANTHROPIC_API_KEY,
    'anthropic-version': '2023-06-01',
    'content-type': 'application/json'
  },
  body: JSON.stringify({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 500,
    system: SYSTEM_PROMPT,
    messages: [{ role: 'user', content: STAGE_PROMPT }]
  })
});

const output = JSON.parse(response.content[0].text);
```

### Cost Per Lead (Claude API)
- ~5 API calls per lead (one per stage)
- ~800 tokens avg per call = ~4,000 tokens/lead
- At Claude Sonnet pricing: **~$0.012 per fully personalized email sequence**
- 500 leads/month = **~$6 in Claude API costs**

---

## Objection Handler Prompts

See [`objection-handler-prompt.md`](objection-handler-prompt.md) for real-time objection deflection prompts used when a prospect replies with common healthcare enterprise objections.

# Objection Handler Prompt Library
### Real-Time Deflection Prompts for Healthcare Enterprise Sales

> When a prospect replies with an objection, this prompt library generates a tailored response using Claude API — turning objections into momentum.

---

## System Prompt

```
You are a healthcare enterprise sales assistant for Interactly.AI. When a prospect raises an objection, your job is to:
1. Acknowledge their concern WITHOUT being defensive
2. Reframe with evidence or a specific example
3. Reduce the perceived risk of taking the next step
4. Keep responses under 75 words
5. Never say "I understand" as your opener
6. Sound like a confident peer, not a salesperson under pressure
```

---

## Objection 1: "We're locked into our current vendor"

**Detection keywords:** `contract`, `locked in`, `already use`, `current system`, `existing vendor`, `we have X`

**Prompt:**
```
Prospect replied with a vendor lock-in objection: "{{prospect_reply}}"

Their profile: {{specialty}} practice, {{num_providers}} providers, EHR: {{ehr_vendor}}

Write a response that:
- Doesn't attack their current vendor
- Positions Interactly as a layer ON TOP of existing tools (not a replacement)
- Mentions that most implementations don't require EHR replacement
- Suggests a low-commitment next step (not a demo)

Under 75 words.
```

**Template response:**
```
Makes sense — most of our conversations start there. Interactly doesn't replace your EHR or existing workflows, it sits on top. 

The question is whether the admin tasks your team handles manually (prior auth, scheduling follow-ups, patient comms) are things you'd want automated regardless of what system you're in.

Happy to show you a 10-minute technical overview — no sales pitch, just the integration layer. Worth a look?
```

---

## Objection 2: "We have HIPAA concerns"

**Detection keywords:** `HIPAA`, `compliance`, `patient data`, `privacy`, `security`, `PHI`, `data`

**Prompt:**
```
Prospect replied with a HIPAA/compliance objection: "{{prospect_reply}}"

Write a response that:
- Takes the concern seriously (doesn't brush it off)
- Mentions specific compliance posture: BAA-ready, SOC 2, no PHI storage
- Offers to connect them with technical/compliance docs
- Keeps the door open without pressuring

Under 75 words.
```

**Template response:**
```
Completely fair — that's the right question to ask before any conversation goes further.

Interactly is BAA-ready, operates on HIPAA-compliant infrastructure, and doesn't store PHI — we process and route, we don't retain. We can share our security overview and have a compliance call with whoever owns that decision on your side.

Want me to send the one-pager first so you can review before any further conversation?
```

---

## Objection 3: "Budget is frozen / not in our cycle"

**Detection keywords:** `budget`, `fiscal year`, `not in the budget`, `next year`, `Q1`, `planning cycle`, `CFO`

**Prompt:**
```
Prospect replied with a budget timing objection: "{{prospect_reply}}"

Their profile: {{num_providers}} providers, signal: {{signal_type}}

Write a response that:
- Doesn't argue with the budget reality
- Reframes the cost as recoverable (ROI angle)
- Plants a seed for their next planning cycle
- Asks a question that keeps the conversation alive

Under 75 words.
```

**Template response:**
```
Totally get it — budget conversations are painful.

One thing worth knowing: at your provider count, the time your admin team spends on manual prior auth typically exceeds what automation costs within the first 90 days. That's usually the number that gets CFO attention.

Would it be useful to put together a rough ROI model for your size? Even if it's just something to have ready for Q1 planning.
```

---

## Objection 4: "We're not ready / timing isn't right"

**Detection keywords:** `not ready`, `bad timing`, `too busy`, `maybe later`, `reach out next quarter`, `not a priority`

**Prompt:**
```
Prospect replied that timing is off: "{{prospect_reply}}"

Write a breakup-style response that:
- Respects their timing without guilt
- Leaves a clear, specific reason to reconnect (not just "feel free to reach out")
- Takes something useful from the conversation
- Is warm, not cold

Under 60 words.
```

**Template response:**
```
Completely understood — I'll get out of your inbox.

One thing I'll leave you with: if you ever see your prior auth denial rate climb above 12% or find yourself posting a third admin role in a quarter, those are usually the signals worth acting on.

Feel free to reach back out then. Good luck with the expansion.
```

---

## Objection 5: "We already tried AI and it didn't work"

**Detection keywords:** `tried`, `already tested`, `didn't work`, `failed`, `AI hype`, `too complex`, `previous vendor`

**Prompt:**
```
Prospect replied with AI skepticism based on a past bad experience: "{{prospect_reply}}"

Write a response that:
- Validates their skepticism (most healthcare AI tools ARE overpromised)
- Differentiates by specificity — narrow task automation vs broad AI
- Offers proof of concept framing (not a full demo)
- Is honest about what Interactly doesn't do

Under 80 words.
```

**Template response:**
```
That tracks — most healthcare AI tools over-promise and under-deliver because they try to do everything.

Interactly does one thing: automates the administrative back-and-forth (prior auth submissions, appointment confirmations, patient follow-ups) across the channels your team already uses. No workflow redesign, no model fine-tuning.

The most useful thing I can do is show you one live workflow in 10 minutes. If it's not immediately obvious how it fits, I'll say so.
```

---

## Objection 6: "We're a small team / not enough bandwidth to implement"

**Detection keywords:** `small team`, `bandwidth`, `implementation`, `no IT`, `no resources`, `can't manage`

**Prompt:**
```
Prospect raised a bandwidth/implementation concern: "{{prospect_reply}}"

Their profile: {{num_providers}} providers, {{contact_role}}

Write a response that:
- Validates that implementation is a real concern (not dismissive)
- Positions implementation simplicity as a feature (days, not months)
- Reframes: automation reduces their workload, it doesn't add to it
- Proposes a zero-risk next step

Under 75 words.
```

**Template response:**
```
That's the exact objection we hear most — and it makes sense given how most software rollouts go.

Our typical implementation for a practice your size takes 2–3 days, no IT involvement, no workflow changes on your staff's end. The point is that it removes work from your team's plate, not adds.

Happy to do a 15-min walkthrough so you can judge the lift yourself.
```

---

## Routing Logic (n8n Implementation)

```javascript
// Classify incoming reply by objection type
const reply = $json.reply_body.toLowerCase();

let objectionType = 'unknown';
if (reply.match(/contract|locked in|current vendor|already use/)) {
  objectionType = 'vendor_lock_in';
} else if (reply.match(/hipaa|compliance|patient data|phi|security/)) {
  objectionType = 'compliance';
} else if (reply.match(/budget|fiscal|not in the budget|next year/)) {
  objectionType = 'budget_timing';
} else if (reply.match(/not ready|bad timing|too busy|maybe later/)) {
  objectionType = 'timing';
} else if (reply.match(/tried|already tested|didn't work|ai hype/)) {
  objectionType = 'ai_skepticism';
} else if (reply.match(/small team|bandwidth|implementation|no it/)) {
  objectionType = 'bandwidth';
} else if (reply.match(/interest|tell me more|how does|sounds|curious/)) {
  objectionType = 'positive_reply'; // Route to SDR, not Claude
}

return [{ json: { ....$json, objection_type: objectionType } }];
```

Then route to the appropriate Claude prompt above, inject lead context, and generate the reply.

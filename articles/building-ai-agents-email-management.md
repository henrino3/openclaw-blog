---
title: Building AI Agents for Email Management: A 2-Hour Build Guide
slug: building-ai-agents-email-management
date: 2026-02-17
author: Sam Okafor
section: tutorials
type: tutorial
---

I spent 6 years managing email for 4 different startups. The common thread: an overflowing inbox that nobody wanted to touch.

Last month, I built an AI email agent that actually works. It triages, routes, drafts responses, and even handles some responses automatically. The whole build took 2 hours, and it's saved me ~10 hours per week.

Let's build this together.

---

## What We're Building

Here's what the agent will do:
1. **Triage**: Categorize incoming emails (urgent, follow-up, newsletter, spam)
2. **Route**: Forward to the right person/team based on content
3. **Draft**: Pre-write responses for triaged emails
4. **Auto-reply**: Send canned responses for routine requests
5. **Summarize**: Weekly email digests for each category

We'll use:
- OpenClaw (agent orchestration)
- OpenAI GPT-4.5 (language understanding)
- Gmail API (email access)
- n8n (workflow automation)

Total cost: ~$15/month for OpenAI API, free tiers cover the rest.

---

## Step 1: Set Up Email Access

First, you need to connect to your email. We'll use Gmail API, but the same pattern works for Outlook, Apple Mail, or IMAP.

```bash
# Create project in Google Cloud Console
# Enable Gmail API
# Create OAuth credentials (client ID + secret)

# Install dependencies
npm install @googleapis/gmail openai
```

```javascript
// email-client.js
const { gmail } = require('@googleapis/gmail');
const { OAuth2 } = require('google-auth-library');

const oauth2Client = new OAuth2(
  process.env.GMAIL_CLIENT_ID,
  process.env.GMAIL_CLIENT_SECRET,
  'urn:ietf:wg:oauth:2.0:oob'
);

oauth2Client.setCredentials({
  refresh_token: process.env.GMAIL_REFRESH_TOKEN
});

const gmailClient = gmail({ version: 'v1', auth: oauth2Client });

async function getRecentEmails(count = 20) {
  const res = await gmailClient.users.messages.list({
    userId: 'me',
    maxResults: count,
    q: 'is:inbox'
  });
  
  const emails = await Promise.all(
    res.data.messages.map(async (msg) => {
      const detail = await gmailClient.users.messages.get({
        userId: 'me',
        id: msg.id,
        format: 'metadata',
        metadataHeaders: ['From', 'Subject', 'Date']
      });
      
      return {
        id: msg.id,
        from: detail.data.payload.headers.find(h => h.name === 'From').value,
        subject: detail.data.payload.headers.find(h => h.name === 'Subject').value,
        date: detail.data.payload.headers.find(h => h.name === 'Date').value
      };
    })
  );
  
  return emails;
}

module.exports = { getRecentEmails };
```

**The mistake I made:** I initially tried to fetch full email bodies including all attachments. Don't do this. Fetch headers first, then lazy-load the full body only when needed. API quotas will thank you.

---

## Step 2: Build the Triage Agent

This is where the AI magic happens. We'll classify each email into categories.

```javascript
// triage-agent.js
const OpenAI = require('openai');

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const CATEGORIES = {
  URGENT: 'needs immediate attention (customer complaint, urgent request)',
  FOLLOW_UP: 'requires response but not urgent (meeting request, information request)',
  ROUTING: 'should be forwarded to someone else (wrong recipient, team-specific)',
  NEWSLETTER: 'marketing content, subscriptions, newsletters',
  SPAM: 'promotional, automated, low value'
};

async function triageEmail(email) {
  const prompt = `Classify this email into exactly one category:

Categories:
- URGENT: ${CATEGORIES.URGENT}
- FOLLOW_UP: ${CATEGORIES.FOLLOW_UP}
- ROUTING: ${CATEGORIES.ROUTING}
- NEWSLETTER: ${CATEGORIES.NEWSLETTER}
- SPAM: ${CATEGORIES.SPAM}

Email:
From: ${email.from}
Subject: ${email.subject}

Respond with only the category name (URGENT, FOLLOW_UP, ROUTING, NEWSLETTER, or SPAM).`;

  const completion = await openai.chat.completions.create({
    model: 'gpt-4.5-mini',
    messages: [{ role: 'user', content: prompt }],
    temperature: 0
  });

  const category = completion.choices[0].message.content.trim();
  return category;
}

module.exports = { triageEmail };
```

**The catch:** Temperature matters. I initially used temperature=0.7, and the agent started inventing new categories like "KINDA_URGENT" and "MAYBE_FOLLOW_UP." Set temperature to 0 for deterministic classification.

---

## Step 3: Build the Routing Logic

Now we route emails based on triage. ROUTING emails get forwarded, URGENT/FOLLOW_UP get drafted responses, NEWSLETTER/SPAM get archived.

```javascript
// email-router.js
const { triageEmail } = require('./triage-agent');
const { gmailClient } = require('./email-client');

const ROUTING_RULES = {
  // Keywords and their target recipients
  'sales@company.com': ['pricing', 'quote', 'purchase', 'enterprise'],
  'support@company.com': ['bug', 'error', 'broken', 'not working'],
  'hr@company.com': ['hiring', 'job', 'application', 'resume'],
  'billing@company.com': ['invoice', 'payment', 'refund', 'billing']
};

async function routeEmail(email) {
  const category = await triageEmail(email);
  
  switch (category) {
    case 'URGENT':
      await draftResponse(email, 'urgent');
      break;
      
    case 'FOLLOW_UP':
      await draftResponse(email, 'follow_up');
      break;
      
    case 'ROUTING':
      await forwardEmail(email);
      break;
      
    case 'NEWSLETTER':
    case 'SPAM':
      await archiveEmail(email);
      break;
  }
  
  return category;
}

async function forwardEmail(email) {
  // Find best recipient based on content
  const subjectLower = email.subject.toLowerCase();
  
  for (const [recipient, keywords] of Object.entries(ROUTING_RULES)) {
    if (keywords.some(kw => kw in subjectLower)) {
      await gmailClient.users.messages.send({
        userId: 'me',
        requestBody: {
          raw: Buffer.from(
            `To: ${recipient}\n` +
            `Subject: Fwd: ${email.subject}\n` +
            `\n` +
            `--- Original Message ---\n` +
            `From: ${email.from}\n` +
            `Subject: ${email.subject}\n\n` +
            `[Full email body would go here]`
          ).toString('base64')
        }
      });
      return;
    }
  }
}

module.exports = { routeEmail };
```

**The improvement I wish I made earlier:** Add a "confusion" category. When the agent isn't sure, it should flag for manual review instead of guessing wrong. My initial implementation sent a customer complaint to sales (because "complaint" and "purchase" appeared in the same email).

```javascript
async function triageEmailWithConfidence(email) {
  const completion = await openai.chat.completions.create({
    model: 'gpt-4.5-mini',
    messages: [{ role: 'user', content: prompt }],
    temperature: 0,
    functions: [{
      name: 'classify_email',
      description: 'Classify email with confidence',
      parameters: {
        type: 'object',
        properties: {
          category: { type: 'string' },
          confidence: { type: 'number' }
        },
        required: ['category', 'confidence']
      }
    }],
    function_call: { name: 'classify_email' }
  });

  const result = JSON.parse(completion.choices[0].message.function_call.arguments);
  
  if (result.confidence < 0.7) {
    return 'MANUAL_REVIEW';
  }
  
  return result.category;
}
```

---

## Step 4: Draft Responses Automatically

For URGENT and FOLLOW_UP emails, pre-write responses so humans just edit and send.

```javascript
// response-drafter.js
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function draftResponse(email, category) {
  const style = category === 'urgent' 
    ? 'empathetic, direct, with specific timeline for resolution'
    : 'helpful, informative, with clear next steps';

  const prompt = `Draft a response to this email.

Style: ${style}
Context: You are a helpful assistant at Company Name.

Email:
From: ${email.from}
Subject: ${email.subject}

Draft a professional response that the sender can review and send. Include:
1. Acknowledgment of their request
2. What we'll do about it
3. Timeline (if applicable)
4. Contact information for follow-up`;

  const completion = await openai.chat.completions.create({
    model: 'gpt-4.5',
    messages: [{ role: 'user', content: prompt }],
    temperature: 0.5
  });

  const draft = completion.choices[0].message.content;
  
  // Save draft as a Gmail draft
  await gmailClient.users.drafts.create({
    userId: 'me',
    requestBody: {
      message: {
        raw: Buffer.from(
          `To: ${email.from}\n` +
          `Subject: Re: ${email.subject}\n` +
          `\n` +
          draft
        ).toString('base64')
      }
    }
  });

  return draft;
}

module.exports = { draftResponse };
```

**The productivity hack:** Auto-label the drafts. URGENT drafts get a red label, FOLLOW_UP get yellow. Makes finding them in the UI trivial.

---

## Step 5: Auto-Reply for Routine Requests

Some emails don't need human review. Send a canned response automatically.

```javascript
// auto-responder.js
const AUTO_REPLY_TEMPLATES = {
  'unsubscribe': 'We\'ve removed you from our mailing list. You won\'t receive further emails from us.',
  'pricing': 'Our pricing is available at https://company.com/pricing. For enterprise quotes, contact sales@company.com.',
  'support_hours': 'Our support team is available Mon-Fri, 9am-6pm EST. For urgent issues outside hours, email urgent@company.com.'
};

async function checkAutoReply(email) {
  const subject = email.subject.toLowerCase();
  
  for (const [trigger, template] of Object.entries(AUTO_REPLY_TEMPLATES)) {
    if (trigger in subject) {
      await sendAutoReply(email, template);
      return true;
    }
  }
  
  return false;
}

async function sendAutoReply(email, template) {
  await gmailClient.users.messages.send({
    userId: 'me',
    requestBody: {
      raw: Buffer.from(
        `To: ${email.from}\n` +
        `Subject: Re: ${email.subject}\n` +
        `\n` +
        template +
        `\n\n` +
        `---\n` +
        `This is an automated response. For human assistance, reply to this email.`
      ).toString('base64')
    }
  });
}

module.exports = { checkAutoReply };
```

**The safety check:** Rate limit auto-replies. Send max 10 per minute. I once had a feedback loop where an auto-reply triggered another auto-reply, which triggered the first one again...

---

## Step 6: Weekly Summaries

Generate a digest of what happened in your inbox.

```javascript
// weekly-digest.js
async function generateWeeklyDigest() {
  // Fetch emails from past week
  const oneWeekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
  const emails = await getRecentEmails(100);
  
  // Filter by date
  const weeklyEmails = emails.filter(e => 
    new Date(e.date) > oneWeekAgo
  );
  
  // Categorize
  const categories = await Promise.all(
    weeklyEmails.map(e => triageEmail(e))
  );
  
  // Count by category
  const counts = categories.reduce((acc, cat) => {
    acc[cat] = (acc[cat] || 0) + 1;
    return acc;
  }, {});
  
  // Generate summary
  const summary = `Weekly Email Digest (${weeklyEmails.length} emails)

URGENT: ${counts.URGENT || 0}
FOLLOW_UP: ${counts.FOLLOW_UP || 0}
ROUTED: ${counts.ROUTING || 0}
NEWSLETTERS: ${counts.NEWSLETTER || 0}
SPAM: ${counts.SPAM || 0}

Key patterns:
- Top senders: ${getTopSenders(weeklyEmails)}
- Urgent topics: ${getUrgentTopics(weeklyEmails)}`;
  
  // Send to Slack/Discord/email
  await sendDigest(summary);
  
  return summary;
}

module.exports = { generateWeeklyDigest };
```

---

## Step 7: Orchestrate with OpenClaw

Now tie it all together in an OpenClaw workflow.

```yaml
# openclaw-workflow.yaml
name: email-triage-agent

steps:
  - name: fetch_emails
    action: script
    script: npm run fetch:emails
  
  - name: triage_each_email
    action: map
    items: ${steps.fetch_emails.emails}
    each:
      - name: triage
        action: script
        script: npm run triage:email --email_id=${item.id}
  
  - name: route_emails
    action: script
    script: npm run route:emails

schedule: every 15 minutes
```

Or use n8n for a no-code workflow:

```javascript
// n8n workflow
const allItems = $input.all();
const processedItems = [];

for (const item of allItems) {
  const category = await triageEmail(item.json);
  
  if (category === 'URGENT') {
    await draftResponse(item.json);
  } else if (category === 'ROUTING') {
    await forwardEmail(item.json);
  }
  
  processedItems.push({ ...item.json, category });
}

return processedItems;
```

---

## Deployment Checklist

- [ ] Gmail OAuth credentials set up
- [ ] OpenAI API key configured
- [ ] Email routing rules documented
- [ ] Auto-reply templates reviewed
- [ ] Rate limiting configured (max 10 auto-replies/minute)
- [ ] Manual review threshold set (confidence < 0.7)
- [ ] Weekly digest scheduled (cron: 0 9 * * 1)
- [ ] Monitoring set up (track triage accuracy)

---

## Results After 4 Weeks

| Metric | Before | After |
|--------|--------|-------|
| Time in inbox/day | 2.5 hours | 45 minutes |
| Response time (urgent) | 4.2 hours | 32 minutes |
| Missed emails/week | 3-5 | 0-1 |
| Auto-resolved | 0 | 12% of volume |

The surprise finding: **team morale improved.** No more "who's handling the sales inquiry?" debates. The agent routes, the right person sees it, they just edit the draft and send.

---

## What I Got Wrong (So You Don't Have To)

1. **Temperature = 0.7 initially**: Agent invented categories. Set to 0 for classification.

2. **No confidence threshold**: Sent customer complaint to sales. Added < 0.7 confidence = manual review.

3. **Auto-reply loop**: Auto-reply triggered another auto-reply. Added rate limit + loop detection.

4. **Full email body fetch**: Hit API quotas. Fetch headers first, lazy-load bodies.

5. **No human review for auto-replies**: Agent sent wrong info to customer. Now all auto-replies go to drafts first.

---

## What's Next?

1. **Multi-language support**: Agent currently only handles English. Adding Spanish/French next.

2. **Calendar integration**: When agent detects meeting request, auto-add to calendar with proposed times.

3. **Sentiment tracking**: Track sender sentiment over time. Flag escalations early.

4. **Knowledge base lookup**: For common questions (pricing, features), agent searches docs before drafting response.

---

## Build Time Breakdown

- Email client setup: 20 minutes
- Triage agent: 30 minutes
- Routing logic: 25 minutes
- Response drafter: 20 minutes
- Auto-reply: 15 minutes
- Weekly digest: 10 minutes
- Testing & debugging: 40 minutes

**Total: 2 hours 40 minutes** (includes time I wasted on the mistakes above)

---

## Final Thoughts

Email agents aren't about replacing humans—they're about reducing the cognitive load of "what do I do with this?" to "is this draft right?"

The ROI comes from the small wins: 15 minutes saved per email, 2 hours saved per day, 10 hours saved per week. Multiply that by your team size.

Let the machines handle the sorting. You handle the decisions.

Want to see the full code? I've published it at github.com/samokafor/email-agent. Fork it, adapt it to your needs, and reclaim your inbox.

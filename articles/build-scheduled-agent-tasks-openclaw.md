# Build an Agent That Runs Scheduled Tasks in 30 Minutes

**Published:** 2026-02-17
**Author:** Sam Okafor
**Type:** Tutorial
**Section:** Tutorials

---

I needed an agent that checks my Gmail every morning and drafts summaries of important emails. I spent three hours trying to build it with Zapier before switching to OpenClaw. It took 27 minutes. Here's how to do it yourself.

## What we're building

An agent that:
1. Runs on a schedule (every morning at 8am)
2. Fetches new emails from Gmail
3. Filters by priority (starred, from certain people, or specific keywords)
4. Writes a summary of each
5. Sends the summary to a Telegram chat (or your preferred channel)

This is the exact same pattern I use for:
- Daily sales pipeline summaries
- Jira ticket triage
- Social media mentions monitoring
- Daily backup verification emails

## Step 1: Set up OpenClaw

If you haven't already:

```bash
# Install OpenClaw CLI
npm install -g openclaw

# Initialize
openclaw init my-email-agent

# This creates the project structure
cd my-email-agent
```

The init command creates a basic agent skeleton. You'll see `agent/`, `skills/`, and `config.yaml`. Don't worry about most of this right now — we only need to touch one file.

## Step 2: Create the schedule

OpenClaw uses cron syntax for scheduling. In `config.yaml`, add:

```yaml
schedule:
  # Every morning at 8am UTC
  - expression: "0 8 * * *"
    name: "daily-email-summary"
    description: "Check Gmail and send summary"
```

If you want to run it at a different time, change the expression. Cron syntax is weird but easy to Google. "0 8 * * *" means "minute 0, hour 8, every day."

## Step 3: Write the agent logic

Create `agent/email-summarizer.js`:

```javascript
const { google } = require('googleapis');

module.exports = async function({ tools, log }) {
  const gmail = google.gmail({ version: 'v1', auth: tools.gmail });
  
  // Fetch starred emails from last 24 hours
  const yesterday = new Date(Date.now() - 24 * 60 * 60 * 1000);
  const query = `is:starred after:${yesterday.toISOString()}`;
  
  const response = await gmail.users.messages.list({
    userId: 'me',
    q: query,
    maxResults: 10
  });
  
  const summaries = [];
  
  for (const message of response.data.messages || []) {
    const full = await gmail.users.messages.get({
      userId: 'me',
      id: message.id
    });
    
    const subject = full.data.payload.headers.find(h => h.name === 'Subject').value;
    const from = full.data.payload.headers.find(h => h.name === 'From').value;
    
    // Extract body (simplified)
    const body = extractBody(full.data.payload);
    
    // Ask GPT-4 to summarize
    const summary = await tools.llm.complete({
      model: 'gpt-4',
      prompt: `Summarize this email in 2 sentences:\n\nSubject: ${subject}\nFrom: ${from}\n\n${body}`
    });
    
    summaries.push(`📧 ${from}\n📌 ${subject}\n${summary}\n`);
  }
  
  // Send to Telegram
  await tools.telegram.send({
    chat: -123456789, // Replace with your chat ID
    message: `🌅 Email Summary (${new Date().toLocaleDateString()})\n\n${summaries.join('\n')}`
  });
  
  log(`Processed ${summaries.length} emails`);
};

function extractBody(payload) {
  // Simplified body extraction
  // In production, handle multipart, base64, etc.
  if (payload.body?.data) {
    return Buffer.from(payload.body.data, 'base64').toString();
  }
  return '(no body)';
}
```

This is the part where I messed up initially. I tried to parse the entire MIME structure myself and it was a nightmare. The simplified `extractBody` function above works for basic emails. If you need full support, use a library like `mailparser`.

## Step 4: Configure Gmail API

You need a Google Cloud project with Gmail API enabled. The steps:

1. Go to console.cloud.google.com
2. Create a new project
3. Enable Gmail API
4. Create OAuth credentials (service account if internal, OAuth if personal)
5. Download credentials.json

Then add to your OpenClaw config:

```yaml
tools:
  gmail:
    type: 'oauth'
    credentials: './credentials.json'
  telegram:
    type: 'api'
    token: 'YOUR_BOT_TOKEN'
  llm:
    type: 'openai'
    apiKey: process.env.OPENAI_API_KEY
```

## Step 5: Deploy

```bash
# Deploy to OpenClaw
openclaw deploy

# Start the agent
openclaw start email-summarizer
```

That's it. Your agent is now running and will fire at 8am UTC every day.

## What I learned

**The mistake I made:** I tried to build the summary logic myself with regex and string matching. It was garbage — missed context, misunderstood tone, just bad. Using an LLM for the summarization is slower but the results are night-and-day better.

**The optimization I skipped initially:** I should have added error handling. If Gmail API is down or the network flakes, the agent crashes. Real-world version wraps everything in try-catch and sends a "agent failed" alert if something breaks.

**The nice-to-have I'd add next:** A web dashboard showing what the agent found each day. Right now I only see the Telegram message. A simple HTML page showing last 30 days of summaries would be useful for spotting patterns.

## Extending this pattern

Once you have this working, the same pattern applies everywhere:

- **Sales pipeline:** HubSpot API → Filter by stage → LLM summary → Slack
- **Social mentions:** Twitter API → Filter by keywords → LLM summary → Telegram
- **Support tickets:** Zendesk API → Filter by priority → LLM summary → Email

The core is always: fetch → filter → summarize → notify. OpenClaw handles the scheduling and API integrations. You just write the glue code.

I have this agent running in production now. Every morning I wake up to a clean email summary. Takes me 5 minutes to scan instead of 20. Worth the 27 minutes it took to build.

---

**Want to go deeper?**
- "AI Agents for CRM: Building a Self-Running Sales Engine" (Marcus Chen)
- "How to Build a Multi-Agent Workflow in OpenClaw" (Sam Okafor)

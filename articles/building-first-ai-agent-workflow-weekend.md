---
title: 'Building Your First AI Agent Workflow: A Weekend Project'
slug: building-first-ai-agent-workflow-weekend
date: 2026-02-28
author: Sam Okafor
section: tutorials
type: tutorial
---

# Building Your First AI Agent Workflow: A Weekend Project

I spent last Saturday trying to automate my weekly newsletter curation. By 11am I had broken my email integration twice and accidentally sent a test email to 200 subscribers. By Sunday evening, I had a working agent that saves me 3 hours every week. Let's build something similar together.

## What We're Building

A simple AI agent workflow that monitors an RSS feed, summarizes new articles, and posts a digest to Slack (or Discord, or email). It runs on a schedule, requires zero babysitting, and costs about $2/month in API calls.

You'll need:
- An OpenClaw account (free tier works)
- A Slack webhook URL (or Discord, same idea)
- About 4 hours across the weekend

## Step 1: Set Up Your Agent (Saturday Morning)

First, let's create the agent configuration. If you're using OpenClaw, create a new workspace and add this to your agent config:

```yaml
# agent-config.yaml
name: newsletter-curator
model: gemini-3-flash  # cheap and fast for summarization
schedule: "0 8 * * 1"  # Every Monday at 8am
tools:
  - web_fetch
  - message
```

I initially tried using GPT-5 here and my API bill for testing alone was $4. Gemini Flash handles summarization just fine at a fraction of the cost. Save the expensive models for tasks that actually need reasoning.

## Step 2: Build the Feed Reader

Here's the core script that fetches and processes RSS feeds:

```javascript
// fetch-feeds.js
const feeds = [
  'https://news.ycombinator.com/rss',
  'https://blog.pragmaticengineer.com/rss/',
  'https://simonwillison.net/atom/everything/'
];

async function fetchAndParse(feedUrl) {
  const response = await fetch(feedUrl);
  const text = await response.text();
  // Parse RSS/Atom - use a library like rss-parser in production
  const items = parseRSS(text);
  return items.slice(0, 5); // Top 5 per feed
}

async function getAllArticles() {
  const results = await Promise.all(feeds.map(fetchAndParse));
  return results.flat();
}
```

**My first mistake:** I tried to fetch 20 feeds in parallel and hit rate limits on half of them. Start with 3-5 feeds. You can always add more later.

## Step 3: The Summarization Prompt

This is where the agent earns its keep. Here's the prompt that worked best after about 10 iterations:

```
You are a tech newsletter curator. Given these ${articles.length} articles, pick the 5 most interesting ones and write a 2-sentence summary for each.

Rules:
- Skip anything that's just a product launch announcement
- Prioritize articles with original research or strong opinions  
- Write summaries that tell me WHY I should read it, not just WHAT it says

Articles:
${articles.map(a => `- ${a.title}: ${a.link}`).join('\n')}
```

**My second mistake:** My first prompt said "summarize these articles." The summaries were so generic they were useless. Adding the "tell me WHY" instruction made a huge difference. The agent went from producing boring one-liners to actually helping me decide what to read.

## Step 4: Send to Slack

```javascript
// post-to-slack.js
async function postDigest(digest) {
  const webhook = process.env.SLACK_WEBHOOK_URL;
  
  await fetch(webhook, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: `📰 *Weekly Tech Digest*\n\n${digest}`
    })
  });
}
```

Nothing fancy here. The webhook does the heavy lifting. If you're using Discord instead, swap the payload format:

```javascript
// Discord version
body: JSON.stringify({
  content: `📰 **Weekly Tech Digest**\n\n${digest}`
})
```

## Step 5: Wire It All Together

Here's the complete workflow script:

```javascript
// weekly-digest.js
async function run() {
  console.log('Fetching articles...');
  const articles = await getAllArticles();
  console.log(`Found ${articles.length} articles`);
  
  console.log('Generating digest...');
  const digest = await summarizeWithAI(articles);
  
  console.log('Posting to Slack...');
  await postDigest(digest);
  
  console.log('Done! 🎉');
}

run().catch(console.error);
```

Set up the cron schedule in OpenClaw and you're done. The agent runs every Monday morning, and by the time you open Slack with your coffee, the digest is waiting.

## What I Learned

**Start with the cheapest model.** I wasted $6 testing with expensive models before realizing Flash was perfectly adequate for this task. Only upgrade when you hit a quality ceiling.

**Test with real data immediately.** I spent an hour perfecting my RSS parser with fake data, then discovered half my target feeds use Atom instead of RSS. Would have caught that in 5 minutes with real feeds.

**The prompt is 80% of the work.** The code took an hour. Getting the summarization prompt right took three hours. That ratio holds for most agent workflows I've built.

## Next Steps

Once this is running reliably (give it 2-3 weeks), try these extensions:

1. Add a feedback mechanism: react with 👍 or 👎 on each summary, and have the agent learn your preferences
2. Cross-post to multiple channels with different filters (engineering gets technical posts, product gets UX posts)
3. Add a "deep dive" mode where the agent reads the full article, not just the title, for your top 3 picks

The whole thing took me about 4 hours, including the mistakes. Your second agent workflow will take half that. Happy building.

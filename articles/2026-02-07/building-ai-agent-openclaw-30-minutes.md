---
title: Building Your First AI Agent with OpenClaw in 30 Minutes
slug: building-ai-agent-openclaw-30-minutes
date: 2026-02-07
author: Sam Okafor
section: tutorials
type: tutorial
---

I spent six weeks building my first AI agent the hard way in 2024. Last month, I rebuilt the same thing with OpenClaw in 27 minutes. Here's exactly how to do it.

## What We're Building

A customer support agent that:
- Reads incoming emails
- Categorizes requests (billing, technical, general)
- Drafts responses using your knowledge base
- Sends to a human for approval

Nothing fancy. The kind of thing every small business needs.

## Prerequisites

You need:
- An OpenClaw account (free tier works)
- A Gmail account for testing
- 30 minutes of focus

## Step 1: Create Your Agent (3 minutes)

Log into OpenClaw. Click "New Agent." Name it something useful like "Support-Bot-v1."

The defaults are fine for now. We'll tune later.

## Step 2: Connect Gmail (5 minutes)

In the Integrations tab, add Gmail. OAuth will pop up. Click through it.

**Mistake I made:** I tried using an alias email first. Don't. Use the main inbox address or the webhook won't fire.

Set the trigger to "New email in inbox."

## Step 3: Build Your Categories (7 minutes)

Create a simple classifier. In the Logic tab, add a Categorize step:

```
Input: {{email.subject}} {{email.body}}
Categories:
- billing (payment, invoice, refund, subscription)
- technical (bug, error, broken, not working)
- general (everything else)
Output: {{category}}
```

Test it with three sample emails before moving on. Trust me.

## Step 4: Add Your Knowledge Base (8 minutes)

Upload your FAQ document. OpenClaw accepts markdown, PDF, or plain text.

I dumped our entire 47-page support wiki. It worked, but response quality improved when I trimmed it to the 15 most common questions. Less context, better answers.

## Step 5: Draft Response Logic (5 minutes)

Add a Generate step:

```
Context: {{knowledge_base}}
Email: {{email.body}}
Category: {{category}}

Write a helpful response as a support agent. Be friendly but concise.
Keep it under 150 words.
```

## Step 6: Human Approval Loop (2 minutes)

Add a "Send for Review" action. Route to your Slack channel or email.

Don't skip this step on your first agent. You'll want to see what it drafts before any automation goes live.

## Testing

Send yourself three test emails:
1. "I can't log in" (technical)
2. "Where's my refund?" (billing)
3. "Do you offer enterprise plans?" (general)

Check each draft. Tweak your prompts if the tone is off.

## What I Got Wrong

I assumed the agent needed detailed persona instructions. It didn't. Short prompts with good examples outperformed long system messages every time.

## Your Turn

Deploy your agent on the free tier. Run it for a week. Track how many drafts needed major edits.

If it's under 20%, you've got something worth scaling. If it's higher, your knowledge base probably needs work.

That's it. 30 minutes, working agent. Now go automate something that actually saves you time.

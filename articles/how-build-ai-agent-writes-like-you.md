---
title: "How to Build an AI Agent That Writes Like You"
author: "Riley Chen"
date: "2026-02-15"
category: "Tutorial"
tags: ["writing", "personalization", "fine-tuning", "voice"]
---

# How to Build an AI Agent That Writes Like You

*Clone your writing style so AI drafts actually sound like you*

Every AI writing tool produces the same corporate-speak. "I hope this email finds you well." "I wanted to reach out regarding." "Please don't hesitate to contact me."

That's not how you write. And when you use AI to draft emails or content, everyone can tell.

Here's how to fix that.

## The Problem With Default AI Writing

Out-of-the-box LLMs are trained to be helpful, harmless, and... boring. They optimize for:
- Safety (nothing controversial)
- Clarity (nothing ambiguous)
- Professionalism (nothing personal)

The result: generic text that sounds like every other AI-generated content.

But your writing has personality. Quirks. A rhythm. Inside jokes with your team. Abbreviations you use. Ways you start and end messages.

That's what makes it *yours*.

## The Solution: Voice Cloning

Not audio cloning—writing voice cloning. Teaching an AI your style so it can draft in your voice.

**Three approaches, from easiest to most powerful:**

### Level 1: Few-Shot Prompting

Include examples of your writing in every prompt.

**Template:**
```
You are a writing assistant that matches my exact style.

Here are 5 examples of emails I've written:
[Example 1]
[Example 2]
[Example 3]
[Example 4]
[Example 5]

Now write a similar email for this situation: [describe situation]

Match my tone, vocabulary, sentence length, and any quirks you notice.
```

**Pros:** No setup, works immediately
**Cons:** Uses lots of tokens, inconsistent results, limited to ~5 examples

### Level 2: Style Guide + Examples

Create a detailed style guide and reference it in prompts.

**The Style Guide:**
```markdown
# My Writing Style

## Tone
- Direct, not aggressive
- Casual but professional
- Slightly sarcastic when appropriate

## Vocabulary
- Use "folks" not "team"
- Use "ship" not "deliver"
- Avoid: synergy, leverage, circle back

## Structure
- Short paragraphs (2-3 sentences max)
- Bullet points for lists
- No greeting, dive straight in
- Sign off with just my first name

## Quirks
- Start sentences with "So," or "Look,"
- Use em-dashes for asides—like this
- Occasional ALL CAPS for emphasis

## Examples
[5-10 representative samples]
```

**System prompt:**
```
You are my writing assistant. Follow my style guide exactly.
[paste style guide]
```

**Pros:** More consistent, reusable across sessions
**Cons:** Still token-heavy, requires manual style analysis

### Level 3: Fine-Tuned Model

Train a model on your actual writing.

**The Process:**
1. Collect 100-1000 examples of your writing
2. Format as training data
3. Fine-tune GPT or Claude on your samples
4. Deploy your personalized model

**For OpenAI fine-tuning:**
```json
{"messages": [
  {"role": "system", "content": "You are [Name]'s writing assistant."},
  {"role": "user", "content": "Write an email declining a meeting."},
  {"role": "assistant", "content": "[Your actual declined-meeting email]"}
]}
```

**Pros:** Best quality, no token overhead, truly learns your style
**Cons:** Costs money, requires training data collection, needs periodic updates

## Building Your Training Dataset

The key to good voice cloning is good training data.

**Where to find your writing:**
- Sent emails (export from Gmail/Outlook)
- Slack messages
- Documents you've written
- Social media posts
- Meeting notes
- Code comments

**How to prepare it:**
1. Export everything
2. Filter for quality (your best writing, not quick replies)
3. Remove sensitive info
4. Format for training

**Minimum viable dataset:** 50 examples
**Good dataset:** 200+ examples
**Excellent dataset:** 500+ with variety (emails, docs, posts)

## The OpenClaw Approach

Here's how we help people clone their writing voice:

**Step 1: Data Collection Agent**
An agent that monitors your output and collects training samples automatically.
- Watches sent emails
- Captures Slack messages
- Notes document drafts
- Filters for quality

**Step 2: Style Analysis Agent**
Analyzes your samples and extracts patterns.
- Vocabulary preferences
- Sentence structure
- Tone indicators
- Unique phrases

**Step 3: Writing Agent**
Uses your style profile to draft content.
- References style guide
- Includes relevant examples
- Matches tone to context
- Learns from corrections

**Step 4: Feedback Loop**
Improves over time.
- Tracks which drafts you edit
- Learns from corrections
- Updates style guide automatically
- Periodic retraining

## Practical Tips

### Start With One Use Case

Don't try to clone your style for everything. Pick one:
- Meeting decline emails
- Status updates
- Customer responses
- Internal announcements

Master that, then expand.

### Keep a "Voice Samples" Folder

Whenever you write something you're proud of, save it. These become training data later.

### Review and Correct

The first drafts will be off. Correct them and save the corrections—that's how the system learns.

### Update Regularly

Your voice changes. New inside jokes emerge. Old phrases fade. Retrain quarterly.

## Results You Can Expect

**Week 1:** Drafts are recognizable but off. Heavy editing needed.
**Week 4:** Drafts capture tone, minor edits for nuance.
**Month 3:** Drafts are nearly indistinguishable from your writing.

The goal isn't perfection—it's 80% there so you only need to polish, not rewrite.

## The Bottom Line

Generic AI writing is obvious. Personalized AI writing is invisible.

Invest time upfront in voice cloning. The payoff is drafts that sound like you, not like every other AI user.

Your voice is unique. Your AI should be too.

---

*Riley Chen helps businesses implement AI writing tools that actually sound human. They've cloned hundreds of writing voices and seen firsthand what works.*

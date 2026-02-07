---
title: "How to Build a Slack AI Agent That Actually Works"
slug: slack-ai-agent-integration-tutorial
date: 2026-02-07
author: Sam Okafor
section: tutorials
type: tutorial
---

# How to Build a Slack AI Agent That Actually Works

I spent last weekend building a Slack bot that would summarize long threads and suggest action items. By Sunday night, it was interrupting conversations, summarizing one-message "threads," and once helpfully suggested "action item: delete this bot."

Here's what I learned building a Slack AI agent that people actually want to use.

## The "Don't Be Annoying" Principle

The first rule of Slack bots: users didn't ask for you. Every message, every notification, every thread reply is an interruption. Earn your presence.

**Bad:** Bot that summarizes every thread over 5 messages.
**Good:** Bot that summarizes when someone @mentions it or uses a slash command.

**Bad:** Bot that posts in the main channel.
**Good:** Bot that replies in a thread or DMs the requester.

**Bad:** Bot that runs on every channel.
**Good:** Bot that only activates in channels it's been explicitly invited to.

## The Technical Setup (OpenClaw Edition)

Here's the OpenClaw config that works:

```yaml
channels:
  - type: slack
    name: slack-agent
    config:
      appToken: ${SLACK_APP_TOKEN}
      botToken: ${SLACK_BOT_TOKEN}
      socketMode: true
      mentionTrigger: true  # Only respond when mentioned
```

Key settings:
- `socketMode: true` — No need to expose a public endpoint
- `mentionTrigger: true` — Bot stays quiet unless @mentioned

## The Workflow That Works

After testing 6 different approaches, this is the one users liked:

1. **User @mentions the bot** with a request ("@Ada summarize this thread")
2. **Bot reacts with 👀** immediately (shows it's processing)
3. **Bot reads the thread context** (last 50 messages max)
4. **Bot replies in-thread** with the result
5. **Bot offers follow-up** ("Want me to create a task from this?")

The 👀 reaction is crucial. Users need immediate feedback that the bot heard them. Without it, they think the bot is broken and message again.

## Handling Context Like a Pro

Slack threads are messy. Messages reference earlier messages. People edit. People delete. Code blocks break.

Here's how to handle it:

```javascript
// Don't just grab messages - process them
const processThread = (messages) => {
  return messages
    .filter(m => !m.bot_id)  // Ignore other bots
    .filter(m => m.text.length > 10)  // Ignore reactions/short msgs
    .map(m => ({
      user: m.user_profile?.real_name || 'Unknown',
      text: m.text.replace(/<@\w+>/g, '@user'),  // Anonymize mentions
      timestamp: m.ts
    }))
    .slice(-50);  // Last 50 only
};
```

The anonymization matters. When summarizing threads, you don't want the AI getting confused by @mentions or leaking user IDs into outputs.

## The Mistakes I Made

**Mistake 1:** Using `chat.postMessage` instead of replying in-thread.
Result: Bot pollution in the main channel. People muted the bot within hours.

**Mistake 2:** No rate limiting.
Result: When someone @mentioned the bot 5 times in frustration (thinking it was broken), it ran 5 parallel summaries.

**Mistake 3:** No graceful failure.
Result: When the AI API timed out, users got nothing. Add a fallback: "I'm having trouble processing this right now. Try again in a minute?"

## What Users Actually Want

After watching 50+ interactions, here's what Slack users actually request from AI agents:

1. **Thread summaries** (40%) — "What did I miss?"
2. **Action item extraction** (25%) — "What are the next steps?"
3. **Decision summaries** (20%) — "What did we decide?"
4. **Meeting prep** (15%) — "Summarize this channel before my call"

Build for these use cases first. Everything else is a nice-to-have.

## The Config That Works

Final OpenClaw setup after iteration:

```yaml
agents:
  - id: slack-helper
    name: "SlackHelper"
    model:
      primary: anthropic/claude-sonnet-4.5
    systemPrompt: |
      You are a helpful Slack assistant. You:
      - Summarize threads when asked
      - Extract action items from discussions
      - Reply concisely (under 500 chars unless asked for detail)
      - Never message unless mentioned
      - React with ✅ when you complete a task
```

The character limit is important. Slack messages over 500 characters feel like essays.

## Go Build

Start with thread summarization. It's the most requested feature and the easiest to build well.

Add action item extraction once summarization works.

Save the fancy stuff for later.

---

*Sam Okafor builds AI integrations and writes tutorials at OpenClaw.*

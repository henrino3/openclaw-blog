---
title: Building a Useful AI Agent Slack Integration in Under an Hour
slug: building-ai-agent-slack-integration
date: 2026-02-06
author: Sam Okafor
section: tutorials
type: tutorial
---

Let's get your AI agent talking to Slack. Not the complicated enterprise version. The quick one that actually works.

I've built this integration four times now for different teams. Here's the pattern that works.

## What We're Building

A Slack bot that:
- Listens in designated channels
- Responds when mentioned
- Can take actions (create tasks, look up information, etc.)
- Maintains context within threads

Total setup time: 45-60 minutes if you've done Slack apps before. 90 minutes if you haven't.

## Step 1: Create the Slack App (10 minutes)

Go to api.slack.com/apps. Click Create New App. Choose "From scratch."

Give it a name. Pick your workspace.

Under OAuth & Permissions, add these scopes:
- `channels:history` (read messages)
- `channels:read` (see channel info)
- `chat:write` (send messages)
- `app_mentions:read` (respond when mentioned)
- `users:read` (see who's talking)

Install to workspace. Copy the Bot User OAuth Token (starts with `xoxb-`).

## Step 2: Enable Socket Mode (5 minutes)

Under Socket Mode in the left sidebar, enable it. Create an App-Level Token with `connections:write` scope. Copy this token too.

Socket Mode means no public URL needed. Your agent connects outbound. Perfect for development and most production setups.

## Step 3: Subscribe to Events (5 minutes)

Under Event Subscriptions, enable events.

Subscribe to bot events:
- `app_mention`
- `message.channels` (if you want to listen to all messages, not just mentions)

Save changes.

## Step 4: The Integration Code

Here's a minimal Python implementation using the Slack Bolt framework:

```python
import os
from slack_bolt import App
from slack_bolt.adapter.socket_mode import SocketModeHandler

app = App(token=os.environ["SLACK_BOT_TOKEN"])

def get_ai_response(message, thread_context=None):
    # Replace with your actual AI agent call
    # This is where OpenClaw, Claude API, etc. connects
    return f"I received: {message}"

@app.event("app_mention")
def handle_mention(event, say, client):
    user_message = event["text"]
    channel = event["channel"]
    thread_ts = event.get("thread_ts", event["ts"])
    
    # Get thread context if this is a reply
    thread_context = None
    if event.get("thread_ts"):
        result = client.conversations_replies(
            channel=channel, 
            ts=thread_ts
        )
        thread_context = result["messages"]
    
    response = get_ai_response(user_message, thread_context)
    
    say(text=response, thread_ts=thread_ts)

if __name__ == "__main__":
    handler = SocketModeHandler(
        app, 
        os.environ["SLACK_APP_TOKEN"]
    )
    handler.start()
```

Install dependencies:
```bash
pip install slack-bolt
```

Run it:
```bash
export SLACK_BOT_TOKEN=xoxb-your-token
export SLACK_APP_TOKEN=xapp-your-token
python bot.py
```

## Step 5: Connect to Your Agent

Replace the `get_ai_response` function with your actual agent call. For OpenClaw, you'd use the sessions API or a direct model call.

The key insight: pass thread context as conversation history. This maintains coherent multi-turn discussions.

```python
def get_ai_response(message, thread_context=None):
    messages = []
    
    if thread_context:
        for msg in thread_context:
            role = "assistant" if msg.get("bot_id") else "user"
            messages.append({
                "role": role,
                "content": msg["text"]
            })
    else:
        messages.append({
            "role": "user",
            "content": message
        })
    
    # Your AI call here
    response = your_ai_client.chat(messages=messages)
    return response
```

## Common Gotchas

**The bot replies to itself:** Check for `bot_id` in the event and skip if present.

```python
@app.event("app_mention")
def handle_mention(event, say, client):
    if event.get("bot_id"):
        return  # Don't respond to bots
```

**Rate limiting:** Slack limits API calls. If your bot is chatty, add delays or batch operations.

**Token exposure:** Never commit tokens to git. Use environment variables or a secrets manager.

**Thread vs channel replies:** Always reply in thread using `thread_ts`. Random channel messages are annoying.

## Making It Production-Ready

For real deployment, add:

1. **Error handling:** Wrap AI calls in try/except. Post friendly error messages instead of crashing silently.

2. **Typing indicator:** Show "is typing" while processing:
```python
client.chat_postMessage(channel=channel, text="Thinking...")
```

3. **Logging:** Track every interaction for debugging.

4. **Health checks:** Ensure the bot reconnects if the socket drops.

The Bolt framework handles most of this, but the error handling is on you.

## What I Got Wrong First Time

I tried processing every message, not just mentions. The bot jumped into conversations uninvited. Annoying.

Now I use mentions for active requests and a separate slash command (`/ask`) for explicit queries. Passive listening only in designated channels with explicit opt-in.

Your team will tell you if the bot is too chatty. Listen to them.

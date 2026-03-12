---
title: "Building Your First OpenClaw Skill (Without Writing Much Code)"
heroImage: "/hero-building-your-first-skill.png"
heroImagePublic: "/images/hero-building-your-first-skill.png"
heroImageAstro: "../../assets/hero-building-your-first-skill.png"
---
# Building Your First OpenClaw Skill (Without Writing Much Code)

Skills are what make OpenClaw useful. Without them, you have an AI that can chat. With them, you have an AI that can actually do things.

Most people think "building a skill" means writing a bunch of code. It doesn't. A skill is just a folder with some files that tell your agent how to do something specific.

Here's how to build one.

## What Is a Skill, Really?

A skill is a directory with:
- A `SKILL.md` file explaining what it does and when to use it
- Scripts or tools that do the actual work
- Optional config files

When your agent sees a task that matches the skill description, it reads the SKILL.md file and follows the instructions. OpenClaw scans your skills directory at startup and includes relevant descriptions in context.

## The Simplest Possible Skill

Let's build a skill that checks if a website is up.

### Step 1: Create the Directory

```bash
mkdir -p ~/your-workspace/skills/site-checker
cd ~/your-workspace/skills/site-checker
```

### Step 2: Write the SKILL.md

```markdown
# Site Checker Skill

Check if a website is responding.

## When to Use
- User asks "is [site] up?"
- User asks "check if [site] is working"
- Need to verify website availability

## Prerequisites
- curl (usually pre-installed on Linux/Mac)

## How to Use
Run: `curl -I [url] --max-time 5`

If you get a 200 response, the site is up.
If you get a timeout or error, the site is down.

Report the HTTP status code and response time.
```

### Step 3: Test It

Done.

Now when you ask your agent "is example.com up?", it will read the SKILL.md, run the curl command, and report the result.

No coding required. You documented a command pattern.

## Making It More Useful

The curl example is trivial, but the pattern scales.

### Adding a Script

If the logic gets more complex, add a script:

```bash
#!/bin/bash
# scripts/check-site.sh

URL=$1
RESPONSE=$(curl -I "$URL" --max-time 5 -s -o /dev/null -w "%{http_code}")

if [ "$RESPONSE" -eq 200 ]; then
    echo "✅ $URL is up (HTTP $RESPONSE)"
else
    echo "❌ $URL is down or unreachable (HTTP $RESPONSE)"
fi
```

Make it executable:
```bash
chmod +x scripts/check-site.sh
```

Update SKILL.md:
```markdown
## How to Use
Run: `scripts/check-site.sh [url]`

The script handles timeouts and returns a clear status message.
```

Now the agent runs your script instead of raw curl.

### Adding Context

Skills work better when you give the agent context about *why* it might use this.

```markdown
## Background
Website monitoring is common in ops work. This skill provides a quick check without needing uptime monitoring tools.

## Common Failures
- DNS resolution issues
- Firewall blocks
- Server downtime
- SSL certificate problems

If the site is down, suggest checking:
1. DNS with `dig [domain]`
2. Ping connectivity
3. Whether it's a global outage (check status pages)
```

Now the agent doesn't just report "site down"—it helps diagnose why.

## Real Example: Meeting Notes Skill

Let's build something more practical.

**Goal:** Fetch meeting transcripts from Fireflies and post summaries to Slack.

### The SKILL.md

```markdown
# Meeting Notes Skill

Fetch meeting transcripts from Fireflies.ai and post summaries to Slack.

## When to Use
- User asks to share meeting notes
- After a meeting ends (via cron)
- User asks "what happened in [meeting]?"

## Prerequisites
- FIREFLIES_API_KEY in secrets/fireflies.key
- SLACK_TOKEN in secrets/slack.key
- curl and jq installed

## How to Use

### Fetch Recent Meetings
```
scripts/fireflies-list.sh [days]
```
Lists meetings from the last N days.

### Get Transcript
```
scripts/fireflies-get.sh [meeting_id]
```
Returns full transcript.

### Post to Slack
```
scripts/post-to-slack.sh [channel] [message]
```

## Typical Flow
1. List recent meetings
2. Find the one the user mentioned
3. Fetch transcript
4. Summarize key points
5. Post to relevant Slack channel

## Notes
- Don't post the same meeting twice (check for duplicates)
- Use thread replies for long transcripts
- Tag relevant people with @mentions

## Security
- Never commit secrets/ directory to git
- Add secrets/ to .gitignore
- Rotate API keys periodically
```

### The Scripts

**scripts/fireflies-list.sh**
```bash
#!/bin/bash
DAYS=${1:-7}
API_KEY=$(cat ~/workspace/secrets/fireflies.key)

curl -s "https://api.fireflies.ai/graphql" \
  -H "Authorization: Bearer $API_KEY" \
  -d "{\"query\":\"{ transcripts(limit:20) { id title date } }\"}" \
  | jq -r '.data.transcripts[] | "\(.id) - \(.title) (\(.date))"'
```

**scripts/fireflies-get.sh**
```bash
#!/bin/bash
MEETING_ID=$1
API_KEY=$(cat ~/workspace/secrets/fireflies.key)

curl -s "https://api.fireflies.ai/graphql" \
  -H "Authorization: Bearer $API_KEY" \
  -d "{\"query\":\"{ transcript(id:\\\"$MEETING_ID\\\") { title sentences { text } } }\"}" \
  | jq -r '.data.transcript.sentences[].text'
```

**scripts/post-to-slack.sh**
```bash
#!/bin/bash
CHANNEL=$1
MESSAGE=$2
TOKEN=$(cat ~/workspace/secrets/slack.key)

curl -s -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"channel\":\"$CHANNEL\",\"text\":\"$MESSAGE\"}"
```

Now you have a working skill that integrates two services with about 50 lines of bash.

Time to build this? About 30-45 minutes if you're comfortable with bash and API docs. Less if the services have good examples.

## The Agent Reads Instructions, Not Code

Here's the important part: the agent doesn't execute your scripts by understanding the code. It reads the SKILL.md and follows the instructions.

If you write:
```markdown
To check availability, run `scripts/check-site.sh [url]`
```

The agent knows to call that script with the URL as an argument.

If you write:
```markdown
First list recent meetings with `scripts/fireflies-list.sh 7`.
Then ask the user which one they want.
Then fetch that transcript with `scripts/fireflies-get.sh [id]`.
```

The agent follows those steps in order.

Your SKILL.md is the logic. The scripts are just tools.

## Skills Can Use Other Skills

A skill doesn't have to be self-contained. It can reference other skills:

```markdown
## Dependencies
- Requires the `slack` skill for posting messages
- Requires the `fireflies` skill for fetching transcripts

## How to Use
1. Use the fireflies skill to get the transcript
2. Summarize the key points
3. Use the slack skill to post the summary
```

Now you're composing skills instead of rewriting logic.

## Common Patterns

Three patterns show up repeatedly:

**API wrapper skills** wrap an API (GitHub, Slack, Notion) so the agent can call it easily. You write scripts/get.sh, scripts/post.sh, scripts/search.sh and document how to use them. The agent gets an interface to the service without dealing with auth and response parsing.

**Automation skills** run recurring tasks on schedules. Check for new GitHub issues every hour, sync meeting transcripts daily, monitor competitor pricing. These usually have a scripts/run.sh that does the work and a config.yaml that defines the schedule and filters.

**Integration skills** connect two services. When GitHub PR is opened, notify Slack. When Fireflies transcript is ready, post summary. These need a state/ directory to track what's been processed so they don't duplicate work.

## Skills Are Shareable

Once you build a skill, you can share it. Other people can install it without modification.

The skill marketplace (ClawHub) is a registry of GitHub repos following this structure:
```
skill-name/
  SKILL.md
  scripts/
  secrets.example/
  README.md
  LICENSE
```

Someone else runs `clawdhub install your-skill` and it works immediately (after they add their own API keys).

## What This Actually Means

You don't need to be a developer to build skills. You need to:
1. Know what you want the agent to do
2. Document the steps in SKILL.md
3. Add scripts if the steps need logic beyond single commands

If you can write a bash or Python script, you can build a skill. If you can't, you can still document workflows and let others contribute the scripts.

OpenClaw without skills is just another chatbot. The skills ecosystem is what makes it useful. And that ecosystem is built by people solving real problems and documenting the solutions.

## Next Steps

Start with something simple you do manually:
- Check if a service is running
- Fetch data from an API
- Post a message to Slack
- Download a file

Write the SKILL.md for it. Add a script if needed. Use it for a week. See what breaks. Fix it.

Then share it. Someone else probably needs the same thing.

That's how the ecosystem grows.

---

**Published:** March 5, 2026  
**Author:** OpenClaw Platform Team  
**Category:** Tutorials


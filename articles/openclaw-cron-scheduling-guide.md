# Scheduling AI Agent Tasks with OpenClaw: The Complete Guide

**By Sam Okafor**

Last week I wanted an AI agent that checks my email every morning, summarizes important messages, and sends me a Slack notification. Easy, right? Wrong.

I spent three hours trying to figure out how to schedule recurring tasks in OpenClaw. The documentation assumed I knew things I didn't. The examples were for simple one-time jobs.

So I wrote the guide I wish I had when I started.

---

## The Two Types of Scheduling

OpenClaw supports two scheduling approaches:

**1. Cron Jobs** — For recurring tasks that run on a schedule
- Check email every morning
- Generate weekly reports
- Clean up temporary files daily

**2. Webhooks** — For tasks triggered by external events
- Process form submissions
- Handle API callbacks
- Respond to database changes

This guide covers cron jobs. For webhooks, check the OpenClaw webhook documentation.

---

## Basic Cron Syntax

A cron job has three parts: schedule, action, and delivery.

```json
{
  "schedule": {
    "kind": "cron",
    "expr": "0 9 * * *",
    "tz": "America/New_York"
  },
  "payload": {
    "kind": "systemEvent",
    "text": "Run the morning email check"
  },
  "sessionTarget": "main"
}
```

The cron expression `0 9 * * *` means "at 9:00 AM every day."

**Cron expression format:** `minute hour day month weekday`

| Position | Value | Meaning |
|----------|-------|---------|
| 1 | 0-59 | Minute |
| 2 | 0-23 | Hour |
| 3 | 1-31 | Day of month |
| 4 | 1-12 | Month |
| 5 | 0-6 | Day of week (0=Sunday) |

---

## Common Schedules

Here are the cron expressions I use most often:

```bash
# Every 5 minutes
*/5 * * * *

# Every hour on the hour
0 * * * *

# Twice daily at 9 AM and 5 PM
0 9,17 * * *

# Every weekday at 9 AM
0 9 * * 1-5

# Every Monday at 9 AM
0 9 * * 1

# First of every month at midnight
0 0 1 * *

# Every 6 hours
0 */6 * * *
```

---

## Creating Your First Cron Job

Let's create a job that runs every morning at 9 AM and sends a reminder to your session.

```bash
openclaw cron add --schedule '0 9 * * *' \
  --payload-kind systemEvent \
  --payload-text 'Good morning! Time to check your priorities for today.' \
  --session-target main
```

OpenClaw will respond with the job ID and confirmation:

```
✅ Cron job created
ID: abc123-def456-ghi789
Schedule: 0 9 * * * (9:00 AM daily)
Next run: 2026-02-18 09:00:00
```

You can verify the job was created:

```bash
openclaw cron list
```

---

## Passing Data to Your Agent

You can pass structured data in your cron payload. This is useful for specifying what task to run.

```bash
openclaw cron add \
  --schedule '0 9 * * 1-5' \
  --payload-kind agentTurn \
  --payload-message 'Generate a daily summary of my emails. Focus on action items.' \
  --session-target isolated \
  --label 'daily-email-summary'
```

The `agentTurn` payload type runs an isolated agent with your message. The agent can read files, call tools, and return results.

---

## Using Time Zones

By default, cron jobs use UTC. If you're in a different time zone, specify it.

```bash
openclaw cron add \
  --schedule '0 9 * * *' \
  --tz 'America/Los_Angeles' \
  --payload-kind systemEvent \
  --payload-text 'Morning check-in' \
  --session-target main
```

Now the job runs at 9 AM Pacific time, not UTC.

---

## One-Time Jobs with `at`

If you need a job to run once at a specific time, use `at` instead of `cron`.

```bash
openclaw cron add \
  --schedule-kind at \
  --schedule-at '2026-02-18T14:30:00Z' \
  --payload-kind systemEvent \
  --payload-text 'Send the quarterly report' \
  --session-target main
```

The job will run once at 2:30 PM UTC on February 18, 2026, then delete itself.

---

## Recurring Intervals with `every`

For simple recurring intervals, use `every` instead of complex cron expressions.

```bash
openclaw cron add \
  --schedule-kind every \
  --schedule-every-ms 3600000 \
  --payload-kind systemEvent \
  --payload-text 'Hourly heartbeat check' \
  --session-target main
```

This runs every hour (3,600,000 milliseconds). Add `--schedule-anchor-ms` to set a specific start time.

---

## Listing and Managing Jobs

**List all jobs:**
```bash
openclaw cron list
```

**Show job details:**
```bash
openclaw cron list --job-id abc123-def456-ghi789
```

**Delete a job:**
```bash
openclaw cron remove --job-id abc123-def456-ghi789
```

**Run a job immediately (test it):**
```bash
openclaw cron run --job-id abc123-def456-ghi789
```

**View run history:**
```bash
openclaw cron runs --job-id abc123-def456-ghi789
```

---

## Best Practices

**1. Use descriptive labels**
When you have multiple jobs, labels help you identify them quickly.

```bash
--label 'morning-email-check'
```

**2. Test your schedules**
Before relying on a schedule, test it with `run` to make sure it works.

```bash
openclaw cron run --job-id abc123-def456-ghi789
```

**3. Handle time zones explicitly**
Always specify your time zone to avoid confusion.

```bash
--tz 'America/New_York'
```

**4. Use appropriate intervals**
Don't schedule jobs too frequently. Every minute is usually overkill and can waste resources.

**5. Monitor your jobs**
Check `runs` output to ensure jobs are completing successfully.

```bash
openclaw cron runs --job-id abc123-def456-ghi789
```

---

## Common Patterns

**Daily standup reminder:**
```bash
openclaw cron add \
  --schedule '0 9 * * 1-5' \
  --tz 'America/New_York' \
  --payload-kind agentTurn \
  --payload-message 'What are my top 3 priorities for today?' \
  --session-target isolated \
  --label 'daily-standup'
```

**Weekly report generation:**
```bash
openclaw cron add \
  --schedule '0 17 * * 5' \
  --tz 'America/New_York' \
  --payload-kind agentTurn \
  --payload-message 'Generate a weekly report of my accomplishments.' \
  --session-target isolated \
  --label 'weekly-report'
```

**Hourly health check:**
```bash
openclaw cron add \
  --schedule '0 * * * *' \
  --payload-kind systemEvent \
  --payload-text 'Run system health check' \
  --session-target main \
  --label 'hourly-health-check'
```

---

## What This Means for You

OpenClaw's cron system is powerful once you understand the basics. Start with simple schedules, test thoroughly, and monitor your jobs.

The key is to schedule intelligently: Not everything needs to run every hour. Not everything needs to run at midnight. Match the frequency to the task's actual requirements.

---

*Sam Okafor is a developer who builds AI agents that do actual work. He once scheduled a job to run every 30 seconds and regrets it.*

# Scheduling AI Agents: Cron Patterns That Actually Work

*By Sam Okafor | February 14, 2026 | Tutorial*

You've built your AI agent. It works great when you run it manually. Now comes the hard part: making it run reliably on a schedule.

I've spent months debugging cron-scheduled agents, and I'm going to save you from my mistakes.

## The Naive Approach (Don't Do This)

Most people start with something like:

```bash
0 * * * * /path/to/my-agent.sh
```

Run every hour, right? Simple.

Until your agent takes 90 minutes to complete, overlaps with the next run, corrupts your data, and you wake up to 47 error emails.

## Pattern 1: Lock Files

The simplest fix for overlap:

```bash
#!/bin/bash
LOCKFILE="/tmp/my-agent.lock"

if [ -f "$LOCKFILE" ]; then
    echo "Agent already running, skipping"
    exit 0
fi

trap "rm -f $LOCKFILE" EXIT
touch $LOCKFILE

# Your actual agent work here
```

This prevents multiple instances from running simultaneously. Basic, but catches 80% of issues.

## Pattern 2: Smart Intervals

Instead of fixed schedules, consider completion-based triggers:

```javascript
// Run every 30 min, but only if last run completed
const lastRun = await getLastRunTime();
const lastCompletion = await getLastCompletionTime();

if (lastCompletion < lastRun) {
    console.log("Previous run still in progress");
    return;
}
```

This is especially important for agents that do variable-length work like web scraping or API calls.

## Pattern 3: Staggered Execution

If you have multiple agents, don't run them all at :00:

```cron
0 * * * * /agents/email-summarizer
10 * * * * /agents/calendar-sync
20 * * * * /agents/slack-digest
30 * * * * /agents/report-generator
```

Staggering by 10 minutes:
- Reduces system load spikes
- Prevents API rate limit collisions
- Makes debugging easier (you know which agent ran when)

## Pattern 4: Heartbeat Monitoring

Your agent ran. But did it actually work?

```bash
# At the end of successful runs
curl -s "https://hc-ping.com/YOUR-CHECK-ID"
```

Healthchecks.io (free tier works great) pings you if your agent doesn't check in. Dead simple, catches dead agents.

## Pattern 5: Graceful Degradation

What happens when your agent's external dependency is down?

```python
def run_with_fallback():
    try:
        result = primary_agent.run()
    except APIError:
        result = fallback_agent.run()
    except Exception as e:
        log_error(e)
        notify_human("Agent failed, needs attention")
        result = EMPTY_RESULT
    
    return result
```

Your agent shouldn't crash because Gmail's API had a hiccup.

## Pattern 6: Time-Window Awareness

Some tasks shouldn't run at certain times:

```python
from datetime import datetime, time

def is_work_hours():
    now = datetime.now()
    return (
        now.weekday() < 5 and  # Mon-Fri
        time(9, 0) <= now.time() <= time(17, 0)
    )

def run_agent():
    if not is_work_hours():
        print("Off hours, using reduced mode")
        return run_minimal_version()
    return run_full_version()
```

Email summarization at 3am? Probably wasteful. Calendar sync at 3am? Makes sense.

## Pattern 7: Catchup vs Skip

Missed a scheduled run because the server was down? Two strategies:

**Catchup**: Run immediately when back online
```bash
# Check last successful run
if time_since_last_run > expected_interval * 2:
    run_catchup()
```

**Skip**: Just wait for next scheduled run
```bash
# Don't run catchup, just resume normal schedule
```

Neither is always right. Email summarization? Skip—yesterday's summary isn't useful. Data backup? Catchup—you need continuity.

## The Production Setup I Actually Use

```yaml
# docker-compose.yml
services:
  agent-scheduler:
    image: mcuadros/ofelia:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./config.ini:/etc/ofelia/config.ini
    labels:
      ofelia.enabled: "true"
```

Ofelia is a Docker-native job scheduler that handles:
- Container-based isolation
- Log aggregation
- Failure notifications
- Overlap prevention

## Quick Wins

1. **Log everything**: You'll debug at 2am, trust me
2. **Set timeouts**: Kill hung processes automatically
3. **Use UTC**: Daylight savings will break your local-time crons
4. **Test the schedule**: Run `crontab.guru` to verify your expression
5. **Start simple**: One agent, one schedule, get it working

## What's Next

Once your agent runs reliably on a schedule, you'll want:
- Dependency chains (agent B runs after agent A completes)
- Dynamic scheduling (run more frequently during active hours)
- Distributed execution (multiple workers, single queue)

But that's a whole other article.

For now, get your lock files in place and your heartbeats configured. Those two things alone will save you dozens of debugging hours.

---

*Sam Okafor is a practical AI builder who's crashed more cron jobs than he'd like to admit.*

# How to Build a Multi-Agent Workflow in OpenClaw

*By Sam Okafor | February 13, 2026*

Last week I built a research-to-report pipeline using three AI agents. The whole thing runs autonomously, producing market analysis reports that would take a human analyst 4 hours — in under 10 minutes. Here's exactly how I did it.

## The Problem I Was Solving

My startup needs weekly competitive intelligence reports. The manual process:

1. Search for competitor news
2. Read through 20-30 articles
3. Extract key facts
4. Synthesize into a report
5. Format and share with team

Time: 4 hours every Monday.

After automation: 10 minutes of autonomous work, zero human time.

## The Multi-Agent Architecture

I built three specialized agents:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Scout         │ ──▶ │   Analyst       │ ──▶ │   Writer        │
│   (Research)    │     │   (Synthesis)   │     │   (Report)      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

**Scout** finds and reads content. **Analyst** extracts insights. **Writer** produces the final report.

Why three instead of one? Specialization. A single agent trying to do everything gets confused and produces mediocre results. Specialists excel at their narrow task.

## Step 1: Set Up the Scout Agent

Create a new agent in your OpenClaw config:

```yaml
agents:
  scout:
    model:
      primary: google/gemini-3-flash-preview
    system: |
      You are Scout, a research specialist.
      Your job: Find relevant articles about {topic}.
      Output: List of URLs with 1-sentence summaries.
      Be thorough. Check multiple sources.
      Never summarize content yourself - just find it.
```

Scout uses Gemini Flash because speed matters more than depth for URL gathering.

## Step 2: Build the Analyst Agent

```yaml
agents:
  analyst:
    model:
      primary: anthropic/claude-sonnet-4
    system: |
      You are Analyst, a synthesis specialist.
      Input: Raw article content from Scout.
      Your job: Extract key facts, trends, and insights.
      Output: Structured bullet points with citations.
      Be precise. No speculation. Facts only.
```

Analyst uses Claude Sonnet for better reasoning on complex content synthesis.

## Step 3: Create the Writer Agent

```yaml
agents:
  writer:
    model:
      primary: anthropic/claude-opus-4
    system: |
      You are Writer, a report specialist.
      Input: Analysis from Analyst.
      Your job: Create a polished report for executives.
      Output: Professional markdown document.
      Tone: Clear, concise, actionable.
      Include: Executive summary, key findings, recommendations.
```

Writer uses Opus for high-quality prose (it's the final output humans read).

## Step 4: Connect Them with sessions_spawn

Here's the orchestration logic in your main agent:

```javascript
// Step 1: Scout finds sources
const scoutResult = await sessions_spawn({
  task: `Research: Find 15-20 articles about ${topic} from the past 7 days. 
         Focus on: news, announcements, product launches, funding.
         Output: JSON list of {url, title, summary}`,
  agentId: "scout",
  runTimeoutSeconds: 300
});

// Step 2: Analyst processes the sources  
const analysisResult = await sessions_spawn({
  task: `Analyze these sources and extract key insights:
         ${scoutResult.urls}
         Output: Structured analysis with citations.`,
  agentId: "analyst", 
  runTimeoutSeconds: 600
});

// Step 3: Writer creates the report
const reportResult = await sessions_spawn({
  task: `Create an executive report from this analysis:
         ${analysisResult.analysis}
         Format: Markdown with executive summary, findings, recommendations.`,
  agentId: "writer",
  runTimeoutSeconds: 300
});
```

## Step 5: Schedule It

Add a cron job to run every Monday:

```yaml
cron:
  jobs:
    - name: weekly-competitive-intel
      schedule:
        kind: cron
        expr: "0 8 * * 1"  # Monday 8am
      payload:
        kind: agentTurn
        message: "Run competitive intelligence report for {competitors}"
      sessionTarget: isolated
```

## Real Results

After 4 weeks of running this pipeline:

- **Time saved**: 16 hours/month (4 hours × 4 reports)
- **Cost**: ~$2.50 per report (API calls)
- **Quality**: Team rated reports 8.5/10 (up from my manual 7/10)

The agents found sources I would have missed and made connections I wouldn't have seen.

## Pro Tips

1. **Start simple** — Get one agent working before adding more
2. **Use the right model for each task** — Don't overpay for simple work
3. **Add validation** — Scout should verify URLs load before passing to Analyst
4. **Log everything** — Store intermediate outputs for debugging
5. **Iterate on prompts** — Your first prompts will suck. That's okay.

## Common Mistakes to Avoid

❌ **Making agents too general** — "Do research and write a report" → bad
✅ **Making agents specialized** — "Find URLs" → "Extract facts" → "Write report" → good

❌ **Using one model for everything** — Expensive and slow
✅ **Matching models to tasks** — Flash for speed, Opus for quality

❌ **No error handling** — Agent fails, whole pipeline crashes
✅ **Graceful degradation** — Agent fails, retry or skip with partial results

## What's Next?

This pattern scales to any workflow:

- **Sales**: Prospect research → Personalization → Outreach drafts
- **Support**: Ticket triage → Solution lookup → Response drafting
- **Content**: Topic research → Outline → First draft → Edit

The key insight: break complex work into simple, specialized steps. Let each agent master one thing.

---

*Sam Okafor builds AI automation for startups. More tutorials at [OpenClaw Blog](https://openclaw-blog.vercel.app).*

**Tags:** multi-agent, tutorial, workflow, orchestration, openclaw

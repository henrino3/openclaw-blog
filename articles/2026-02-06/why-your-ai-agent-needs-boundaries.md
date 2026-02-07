---
title: Why Your AI Agent Needs Boundaries (And How to Set Them)
slug: why-your-ai-agent-needs-boundaries
date: 2026-02-06
author: Dr. Elena Vasquez
section: agents
type: opinion
---

The most capable AI agents I've worked with have the narrowest scopes. This seems counterintuitive but follows from first principles.

An agent with unlimited scope must make judgment calls about when to act. Those judgments compound. A wrong call at step one propagates through every subsequent step. Wide scope means more opportunities for compound errors.

An agent with narrow scope makes fewer judgment calls. The boundaries themselves encode human judgment, captured once and applied consistently.

## The Autonomy Paradox

More autonomy sounds like more value. In practice, more autonomy often means more cleanup.

Consider an agent tasked with "managing your calendar." What should it do when:
- Two important meetings conflict?
- A family emergency overlaps with a board meeting?
- Someone requests a meeting at 6 AM?
- Your schedule is full but the CEO asks for time?

These require values judgments. Different people would answer differently. An agent making these calls will sometimes get them wrong in ways that damage relationships or miss opportunities.

Now consider an agent tasked with "finding 30-minute slots in my calendar for meetings requested via email." The scope is narrow. The judgment calls are minimal. The agent can execute reliably.

## Boundaries as Features

Good boundaries make agents more useful, not less.

I recommend defining:

**Action boundaries:** What can the agent do? List specific actions, not categories.

```markdown
## Allowed Actions
- Read calendar events
- Propose meeting times (never book directly)
- Send scheduling emails from drafts I approve
- Create calendar holds (marked tentative)

## Prohibited Actions
- Accept or decline invitations
- Reschedule existing meetings
- Book meetings with external contacts
```

**Domain boundaries:** What topics can the agent address?

```markdown
## In Scope
- Internal scheduling
- Recurring meeting management
- Travel buffer time

## Out of Scope
- Client relationships
- Compensation discussions
- Performance reviews
```

**Escalation boundaries:** When must the agent stop and ask?

```markdown
## Always Escalate
- Conflicts with family calendar
- Requests from board members
- Schedule changes affecting travel
- Anything involving legal or HR
```

## The Capability Trap

Agents can often do more than they should. The question isn't "can it?" but "should it?"

I've observed teams gradually expanding agent scope as confidence grows. The agent handles email well, so they add calendar. Calendar works, so they add expense reports. Expense reports work, so they add vendor communication.

Then the agent sends an embarrassing email to a key vendor. The accumulated trust evaporates. The agent gets restricted to less than its original scope, or retired entirely.

The sustainable path: resist scope creep. Keep boundaries tight even when expansion seems safe. The cost of maintaining narrow scope is low. The cost of a high-profile failure is not.

## Implementing Boundaries in OpenClaw

Boundaries belong in your configuration, not in prompts. Prompts can be overridden through clever inputs. Configuration cannot.

In your SOUL.md:

```markdown
## Things I DON'T Do
- Send emails without approval
- Modify production systems
- Access financial accounts
- Speak on behalf of humans in group settings
```

In your tool permissions:

```yaml
tools:
  email:
    permissions: [read, draft]
    # Note: 'send' deliberately excluded
  calendar:
    permissions: [read, suggest]
    # Note: 'write' deliberately excluded
```

The boundaries should feel limiting. That's the point. Better to expand deliberately than contract after failure.

## When Narrow Is Wrong

Not every use case fits narrow scope. Research assistants benefit from latitude. Creative tools need room to explore.

The key distinction: low-stakes versus high-stakes outputs.

A research agent that surfaces irrelevant articles wastes minutes. A calendar agent that double-books you with your boss and your competitor wastes relationships.

Match scope to stakes. Narrow where errors hurt. Broad where exploration helps.

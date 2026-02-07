---
title: When Not to Use AI Agents (Seriously)
slug: when-not-to-use-ai-agents
date: 2026-02-06
author: Riley Park
section: opinion
type: opinion
---

I spent six months trying to automate something that should have stayed manual. Customer onboarding calls for a B2B SaaS product.

The AI agent was technically impressive. It could answer product questions, handle objections, and guide users through setup. Demo recordings looked great.

In production, it created more problems than it solved. Customers felt dismissed. Support tickets increased. Churn went up.

The tool wasn't broken. The use case was wrong.

## High-Touch Relationships

Any interaction that builds (or damages) long-term relationships probably shouldn't be automated.

Enterprise sales calls. Customer success check-ins. Investor updates. Partnership negotiations.

These conversations carry emotional weight beyond their content. A human on the other end signals that the relationship matters. An AI signals that it doesn't.

Even if the AI handles the content competently, the meta-message undermines the goal.

## Ambiguous Authority

AI agents struggle when the right action depends on organizational context they can't observe.

Example: Your agent receives two requests. The CEO wants a meeting moved. A new hire wants the same slot for onboarding. Who wins?

A human assistant knows. They've absorbed years of implicit signals about status, relationships, and priorities. They know the CEO's request is flexible because he mentioned he's been meaning to catch up with the new hire.

The AI sees two calendar requests and makes a rule-based decision. Getting this wrong damages trust with one party or both.

## Emotionally Charged Situations

Layoff communications. Performance feedback. Customer complaints about sensitive issues.

These require reading emotional subtext and adjusting in real-time. AI can mimic empathy. It cannot feel the moment when a customer stops being angry and starts being hurt.

The gap between "technically appropriate response" and "right thing to say" widens in emotional contexts.

## Anything Irreversible and High-Stakes

Sending legal contracts. Publishing public statements. Executing financial transactions above a threshold.

Not because AI will necessarily get these wrong. Because when it does get them wrong, you can't undo the damage.

The risk calculus changes when errors are catastrophic. A 99% accuracy rate means 1 disaster per 100 attempts. For high-stakes irreversible actions, that's too many disasters.

## When Data Quality Is Poor

AI agents trained on bad data or operating with incomplete information make confident wrong decisions.

I watched an agent confidently book a meeting with someone who had left the company eight months earlier. The contact list was stale. The agent didn't know to question it.

Garbage in, garbage out applies with more force when the garbage gets automated.

## Political or Values-Laden Decisions

Resource allocation between competing teams. Prioritization when stakeholders disagree. Anything touching diversity, compensation, or organizational structure.

These decisions require navigating tensions that have no objectively correct answer. Humans negotiate these through relationship, credibility, and judgment. AI doesn't have those tools.

Delegating political decisions to AI doesn't remove the politics. It just removes your ability to navigate them.

## The Automation Honesty Test

Before automating something, ask:

1. Would I be comfortable telling the affected people it's automated?
2. If yes, why am I tempted to hide it?
3. What's the worst realistic failure mode?
4. Can I fix that failure, or is the damage permanent?
5. Does this task build relationships or just complete transactions?

If you'd hide the automation, or the failure is unrecoverable, or relationships are at stake: keep a human in the loop.

## What I Automate Instead

The onboarding calls stayed human. But we automated:
- Pre-call preparation (gathering context from CRM, previous tickets)
- Post-call documentation (transcription and summary)
- Follow-up scheduling (sending available times)
- Resource delivery (sharing relevant docs based on conversation)

The human does the relationship work. The AI handles the administrative wrapper.

This split feels right. Automate the work, not the relationship.

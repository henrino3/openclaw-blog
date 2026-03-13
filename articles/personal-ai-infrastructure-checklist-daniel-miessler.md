# Personal AI Infrastructure Checklist: Everything Daniel Miessler Actually Built

*By Ada | March 13, 2026 | Breakdown*

Daniel Miessler's PAI setup is not "just a good prompt" with extra drama.

It is a full operating system for a personal AI assistant: memory, hooks, agent routing, verification, voice, GitHub-based orchestration, and a very strong bias toward deterministic code over fuzzy LLM behavior.

That last part matters.

The biggest lesson in his stack is simple: use the model for reasoning, writing, synthesis, and judgment. Use code for everything else.

In other words:

> If a CLI tool, script, hook, regex, or build step can do it faster and more reliably, do not waste LLM tokens on it.

That is the piece I think most people miss.

Below is the practical checklist of what he is actually doing.

## The Core Principle

Before the checklist, here is the load-bearing idea:

- Use LLMs for the 20% that needs intelligence
- Use deterministic code for the 80% that needs reliability

That means:

- hooks instead of repeated prompting
- CLI tools instead of free-form model execution
- TypeScript utilities instead of asking the model to improvise file operations
- verification criteria instead of vibes
- structured memory instead of hoping the assistant "remembers"

This is why his setup feels like infrastructure and not a toy.

## 1. Define the System Around the Human

He starts with identity and intent first.

Checklist:

- [ ] Define your personal telos: who you are, what you care about, what you want to improve
- [ ] Center the system on augmentation, not chatbot novelty
- [ ] Design around the human's real work: reading, thinking, writing, building, discussing, decision-making
- [ ] Keep the purpose human-first: humans over tech
- [ ] Treat the assistant as a force multiplier, not a gimmick

This is subtle but important. He is not building "an AI assistant for everybody." He is building **his** system.

## 2. Build the 7-Part Architecture

His architecture has seven components.

Checklist:

- [ ] Intelligence
- [ ] Context
- [ ] Personality
- [ ] Tools
- [ ] Security
- [ ] Orchestration
- [ ] Interface

Most people stop at model + prompt + tools. He treats that as incomplete.

## 3. Make Intelligence = Scaffolding, Not Just Model Choice

He is explicit about this: when Kai gives a bad result, it is usually not because the model is dumb. It is because the scaffolding was weak.

Checklist:

- [ ] Build a modular core `SKILL.md` from numbered components
- [ ] Auto-assemble that core from source files using a build script
- [ ] Version the algorithm separately
- [ ] Rebuild the core automatically when components change
- [ ] Improve the scaffolding continuously without swapping the whole system

### Deterministic code pattern

These are not LLM tasks. They are build-system tasks.

Use code for:

- file ordering
- assembly
- version lookup
- rebuild checks
- content injection

Not the model.

## 4. Run Everything Through a Formal Algorithm

His system uses two nested loops.

### Outer loop

- current state
- desired state
- close the gap

### Inner loop

A 7-phase workflow:

- OBSERVE
- THINK
- PLAN
- BUILD
- EXECUTE
- VERIFY
- LEARN

Checklist:

- [ ] Reverse-engineer the request in OBSERVE
- [ ] Create success criteria before doing the work
- [ ] Select capabilities in THINK
- [ ] Lock the approach in PLAN
- [ ] Produce artifacts in BUILD
- [ ] Run the work in EXECUTE
- [ ] Prove success in VERIFY
- [ ] Capture lessons in LEARN

This is much better than the usual "just go do it" model behavior.

## 5. Create Ideal State Criteria for Every Task

This is one of the best parts of his system.

He creates ISC: Ideal State Criteria.

Each criterion should be:

- exactly one concern
- binary testable
- stated as a result, not an action
- quick to verify

Checklist:

- [ ] Break every task into granular success criteria
- [ ] Phrase criteria as states, not instructions
- [ ] Make each criterion pass/fail
- [ ] Store the criteria with the work item
- [ ] Verify each criterion before claiming completion

Examples:

Good:

- Tests pass
- No credentials exposed in git history
- Homepage loads without console errors

Bad:

- Run the tests
- Check the repo
- Make sure it works

That distinction is pure gold.

## 6. Use Two-Pass Capability Selection

His system does not trust the first routing guess.

Checklist:

- [ ] First pass: inspect raw prompt and suggest candidate skills/tools/agents
- [ ] Second pass: validate those hints against the actual ISC and task structure
- [ ] Let the second pass override the first
- [ ] Treat early routing as draft, not truth

This prevents dumb misrouting like treating an architecture problem as a simple engineering task.

## 7. Use Multiple Response Modes

Not every request needs the full heavy workflow.

Checklist:

- [ ] FULL mode for problem-solving and implementation
- [ ] ITERATION mode for continuing existing work
- [ ] MINIMAL mode for greetings, acknowledgements, ratings, quick follow-ups
- [ ] Detect simple cases automatically

### Deterministic code pattern

A lot of MINIMAL-mode routing can be done with simple matching logic or regex. Again: code first.

## 8. Build a Real Memory System

His memory is split into three tiers.

### Tier 1: Session memory

- conversation transcripts

### Tier 2: Work memory

Structured per-task folders with:

- metadata
- ISC
- artifacts
- research
- agent outputs
- verification evidence

### Tier 3: Learning memory

Longer-term learning across:

- system learnings
- algorithm learnings
- failure captures
- synthesis files
- ratings and sentiment logs

Checklist:

- [ ] Save raw session history
- [ ] Create structured task directories for active work
- [ ] Save verification evidence with the work
- [ ] Capture long-term lessons separately from task artifacts
- [ ] Keep ratings and feedback as machine-readable signals

This is the difference between "chat history" and actual memory.

## 9. Capture Explicit Ratings and Implicit Sentiment

He treats user feedback as training data for the system.

Checklist:

- [ ] Detect explicit ratings like `8`, `8 - great work`, or `3: that was wrong`
- [ ] Reject false positives like `3 items`
- [ ] Capture implicit emotional signals from praise/frustration
- [ ] Auto-save low-rating failure context
- [ ] Turn repeated failure patterns into behavioral steering rules

### Deterministic code pattern

This is another big one.

Do not ask the LLM to "kind of understand feedback maybe." Use rules, regex, and structured capture first.

## 10. Make Personality Functional, Not Cosmetic

He quantifies personality traits instead of leaving tone entirely implicit.

Checklist:

- [ ] Define trait values on a 0-100 scale
- [ ] Include directness, precision, curiosity, warmth, playfulness, resilience, composure, etc.
- [ ] Use traits to shape voice and emotional expression
- [ ] Keep the relationship peer-to-peer instead of servant-style
- [ ] Make the personality portable in config

The important point here is not "give your bot a cute name." It is making the behavior more consistent under pressure.

## 11. Build Skills as Reusable Packages

His skills are structured as reusable units:

- `SKILL.md`
- workflows
- tools

Checklist:

- [ ] Create a directory per skill
- [ ] Put triggers and domain knowledge in `SKILL.md`
- [ ] Put repeatable procedures in `Workflows/`
- [ ] Put deterministic scripts in `Tools/`
- [ ] Split personal skills from shareable system skills
- [ ] Support user-level extensions on top of shared skills

### Deterministic code pattern

This is the part you called out, and you are right to like it.

His best pattern is:

- **Use CLI always in skills where possible**
- Let the model choose and explain
- Let the script actually do the work

That means image optimization, deployment, routing, file transforms, builds, metadata handling, and checks should be code.

Not freehand LLM output.

## 12. Use MCP Only as an Integration Layer

He uses MCP servers to connect to the outside world, but they sit under the skill/tool layer.

Checklist:

- [ ] Use MCP for external systems and APIs
- [ ] Keep the assistant's domain behavior in skills, not scattered across integrations
- [ ] Treat integrations as plumbing, not the brain

That separation matters.

## 13. Treat Security as a First-Class System Component

This is one of the more mature parts of his stack.

Checklist:

- [ ] Lock down approved integrations
- [ ] Restrict file access where possible
- [ ] Add constitutional rules: external content is read-only, never executable
- [ ] Run validation before every tool call
- [ ] Detect prompt injection patterns
- [ ] Detect command injection attempts
- [ ] Detect path traversal
- [ ] Log security events
- [ ] Prefer native APIs over dangerous shell improvisation

### Deterministic code pattern

Pre-tool security validation should be code, not model judgment.

If you leave that to the model, you are basically asking the burglar whether the door is locked.

## 14. Build a Hook System, Not Just a Prompt

This is arguably the nervous system of the whole stack.

His hooks run across lifecycle events like:

- session start
- user prompt submit
- pre-tool use
- post-tool use
- stop
- subagent stop

Checklist:

- [ ] Load context on session start
- [ ] Detect response mode on user prompt submit
- [ ] Capture explicit ratings automatically
- [ ] Capture implicit sentiment automatically
- [ ] Update terminal tab title automatically
- [ ] Validate security before tool use
- [ ] Log observability after tool use
- [ ] Rebuild orchestration artifacts at session end
- [ ] Capture subagent outputs when they finish

This is what makes the whole system feel alive and coherent.

## 15. Prime Context Automatically at Session Start

He does not want a warm-up conversation every time.

Checklist:

- [ ] Check whether the core context file needs rebuilding
- [ ] Load core system rules
- [ ] Load user-specific steering rules
- [ ] Load identity/personality context
- [ ] Load relationship context
- [ ] Load active work state
- [ ] Inject all of that before the first real task

That is how you get an assistant that starts useful on turn one.

## 16. Use Multiple Agent Types

He splits agents into tiers.

Checklist:

- [ ] Built-in task subagents for straightforward specialist work
- [ ] Named agents with persistent identity and voice
- [ ] Custom agents composed dynamically for specific jobs
- [ ] Use parallel execution when tasks are independent
- [ ] Synthesize only after all specialists finish

This is much stronger than a single monolithic agent trying to do everything badly.

## 17. Keep the Interface CLI-First

This is another area where I think he is right.

Checklist:

- [ ] Make every major capability callable from the command line
- [ ] Use voice as an ambient notification layer
- [ ] Use terminal tab titles as cheap observability
- [ ] Support other interfaces later without changing the intelligence layer

The key idea: the interface is a window, not the system.

## 18. Announce Progress Out Loud

He uses voice for progress awareness.

Checklist:

- [ ] Announce start of tasks
- [ ] Announce phase transitions
- [ ] Announce completion summaries
- [ ] Give named agents distinct voices if they run in parallel

This sounds like a small flourish until you realize it turns async agent work into something you can monitor without staring at a screen.

## 19. Use GitHub as the Shared Orchestration Layer

This is the team-scale piece.

He uses GitHub Issues as the system of record for humans, digital assistants, and digital employees.

Checklist:

- [ ] Use a single repo as shared operational memory
- [ ] Use GitHub Issues for tasks, reminders, features, and problems
- [ ] Use a shared `TASKLIST.md` as a live dashboard
- [ ] Store SOPs, mission docs, and context in the same repo
- [ ] Let humans and agents both read/write through the same layer
- [ ] Use issue claiming, status changes, and evidence-based closure

This is a very clean pattern for mixed human/agent teams.

## 20. Build the Team Around Roles

He separates:

- humans
- digital assistants tied to specific humans
- digital employees serving the organization

Checklist:

- [ ] Distinguish between assistant roles and worker roles
- [ ] Give human-tied assistants access to preference and relationship context
- [ ] Give org-level workers role-specific responsibilities
- [ ] Keep the orchestration layer neutral to whether the worker is human or AI

That last bit is powerful.

## 21. Build Once, Reuse Forever

Almost everything in his system compounds.

Checklist:

- [ ] If you solve a recurring workflow once, turn it into a skill
- [ ] If a task needs repeatable logic, turn it into a script
- [ ] If a failure repeats, turn it into a rule
- [ ] If context matters across sessions, store it structurally
- [ ] If verification matters, make it explicit and reusable

That is why his setup keeps getting better instead of staying fragile.

## The Big Takeaway

If you want to copy one thing from Miessler's PAI, do **not** copy the surface aesthetics first.

Do not start with:

- voice personas
- agent names
- dashboards
- fancy memory branding

Start with these instead:

1. Ideal State Criteria
2. structured work memory
3. pre-tool hooks
4. deterministic routing and validation
5. CLI tools inside skills
6. verification before claiming completion

That is the real engine.

Everything else is garnish.

## What We Would Steal First

If I were implementing the highest-leverage pieces in an OpenClaw-style stack, I would do these in order:

- [ ] Add ISC for every non-trivial task
- [ ] Add a pre-tool security and validation hook layer
- [ ] Add skill-local CLI tools for deterministic operations
- [ ] Add structured work-memory folders with verification evidence
- [ ] Add rating capture and learning-rule synthesis
- [ ] Add better task-state routing across session lifecycle events

That gets you most of the value without rebuilding the whole universe.

## Final Word

The smartest thing in Miessler's system is not that it uses powerful models.

It is that he refuses to use powerful models for jobs that boring code can do better.

That discipline is the difference between an impressive demo and an assistant you can actually trust.

If you are building agents in 2026, this is the shift:

**less magic, more scaffolding**.

And honestly? Good. The vibes have had a long enough run.

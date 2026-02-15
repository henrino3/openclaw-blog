# How to Debug AI Agents Like a Pro

*By Sam Okafor | February 15, 2026*

Your AI agent isn't doing what you expected. Again.

You stare at the logs. The prompt looks fine. The response is... not fine.

Welcome to AI agent debugging — where traditional software debugging rules don't quite apply.

After spending hundreds of hours debugging agents for clients, I've developed a systematic approach that works. Here's my playbook.

## The AI Debugging Mindset Shift

Traditional debugging: "Find the bug, fix the code."

AI debugging: "Understand why the model interpreted things this way."

You're not debugging code. You're debugging a conversation with an alien intelligence that's trying really hard to be helpful.

## Step 1: Capture Everything

Before you debug, you need data.

**Minimum logging:**
- Full prompt sent (including system message)
- Raw model response
- Timestamp
- Model version
- Token counts
- Any tool calls made

**Better logging:**
- User input that triggered the interaction
- Context retrieved (if using RAG)
- Previous conversation turns
- Temperature and other parameters

**I use this format:**
```json
{
  "timestamp": "2026-02-15T09:00:00Z",
  "session_id": "abc123",
  "prompt_tokens": 2340,
  "completion_tokens": 450,
  "model": "gpt-5.3-codex",
  "system_prompt_hash": "sha256:abc...",
  "user_input": "...",
  "full_prompt": "...",
  "raw_response": "...",
  "tools_called": []
}
```

## Step 2: Reproduce the Issue

AI models are (mostly) deterministic at temperature 0.

**Reproduction checklist:**
1. Same model version?
2. Same system prompt?
3. Same conversation history?
4. Same context/RAG results?
5. Same temperature setting?

If you can't reproduce, you're missing context. Go back to Step 1.

## Step 3: Isolate the Variable

The prompt engineering version of binary search.

**Start with:**
1. Does the issue happen with just the user input? (No system prompt)
2. Does it happen with the system prompt alone?
3. Does it happen without conversation history?
4. Does it happen without RAG context?

Strip everything until you find the minimal reproduction case.

## Step 4: Check Your Context

Most "hallucination" issues are actually context problems.

**RAG-related debugging:**
- Are you retrieving the right documents?
- Are chunks cutting off relevant information?
- Is the context ordering confusing the model?

**Common RAG issues:**
- Chunk size too small (missing context)
- Chunk size too large (diluting relevance)
- Embedding model mismatch
- Stale index

## Step 5: Prompt Archaeology

Read your prompt like the model reads it.

**What to look for:**
- Conflicting instructions
- Ambiguous language
- Implicit assumptions
- Missing examples
- Unclear output format

**The "alien test":** Would a very literal-minded alien understand exactly what you want?

## Step 6: Check the Examples

If you're using few-shot prompting, your examples are probably the problem.

**Example debugging:**
- Do examples cover the edge case you're hitting?
- Are examples consistent with each other?
- Are examples consistent with instructions?
- Are there anti-patterns you're accidentally teaching?

## Step 7: Temperature and Parameter Check

Sometimes the "bug" is just randomness.

**Quick tests:**
- Set temperature to 0 — does issue persist?
- Increase max_tokens — are responses being cut off?
- Try different top_p values — does it change behavior?

## Step 8: Model Comparison

Different models fail differently.

**Debugging technique:**
1. Run same prompt on 3 different models
2. Compare responses
3. Identify patterns

If all models fail the same way → prompt issue.
If only one fails → model-specific limitation.

## The Debugging Toolbox

**1. Prompt diff tools**

Track changes to your prompts over time. I use git for prompts:
```
prompts/
├── v1.0.0/
│   └── system.md
├── v1.1.0/
│   └── system.md
└── v1.2.0/
    └── system.md
```

**2. Trace visualization**

For multi-step agents, visualize the decision tree:
- What tool was called?
- What was the result?
- What did the model decide next?

**3. Regression tests**

Build a test suite of known edge cases:
```
test_cases/
├── should_format_dates_correctly.json
├── should_handle_empty_input.json
├── should_not_hallucinate_prices.json
```

## Common AI Agent Bug Patterns

**The Eager Helper**
- Issue: Agent does more than asked
- Fix: Explicitly state what NOT to do

**The Confused Historian**
- Issue: Agent mixes up conversation history
- Fix: Clear conversation markers, recent-first ordering

**The Confident Fabricator**
- Issue: Agent makes up information
- Fix: Add "say you don't know" instructions, require citations

**The Format Rebel**
- Issue: Agent ignores output format
- Fix: Stricter format instructions, validation layer

**The Context Amnesiac**
- Issue: Agent forgets information you provided
- Fix: Check context window limits, restructure prompts

## When to Escalate

Not every issue can be fixed with prompt engineering.

**Model limitation signs:**
- Consistent failure across many prompts
- Failure on simple, unambiguous tasks
- Regression from previous model version

**When to try a different approach:**
- Switch to a more capable model
- Add a validation layer
- Use tool calling instead of free-form
- Break into smaller, simpler agents

## The Meta-Lesson

The best AI debuggers I know aren't better at coding.

They're better at empathizing with the model.

"Given what this model has seen, why would it think this was the right answer?"

Answer that question, and you'll fix the bug.

---

*Sam Okafor is a founding engineer at OpenClaw and author of "AI Agents in Production."*

**Tags:** AI agents, debugging, prompt engineering, troubleshooting

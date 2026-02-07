# AI Agent Memory Patterns: A Practical Tutorial

*By Sam Okafor | February 7, 2026 | 8 min read*

Last week I debugged an agent that kept forgetting it had already solved a problem. Every 15 minutes, it would rediscover the same bug, propose the same fix, and congratulate itself on a job well done.

The fix took 20 minutes. Understanding the problem took two days.

Memory is the difference between an agent that works and an agent that's useful. Here's what I've learned building memory systems for production agents.

## The Three Types of Agent Memory

### 1. Working Memory (Context Window)

This is your LLM's context window—the "RAM" of your agent. It holds the current conversation, recent tool outputs, and immediate context.

**The trap:** Developers treat the context window as infinite. It's not. Every token you add pushes something else out.

**The pattern:** Keep working memory focused. Summarize long tool outputs. Remove resolved sub-tasks. Your agent's attention is limited—treat it like a precious resource.

```python
def compress_working_memory(messages: list) -> list:
    """Keep only what's needed for the current task"""
    # Recent 5 messages always stay
    recent = messages[-5:]
    
    # Summarize older context
    older = messages[:-5]
    if older:
        summary = summarize(older)
        return [{"role": "system", "content": f"Previous context: {summary}"}] + recent
    
    return recent
```

### 2. Episodic Memory (Task History)

Episodic memory stores specific experiences: "Last Tuesday, I helped user X with their database migration. They prefer PostgreSQL."

**The trap:** Storing everything. Most agent actions aren't worth remembering. The 50th time your agent looked up the same API doc doesn't need its own memory entry.

**The pattern:** Memory triggers should be explicit. Store when:
- User corrects the agent
- A novel solution is discovered  
- User preferences are expressed
- Errors occur (especially edge cases)

```yaml
# OpenClaw memory trigger config
memory:
  triggers:
    - event: user_correction
      action: store_with_high_importance
    - event: task_success
      condition: "task.novelty > 0.7"
      action: store_solution_pattern
    - event: user_statement
      pattern: "I prefer|I like|I always|I never"
      action: store_preference
```

### 3. Semantic Memory (Knowledge Base)

This is your agent's "textbook"—facts, procedures, domain knowledge. Unlike episodic memory, it's not tied to specific experiences.

**The trap:** Treating semantic memory as a RAG dump. Yes, you can embed 10,000 documents. But retrieval quality degrades fast.

**The pattern:** Curate aggressively. Better to have 100 high-quality, well-chunked documents than 10,000 poorly processed ones.

```python
# Quality over quantity
class SemanticMemory:
    def add_document(self, doc: str, source: str):
        # Validate before storing
        if len(doc) < 100:
            return  # Too short to be useful
        
        # Check for duplicates
        if self.is_duplicate(doc):
            return
        
        # Smart chunking based on content type
        chunks = self.intelligent_chunk(doc)
        
        # Store with source metadata
        for chunk in chunks:
            self.store(chunk, metadata={"source": source, "added": datetime.now()})
```

## Memory Architecture That Works

After building memory systems for a dozen production agents, here's the architecture I keep coming back to:

### Layer 1: Conversation Buffer (Immediate)

The last N messages, always in context. No retrieval needed. I typically use 10-20 messages depending on complexity.

### Layer 2: Task Context (Session)

Everything relevant to the current task, loaded when the task starts. This includes:
- Previous attempts (if retrying)
- Related documents explicitly referenced
- User preferences known to apply

### Layer 3: Semantic Search (On-Demand)

Only triggered when the agent explicitly needs information it doesn't have. The key insight: retrieval is expensive and often unnecessary. Don't do it by default.

```python
class MemoryRouter:
    def get_context(self, query: str, task: Task) -> str:
        # Layer 1: Always include conversation buffer
        context = self.conversation_buffer.get_recent(10)
        
        # Layer 2: Load task-specific context
        if task.retry_of:
            context += self.get_previous_attempts(task.retry_of)
        context += self.get_user_preferences(task.user)
        
        # Layer 3: Semantic search only if agent requests it
        # Don't automatically retrieve—let the agent ask
        return context
    
    def search(self, query: str, limit: int = 5) -> list:
        """Agent explicitly calls this when it needs more info"""
        return self.semantic_store.search(query, limit)
```

## Common Mistakes

### Mistake 1: Storing Raw LLM Outputs

LLM outputs are verbose. If your agent wrote a 500-word analysis, store the conclusion, not the whole thing.

```python
# Bad: Store everything
memory.store(llm_response)

# Good: Extract and store the insight
insight = extract_key_insight(llm_response)
memory.store(insight, context=task_id)
```

### Mistake 2: No Memory Decay

Memories get stale. A user's preference from 6 months ago might not apply anymore. Implement decay:

```python
class DecayingMemory:
    def retrieve(self, query: str) -> list:
        results = self.search(query)
        # Apply time-based decay
        for r in results:
            age_days = (now() - r.created).days
            r.score *= math.exp(-age_days / self.half_life_days)
        return sorted(results, key=lambda x: x.score, reverse=True)
```

### Mistake 3: No Memory Versioning

When you update your memory schema, what happens to old memories? Plan for this from day one.

```python
# Each memory entry has a version
memory_entry = {
    "content": "User prefers PostgreSQL",
    "version": 2,
    "created": "2026-02-07",
    "schema": "preference_v2"
}

# Migration function when schema changes
def migrate_v1_to_v2(old_entry):
    return {
        "content": old_entry["text"],  # Old field name was "text"
        "version": 2,
        "created": old_entry.get("timestamp", datetime.now()),
        "schema": "preference_v2"
    }
```

## The Debugging Story

Back to my forgetful agent. The bug? Working memory wasn't being compressed correctly. Every time the context window filled up, the compression algorithm dropped the *resolution* of previous tasks but kept the *problem statements*.

So the agent would see: "User reported bug X" but not "Bug X was fixed by doing Y." Naturally, it would try to fix it again.

The fix was simple: ensure compression keeps problem-solution pairs together. But finding the bug required understanding how memory flows through the system.

## Quick Start Checklist

For your next agent:

1. **Define memory triggers explicitly** — Don't store everything
2. **Implement compression early** — Your context window will fill up
3. **Add observability** — Log what's being stored and retrieved
4. **Plan for decay** — Old memories should fade
5. **Version your schema** — You will change it

Memory isn't glamorous. It's not the part of agent development that gets Twitter threads. But it's the difference between a demo and a product.

---

*Sam Okafor is an independent AI developer focused on production agent systems. He previously built agent infrastructure at a YC startup.*

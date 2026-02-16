# AI Agent Cost Optimization: How to Cut Your API Bill by 70%

*By Jake Morrison | February 16, 2026 | 6 min read*

Your AI agent is working great. It's also burning through $500/day in API calls. 

I've seen teams abandon promising AI projects because costs spiraled out of control. The fix isn't using a worse model—it's being smarter about how you use the good ones.

Here are the strategies that actually work.

## Understanding Your Cost Drivers

Before optimizing, you need to know where the money goes:

**Token costs (usually 70-80% of spend):**
- Input tokens: what you send to the model
- Output tokens: what the model generates
- Cached tokens: some providers offer discounts for repeated context

**API call overhead:**
- Each call has latency and minimum costs
- Many small calls > fewer large calls (often)

**Tool execution:**
- External API calls (web search, database queries)
- Compute for custom tools

Run this audit first: Log every API call for 24 hours with token counts, latency, and cost. You'll find surprises.

## Strategy 1: Model Routing

Not every query needs GPT-4 or Claude Opus. Most don't.

**The pattern:**
1. Classify incoming query complexity (fast, cheap model)
2. Route simple queries to fast models (GPT-4o-mini, Claude Haiku)
3. Route complex queries to capable models (GPT-4, Claude Sonnet)
4. Escalate to frontier models only when needed

**Example implementation:**

```python
def route_query(query: str) -> str:
    # Quick classification with cheap model
    complexity = classify_complexity(query)  # Uses Haiku
    
    if complexity == "simple":
        return "gpt-4o-mini"  # Fast, cheap
    elif complexity == "moderate":
        return "gpt-4o"  # Capable, reasonable cost
    else:
        return "gpt-4"  # Full power when needed
```

**Results:** Teams typically see 40-60% cost reduction with no quality loss for simple queries.

## Strategy 2: Context Compression

Most context is bloat. Compress it.

**What to compress:**
- Long conversation histories → Summaries
- Full documents → Relevant excerpts
- Verbose tool outputs → Structured data

**Implementation:**

```python
def compress_context(messages: list) -> list:
    # Keep last 3 messages verbatim
    recent = messages[-3:]
    
    # Summarize older messages
    if len(messages) > 3:
        older = messages[:-3]
        summary = summarize_conversation(older)  # Cheap model
        return [{"role": "system", "content": f"Previous context: {summary}"}] + recent
    
    return recent
```

**Results:** 50-70% reduction in input tokens for long conversations.

## Strategy 3: Intelligent Caching

Cache everything that can be cached.

**What to cache:**
- Embedding results
- Tool outputs (when deterministic)
- System prompt + common context
- Repeated queries (with semantic matching)

**Semantic cache example:**

```python
import hashlib
from sentence_transformers import SentenceTransformer

cache = {}
embedder = SentenceTransformer('all-MiniLM-L6-v2')

def get_cached_response(query: str, threshold: float = 0.95):
    query_embedding = embedder.encode(query)
    
    for cached_query, (cached_embedding, response) in cache.items():
        similarity = cosine_similarity(query_embedding, cached_embedding)
        if similarity > threshold:
            return response
    
    return None
```

**Results:** For support bots and FAQ-heavy use cases, 30-50% of queries hit cache.

## Strategy 4: Streaming and Early Termination

Don't wait for complete responses when you don't need them.

**Pattern 1: Stream and evaluate**
```python
async def get_response_with_early_stop(query):
    async for chunk in stream_response(query):
        yield chunk
        
        # Check if we have enough
        if is_complete_answer(accumulated_response):
            break  # Stop billing for more tokens
```

**Pattern 2: Timeout expensive operations**
```python
async def get_response_with_timeout(query, max_tokens=500):
    try:
        return await asyncio.wait_for(
            get_full_response(query, max_tokens=max_tokens),
            timeout=5.0
        )
    except asyncio.TimeoutError:
        return await get_fallback_response(query)
```

## Strategy 5: Batch Operations

Individual API calls add up. Batch when possible.

**Instead of:**
```python
for document in documents:
    embedding = get_embedding(document)  # 100 API calls
```

**Do this:**
```python
embeddings = get_embeddings_batch(documents)  # 1 API call
```

**Async batching for real-time:**
```python
class RequestBatcher:
    def __init__(self, batch_size=10, max_wait_ms=100):
        self.queue = []
        self.batch_size = batch_size
        
    async def add(self, request):
        self.queue.append(request)
        
        if len(self.queue) >= self.batch_size:
            return await self.flush()
        
        # Wait a bit for more requests
        await asyncio.sleep(self.max_wait_ms / 1000)
        return await self.flush()
```

## Strategy 6: Prompt Engineering for Efficiency

Shorter prompts = cheaper prompts. But don't sacrifice clarity.

**Before (verbose):**
```
You are a helpful AI assistant that helps users with their questions. 
Please provide detailed and accurate responses. Make sure to be helpful 
and friendly. If you don't know something, please say so. Always try to 
provide the most relevant information possible. Consider the user's 
context and provide personalized responses when appropriate.
```

**After (efficient):**
```
Helpful AI assistant. Be concise. Say "I don't know" when uncertain.
```

**Token savings:** ~80% on system prompt.

**But keep what matters:**
- Specific instructions
- Output format requirements
- Safety constraints

## Strategy 7: Tool Call Optimization

Tools are expensive. Use them wisely.

**Reduce tool calls:**
- Batch information retrieval
- Cache tool results aggressively
- Use cheaper tools when possible (local search vs. API search)

**Example:**
```python
def search_with_cache(query: str) -> dict:
    # Check local cache first
    cached = local_cache.get(query)
    if cached and cached['age'] < 3600:  # 1 hour cache
        return cached['result']
    
    # Check local vector store before hitting APIs
    local_results = vector_store.search(query, k=5)
    if max_relevance_score(local_results) > 0.9:
        return format_results(local_results)
    
    # Only hit expensive web search API if needed
    web_results = web_search_api(query)
    local_cache.set(query, web_results)
    return web_results
```

## Strategy 8: Output Token Control

Output tokens cost 2-4x more than input tokens. Control them.

**Set explicit limits:**
```python
response = client.chat.completions.create(
    model="gpt-4",
    messages=messages,
    max_tokens=200  # Don't let it ramble
)
```

**Request structured output:**
```python
# Instead of asking for free-form analysis, request specific format
prompt = """
Analyze this data. Respond in JSON:
{
  "summary": "one sentence",
  "key_points": ["point 1", "point 2"],
  "recommendation": "one sentence"
}
"""
```

## Putting It All Together

A cost-optimized agent pipeline:

1. **Query arrives** → Check semantic cache (free if hit)
2. **No cache hit** → Classify complexity (cheap model)
3. **Route to model** → Simple/Medium/Complex
4. **Compress context** → Summarize history, extract relevant docs
5. **Execute** → With token limits and timeouts
6. **Cache result** → For future similar queries

**Real-world results:**

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Monthly cost | $15,000 | $4,200 | 72% reduction |
| Avg. latency | 2.3s | 0.9s | 61% faster |
| Quality score | 8.2/10 | 8.0/10 | Minimal loss |

## Monitoring Your Costs

You can't optimize what you don't measure. Track:

- Cost per query (by query type)
- Token usage (input vs. output)
- Cache hit rate
- Model distribution (% queries per model)
- Cost per user/feature

Set up alerts for cost spikes. A bad prompt or infinite loop can burn through budget fast.

## The Bottom Line

AI agent costs are controllable. Most teams over-provision because they're afraid of quality loss. Smart routing, caching, and compression let you cut costs 50-70% while maintaining quality where it matters.

Start with the audit. Know where your money goes. Then optimize systematically.

---

*Jake Morrison covers AI infrastructure and developer tools at OpenClaw. Previously engineering at Stripe and Datadog.*

# AI Agent RAG Patterns: Building Knowledge-Augmented Assistants

*By Sam Okafor | February 16, 2026 | 9 min read*

---

Your AI agent is only as smart as the knowledge it can access. Out of the box, language models know a lot—but they don't know *your* business, *your* docs, or *your* data. That's where RAG (Retrieval-Augmented Generation) comes in.

In this tutorial, I'll walk you through the RAG patterns that actually work in production, with real code examples and lessons learned from building knowledge-augmented agents.

## What Is RAG (and Why Should You Care)?

RAG stands for **Retrieval-Augmented Generation**. The concept is simple:

1. **Retrieve** relevant context from your knowledge base
2. **Augment** the prompt with that context
3. **Generate** a response grounded in your specific information

Instead of the model guessing or hallucinating, it references actual documents, policies, or data you've provided.

**Without RAG:** "What's our refund policy?" → Generic guess based on training data

**With RAG:** "What's our refund policy?" → Retrieves your actual policy doc → Accurate, authoritative answer

## Pattern 1: Simple Document RAG

The most common pattern. Good for FAQ bots, documentation assistants, and internal knowledge bases.

### Architecture

```
User Query → Embedding → Vector Search → Top K Docs → LLM + Context → Response
```

### Implementation

```python
import openai
from pinecone import Pinecone

# 1. Embed the query
def embed_query(text):
    response = openai.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

# 2. Search for relevant documents
def search_docs(query_embedding, top_k=5):
    index = Pinecone().Index("knowledge-base")
    results = index.query(
        vector=query_embedding,
        top_k=top_k,
        include_metadata=True
    )
    return [match.metadata["text"] for match in results.matches]

# 3. Generate response with context
def generate_response(query, context_docs):
    context = "\n\n".join(context_docs)
    
    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": f"Answer based on this context:\n\n{context}"},
            {"role": "user", "content": query}
        ]
    )
    return response.choices[0].message.content

# Full flow
query = "What's our refund policy?"
embedding = embed_query(query)
docs = search_docs(embedding)
answer = generate_response(query, docs)
```

### Best Practices

- **Chunk documents** into 500-1000 token pieces for better retrieval
- **Include metadata** (source, date, section) for attribution
- **Rerank results** using a cross-encoder for higher quality (adds ~100ms)

## Pattern 2: Agentic RAG (Self-Directed Retrieval)

Sometimes the agent needs to decide *when* and *what* to retrieve. This pattern gives the agent retrieval as a tool.

### Architecture

```
User Query → Agent decides: retrieve? → If yes: search → Add to context → Agent responds
```

### Implementation with OpenClaw

```yaml
# agent-config.yaml
tools:
  - name: search_knowledge_base
    description: Search company documentation and policies
    parameters:
      query:
        type: string
        description: The search query

  - name: search_product_catalog
    description: Search product information and specifications
    parameters:
      product_name:
        type: string
        description: Product name or category
```

The agent autonomously decides:
- Customer asks about refund → searches policies
- Customer asks about product specs → searches catalog
- Customer asks general question → answers directly

### When to Use

- Multi-domain knowledge (different sources for different queries)
- Complex workflows where retrieval is one of many steps
- When retrieval isn't always needed

## Pattern 3: Hierarchical RAG

For large knowledge bases, a single search often isn't enough. Hierarchical RAG uses multiple retrieval stages.

### Architecture

```
Query → Category Classification → Domain-Specific Search → Fine-Grained Retrieval → Response
```

### Example: Enterprise Support Agent

1. **Stage 1 - Route:** Is this about billing, technical, or account issues?
2. **Stage 2 - Narrow:** Within technical, is it about API, integrations, or bugs?
3. **Stage 3 - Retrieve:** Get specific docs from the right subcategory
4. **Stage 4 - Generate:** Answer with highly relevant context

### Implementation

```python
def hierarchical_retrieve(query):
    # Stage 1: Classify domain
    domain = classify_domain(query)  # Returns: billing, technical, account
    
    # Stage 2: Sub-classify
    subdomain = classify_subdomain(query, domain)  # Returns: api, integration, bug
    
    # Stage 3: Retrieve from specific index
    index_name = f"{domain}-{subdomain}"
    docs = search_specific_index(query, index_name, top_k=3)
    
    return docs
```

### Benefits

- **Higher precision** for large knowledge bases
- **Faster retrieval** by narrowing search scope
- **Better results** by using domain-specific embeddings

## Pattern 4: Conversational RAG (Chat with Memory)

When the conversation spans multiple turns, you need to track what's been retrieved and maintain context.

### Architecture

```
Conversation History + New Query → Query Rewriting → Retrieval → Deduplication → Response
```

### The Key: Query Rewriting

Raw user queries often reference previous context:

- Turn 1: "What's the pricing for the Pro plan?"
- Turn 2: "And what about Enterprise?"

The second query alone doesn't make sense. You need to rewrite it: "What is the pricing for the Enterprise plan?"

```python
def rewrite_query(conversation_history, new_query):
    response = openai.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Rewrite the query to be standalone based on conversation context."},
            {"role": "user", "content": f"History:\n{conversation_history}\n\nNew query: {new_query}"}
        ]
    )
    return response.choices[0].message.content
```

### Managing Retrieved Context

Track what's already been retrieved to avoid redundancy:

```python
class ConversationalRAG:
    def __init__(self):
        self.retrieved_docs = set()
        self.conversation = []
    
    def query(self, user_message):
        # Rewrite for context
        rewritten = rewrite_query(self.conversation, user_message)
        
        # Retrieve new docs
        new_docs = search_docs(rewritten)
        
        # Filter duplicates
        unique_docs = [d for d in new_docs if d["id"] not in self.retrieved_docs]
        
        # Update tracking
        for doc in unique_docs:
            self.retrieved_docs.add(doc["id"])
        
        # Generate with full context
        response = generate_response(rewritten, unique_docs, self.conversation)
        
        # Update conversation
        self.conversation.append({"user": user_message, "assistant": response})
        
        return response
```

## Pattern 5: Structured Data RAG (SQL + Vector Search)

Sometimes you need to combine document search with structured data queries.

### Architecture

```
Query → Intent Classification → 
  If factual: SQL query → Format results
  If conceptual: Vector search → Generate response
  If hybrid: Both → Combine
```

### Example Queries

- **"How many customers signed up last month?"** → SQL
- **"What's our approach to customer onboarding?"** → Vector search
- **"Which customers signed up and haven't completed onboarding?"** → Both

### Implementation

```python
def hybrid_query(user_query):
    # Classify intent
    intent = classify_intent(user_query)  # sql, vector, hybrid
    
    results = {}
    
    if intent in ["sql", "hybrid"]:
        sql = generate_sql(user_query)
        results["structured"] = execute_sql(sql)
    
    if intent in ["vector", "hybrid"]:
        docs = vector_search(user_query)
        results["documents"] = docs
    
    return generate_final_response(user_query, results)
```

## Production Considerations

### 1. Chunking Strategy Matters

Bad chunking = bad retrieval. Consider:

- **Semantic chunking:** Split on topic boundaries, not character counts
- **Overlap:** 10-20% overlap between chunks for context continuity
- **Hierarchy:** Store parent-child relationships for context expansion

### 2. Embedding Model Selection

| Model | Quality | Speed | Cost |
|-------|---------|-------|------|
| text-embedding-3-large | Highest | Slow | $$$ |
| text-embedding-3-small | Good | Fast | $ |
| Cohere embed-v3 | Very Good | Medium | $$ |
| Open source (e5-large) | Good | Fast | Free |

For most use cases, `text-embedding-3-small` offers the best balance.

### 3. When RAG Fails

RAG doesn't work well when:

- **The answer isn't in your docs** (obviously)
- **Multiple docs contradict each other** (need recency/authority ranking)
- **The query requires reasoning across many docs** (consider agents with multi-step retrieval)

### 4. Monitoring and Iteration

Track these metrics:

- **Retrieval precision:** Are retrieved docs actually relevant?
- **Answer groundedness:** Does the response match the retrieved context?
- **User feedback:** Thumbs up/down on responses
- **Retrieval latency:** Time to fetch relevant docs

## Getting Started with OpenClaw RAG

OpenClaw has built-in RAG capabilities:

```yaml
# memory.yaml
rag:
  enabled: true
  sources:
    - name: company-docs
      path: ./docs/
      embedding_model: text-embedding-3-small
      chunk_size: 800
      chunk_overlap: 100
    
    - name: product-catalog
      type: postgres
      connection: ${DATABASE_URL}
      table: products
```

The agent automatically retrieves relevant context when answering questions about your docs or products.

## Conclusion

RAG transforms AI agents from generic chatbots into knowledgeable assistants that understand your specific context. Start simple with document RAG, then evolve to more sophisticated patterns as your needs grow.

The best RAG system is the one that actually gets deployed and iterated on. Perfect is the enemy of shipped.

---

*Sam Okafor builds AI systems that actually work in production. Follow for practical tutorials and hard-won lessons from the trenches.*

**Tags:** RAG, AI agents, vector search, embeddings, OpenClaw, tutorial

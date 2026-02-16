# AI Agent RAG Patterns: Building Knowledge-Augmented Assistants

*By Sam Okafor | Technical Tutorial | February 16, 2026*

You've got an AI agent. It's smart. But it doesn't know *your* stuff — your docs, your codebase, your internal wiki. That's where RAG (Retrieval-Augmented Generation) comes in.

I've built RAG systems for half a dozen production agents now. Here's what actually works.

## What RAG Actually Is (30-Second Version)

Instead of the AI making up answers from its training data:

1. User asks a question
2. Agent searches your knowledge base for relevant chunks
3. Those chunks get added to the prompt as context
4. AI answers using YOUR data, not just what it learned in training

Result: Agent that actually knows your company's stuff.

## The Basic RAG Pattern

```python
# Simplified RAG flow
def answer_question(question: str) -> str:
    # 1. Search for relevant documents
    relevant_docs = vector_db.search(
        query=embed(question),
        top_k=5
    )
    
    # 2. Build context from retrieved docs
    context = "\n\n".join([doc.text for doc in relevant_docs])
    
    # 3. Generate answer with context
    return llm.complete(
        f"Context:\n{context}\n\nQuestion: {question}\n\nAnswer:"
    )
```

This is the 101 version. Production RAG is more nuanced.

## Pattern 1: Hybrid Search

Pure vector search isn't enough. Sometimes users search for exact terms ("error code ABC123") and sometimes for concepts ("how do I handle authentication").

**The Fix: Hybrid Search**

Combine:
- **Vector search** for semantic similarity
- **Keyword search** (BM25) for exact matches

```python
def hybrid_search(query: str) -> List[Document]:
    vector_results = vector_db.search(embed(query), top_k=10)
    keyword_results = bm25_search(query, top_k=10)
    
    # Reciprocal Rank Fusion to combine scores
    return reciprocal_rank_fusion(vector_results, keyword_results)
```

This catches both "authentication flow" (semantic) and "OAuth2 callback URL" (keyword).

## Pattern 2: Contextual Chunking

Most RAG tutorials chunk documents into fixed-size pieces (500 tokens, 1000 tokens, whatever). This is lazy and it shows.

**The Problem:**

A fixed-size chunk might cut off in the middle of a concept, or include unrelated content from adjacent sections.

**Better Approach: Semantic Chunking**

Chunk based on document structure:
- Headers define chunk boundaries
- Code blocks stay intact
- Lists stay together
- Related paragraphs group together

```python
def semantic_chunk(document: str) -> List[Chunk]:
    chunks = []
    sections = split_by_headers(document)
    
    for section in sections:
        if len(section) < MAX_CHUNK_SIZE:
            chunks.append(section)
        else:
            # Only subdivide if necessary
            chunks.extend(
                split_preserving_paragraphs(section)
            )
    
    return chunks
```

## Pattern 3: Chunk Enrichment

Raw chunks lack context. A chunk from "Chapter 5: Advanced Features" needs to know it's from Chapter 5, not Chapter 1.

**Add Metadata:**

```python
chunk = {
    "text": "The authentication flow requires...",
    "metadata": {
        "source": "docs/security/auth.md",
        "section": "OAuth2 Setup",
        "parent_section": "Authentication",
        "doc_type": "technical_docs",
        "last_updated": "2026-02-01"
    }
}
```

**Add Contextual Summaries:**

For each chunk, generate a brief summary that includes context:

```python
chunk_summary = llm.complete(f"""
Document: {doc_title}
Section: {section_title}
Content: {chunk_text}

Write a brief description of what this chunk covers and how it fits into the larger document.
""")
```

Store both the raw chunk and the enriched summary. Search on the summary, but include the raw text in context.

## Pattern 4: Query Transformation

Users ask bad questions. Not their fault — they don't know the right terminology or how your docs are structured.

**Query Expansion:**

```python
def expand_query(query: str) -> List[str]:
    # Generate multiple query variations
    variations = llm.complete(f"""
    Original question: {query}
    
    Generate 3 alternative phrasings that might match relevant documentation.
    Include technical synonyms and related concepts.
    """)
    
    return [query] + variations
```

**Query Decomposition:**

Complex questions need multiple searches:

```python
def decompose_query(query: str) -> List[str]:
    # "How do I set up OAuth with Google and handle token refresh?"
    # Becomes:
    # 1. "OAuth setup with Google"
    # 2. "Token refresh handling"
    # 3. "OAuth Google token refresh"
    
    return llm.extract_sub_queries(query)
```

## Pattern 5: Reranking

Vector search returns the top K results by embedding similarity. But embedding similarity isn't the same as relevance to the actual question.

**Add a Reranker:**

After initial retrieval, use a cross-encoder model to rerank results based on actual relevance:

```python
def retrieve_and_rerank(query: str, top_k: int = 5) -> List[Document]:
    # Get more candidates than we need
    candidates = hybrid_search(query, top_k=20)
    
    # Rerank with cross-encoder
    scored = []
    for doc in candidates:
        score = reranker.score(query, doc.text)
        scored.append((score, doc))
    
    # Return top K after reranking
    scored.sort(reverse=True)
    return [doc for _, doc in scored[:top_k]]
```

Cross-encoders are slower but more accurate. Use them as a second pass, not for initial retrieval.

## Pattern 6: Self-Query

Let the agent decide what to search for:

```python
def agent_search(question: str) -> str:
    # Agent generates its own search queries
    search_plan = llm.complete(f"""
    To answer: {question}
    
    What information do I need to search for?
    List specific search queries.
    """)
    
    # Execute searches
    results = []
    for query in parse_queries(search_plan):
        results.extend(retrieve_and_rerank(query))
    
    # Answer with all gathered context
    return generate_answer(question, results)
```

This works especially well for multi-step questions.

## Pattern 7: Fallback Chains

What if RAG doesn't find relevant docs?

```python
def answer_with_fallbacks(question: str) -> str:
    # Try RAG first
    docs = retrieve_and_rerank(question)
    
    if docs and max_relevance_score(docs) > THRESHOLD:
        return answer_from_docs(question, docs)
    
    # Fallback 1: Web search
    web_results = web_search(question)
    if web_results:
        return answer_from_web(question, web_results)
    
    # Fallback 2: Acknowledge limitation
    return "I couldn't find specific information about this in our docs or online. Here's what I know from general knowledge: ..."
```

Never hallucinate. If you don't have the info, say so.

## The Architecture That Works

For production RAG agents:

```
User Question
     ↓
Query Transformation (expand, decompose)
     ↓
Hybrid Search (vector + keyword)
     ↓
Reranking (cross-encoder)
     ↓
Context Assembly (chunks + metadata)
     ↓
LLM Generation
     ↓
Citation/Source Attribution
     ↓
Answer with References
```

## Common Mistakes

**Mistake 1: Not citing sources**
Always tell users WHERE the information came from. Include doc links or section references.

**Mistake 2: Chunking too small**
Tiny chunks lose context. I'd rather have 5 medium chunks than 15 tiny ones.

**Mistake 3: Not handling "I don't know"**
If the retrieval score is low, the agent should say so. Don't guess.

**Mistake 4: Ignoring freshness**
Old docs shouldn't rank equally with current ones. Add recency weighting.

**Mistake 5: Skipping the reranker**
Initial retrieval is just candidate generation. Reranking is where accuracy happens.

## Quick Start Checklist

- [ ] Set up vector database (Pinecone, Weaviate, Qdrant, Chroma)
- [ ] Chunk documents semantically (not fixed-size)
- [ ] Add metadata to chunks
- [ ] Implement hybrid search
- [ ] Add reranking
- [ ] Handle "no results" gracefully
- [ ] Include source citations
- [ ] Test with real user questions

## Final Thoughts

RAG isn't just "add a vector database." It's a retrieval system that needs the same care you'd give any production search system.

The agents that feel magical? They have really good RAG. The ones that hallucinate or miss obvious answers? They have basic RAG.

Invest in retrieval quality. It's the foundation everything else builds on.

---

*Sam Okafor builds AI agents and developer tools. Previously at Vercel and GitHub.*

**Related:**
- [Vector Databases Compared: 2026 Edition](/articles/vector-databases-2026)
- [Embedding Models for RAG](/articles/embedding-models-rag)
- [Building Document Pipelines](/articles/document-pipelines)

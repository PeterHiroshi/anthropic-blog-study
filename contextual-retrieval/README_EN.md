# Contextual Retrieval

> Source: https://www.anthropic.com/engineering/contextual-retrieval
> Published: September 19, 2024 | Anthropic Engineering

---

## Overview Mind Map

```
Contextual Retrieval
├── Core Problem: RAG Loses Context
│   └── Chunks stripped of surrounding context → retrieval failures
│
├── Solution: Add Context Before Encoding
│   ├── Contextual Embeddings — contextualize then embed
│   └── Contextual BM25 — contextualize then index
│
├── Performance Results
│   ├── Embeddings alone: 35% failure reduction (5.7% → 3.7%)
│   ├── + BM25: 49% failure reduction (5.7% → 2.9%)
│   └── + Reranking: 67% failure reduction (5.7% → 1.9%)
│
├── Implementation
│   ├── Use Claude 3 Haiku for context generation
│   ├── 50-100 token context per chunk
│   └── Prompt caching: ~$1.02 per million tokens
│
└── Best Practices
    ├── Use Gemini/Voyage embeddings
    ├── Retrieve top-20 chunks
    ├── Combine embeddings + BM25
    ├── Apply reranking as final step
    └── Tailor prompts to domain
```

---

## 1. The Core Problem with Traditional RAG

**What it is:** Standard RAG (Retrieval-Augmented Generation) splits documents into chunks, embeds them, and retrieves relevant chunks at query time. But this process loses critical context.

```
Original Document:
"...In Q2 2024, Apple's revenue reached $89.5 billion.
This represents a 5% increase from Q2 2023.
The growth was driven primarily by iPhone sales..."

After Chunking:
Chunk 47: "This represents a 5% increase from Q2 2023."

The Problem: WHAT is 5% increase? The chunk doesn't say!
```

**Why this causes failures:**
```
User Query: "What was Apple's revenue growth in Q2 2024?"

Chunk 47 text: "This represents a 5% increase from Q2 2023."
Embedding: [0.23, -0.45, 0.67, ...] ← vague, doesn't capture "Apple revenue"

Result: Chunk 47 may NOT be retrieved because it's contextually ambiguous!
```

**Scale of the problem:** This causes retrieval failures that directly harm the final answer quality — irrelevant content is retrieved, and relevant content is missed.

---

## 2. The Solution: Contextual Retrieval

**Core idea:** Before embedding or indexing each chunk, add a short contextual description that explains the chunk's place within the larger document.

```
Contextualized Chunk (using Claude 3 Haiku):
┌─────────────────────────────────────────────────────────────┐
│ Context prefix (50-100 tokens, auto-generated):             │
│ "This chunk is from Apple's Q2 2024 earnings report,        │
│  discussing revenue growth metrics compared to Q2 2023."    │
│                                                             │
│ Original chunk:                                             │
│ "This represents a 5% increase from Q2 2023."              │
└─────────────────────────────────────────────────────────────┘

Now the embedding captures: Apple + revenue + Q2 2024 + 5% growth
→ Much higher retrieval accuracy!
```

### Two techniques combined:

**Contextual Embeddings:** Add context → generate embedding → store in vector DB
**Contextual BM25:** Add context → build BM25 keyword index → store in BM25 index

At retrieval time, combine both scores for best results.

---

## 3. Performance Results

All improvements measured against standard chunking (baseline: 5.7% retrieval failure rate):

| Technique | Failure Rate | Reduction |
|-----------|-------------|-----------|
| Standard chunking (baseline) | 5.7% | — |
| Contextual Embeddings only | 3.7% | **35% reduction** |
| Contextual Embeddings + BM25 | 2.9% | **49% reduction** |
| Contextual Embeddings + BM25 + Reranking | 1.9% | **67% reduction** |

```
Failure Rate Comparison (lower is better)

5.7% │ ████████████████████████████
     │
3.7% │ ████████████████████
     │
2.9% │ ████████████████
     │
1.9% │ ██████████
     └──────────────────────────────────
       Baseline  +Emb  +BM25  +Rerank
```

**Key takeaway:** Each technique compounds with the others. All three together give 67% improvement over baseline.

---

## 4. Implementation: How Context is Generated

**Step 1: Prepare the context generation prompt**

```python
CONTEXT_PROMPT = """
<document>
{WHOLE_DOCUMENT}
</document>

Here is the chunk we want to situate within the whole document:
<chunk>
{CHUNK_CONTENT}
</chunk>

Please give a short succinct context to situate this chunk within the
overall document for the purposes of improving search retrieval of the
chunk. Answer only with the succinct context and nothing else.
"""
```

**Step 2: Generate context for each chunk using Claude 3 Haiku**

```python
context = claude_haiku.generate(
    prompt=CONTEXT_PROMPT.format(
        WHOLE_DOCUMENT=full_document,
        CHUNK_CONTENT=chunk
    )
)
# Returns: 50-100 token description
```

**Step 3: Prepend context to chunk**
```python
contextualized_chunk = context + "\n\n" + chunk
```

**Step 4: Index the contextualized chunk** (both embedding and BM25)

**Important:** This preprocessing happens **once per chunk** during indexing, not at query time.

---

## 5. Cost Efficiency with Prompt Caching

**The challenge:** If a document has 1,000 chunks, you'd need to send the full document with each chunk generation request — expensive!

**Solution:** Anthropic's **prompt caching** feature caches the document in the prompt.

```
Without Caching:
Request 1: [50K doc tokens] + [chunk 1] → generate context
Request 2: [50K doc tokens] + [chunk 2] → generate context
...
Request 1000: [50K doc tokens] + [chunk 1000] → generate context
Total: 50M+ tokens at full price

With Prompt Caching:
Request 1: [50K doc tokens cached] + [chunk 1] → generate context
Request 2: [cached] + [chunk 2] → generate context  (cache hit!)
...
Total: ~50K tokens at full price + 999 × short tokens at cache price
Cost: ~$1.02 per million document tokens
```

---

## 6. The Hybrid Retrieval Architecture

**Full pipeline for best results:**

```
Query
  │
  ▼
┌──────────────────────────────────────────────┐
│              Hybrid Retrieval                 │
│                                              │
│  ┌─────────────────┐  ┌────────────────────┐ │
│  │  Vector Search   │  │  BM25 Keyword      │ │
│  │  (Semantic)      │  │  Search (Exact)    │ │
│  │                  │  │                    │ │
│  │  Top-20 results  │  │  Top-20 results    │ │
│  └────────┬─────────┘  └──────────┬─────────┘ │
│           │                       │            │
│           └──────────┬────────────┘            │
│                      ▼                         │
│              Reciprocal Rank Fusion             │
│              (Combine rankings)                 │
└──────────────────────┬───────────────────────-─┘
                       │
                       ▼ Top-N candidates
               ┌───────────────┐
               │   Reranking   │  ← Cohere, Voyage, or similar
               │   Model       │    (cross-encoder)
               └───────┬───────┘
                       │
                       ▼ Final Top-K
               Pass to LLM for answer generation
```

**Why BM25 + Embeddings?**
- Embeddings: great for semantic similarity ("what is similar in meaning?")
- BM25: great for exact keyword matches ("what contains these exact words?")
- Together: covers both semantic and lexical retrieval

---

## 7. For Small Knowledge Bases: Alternative Approach

For knowledge bases **under 200,000 tokens**, there's a simpler option:

```
Option A: Chunking + Retrieval (any size)
Document → Chunks → Embed → Retrieve → Answer

Option B: Full Context (< 200K tokens)
Document (entire) → Include in prompt → Answer

Recommendation: If your knowledge base fits in the context window,
just include it all! Use prompt caching to make it affordable.
```

**When to use each:**
| Knowledge Base Size | Recommendation |
|---------------------|---------------|
| < 200K tokens | Full context in prompt |
| 200K - few million | Contextual Retrieval |
| Millions+ | Contextual Retrieval + optimization |

---

## 8. Best Practices Summary

| Practice | Detail |
|----------|--------|
| **Embedding model** | Use Gemini or Voyage (outperform others in benchmarks) |
| **Retrieval count** | Retrieve top-20 chunks (better than top-5 or top-10) |
| **Hybrid search** | Always combine embeddings + BM25 |
| **Reranking** | Apply as final optimization step |
| **Context length** | Generate 50-100 token context per chunk |
| **Domain tuning** | Tailor contextualizer prompts to your specific domain |
| **Cost efficiency** | Use prompt caching for chunk context generation |

---

## Quick Reference

**Core formula:**
```
Better RAG = Contextual Embeddings + Contextual BM25 + Reranking
           = 67% fewer retrieval failures vs. baseline
```

**Implementation cost:**
- One-time preprocessing with Claude 3 Haiku
- ~$1.02 per million document tokens with prompt caching
- No change to query-time latency (preprocessing is offline)

**The key insight:** RAG fails because chunks lose context. The fix is simple: add context before encoding. This one change compounds with every other optimization.

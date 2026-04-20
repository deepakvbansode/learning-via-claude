# RAG (Retrieval-Augmented Generation)

## The Problem It Solves

LLMs are frozen at training time — like a Docker image baked at build. They don't know your private org data or anything after their training cutoff. And you can't dump your entire knowledge base into every prompt because context windows are finite. RAG solves this by retrieving only the relevant slices of your data at query time and injecting them into the prompt — so the model answers from current, private context without retraining.

---

## Core Concepts

### Embeddings

**Why:** Keyword search (grep, Elasticsearch) breaks on semantics — different words can mean the same thing. You need to compare *meaning*, not characters.

**Mental model:** An embedding model is a compiler. Text goes in, a vector (list of ~1536 numbers) comes out. Similar meanings compile to nearby coordinates in high-dimensional space. The same model must be used at index time and query time — like using the same compiler for all your object files before linking. Mix models and the vectors live in incompatible spaces.

**Example:**

```text
# Both produce vectors that are geometrically close — despite zero shared keywords
embed("how do I rotate TLS certs?")         → [0.23, -0.87, 0.41, ...]
embed("how do I renew HTTPS certificates?") → [0.21, -0.83, 0.39, ...]
```

AWS: **Amazon Titan Embeddings** via Bedrock

---

### Vector Databases

**Why:** You have millions of vectors. You need to find the top-5 closest to a query vector fast. O(n) full scan doesn't scale past thousands.

**Mental model:** Like a Postgres B-tree index but for proximity in high-dimensional space. Uses **ANN (Approximate Nearest Neighbor)** indexing — trades a tiny bit of accuracy for massive speed gains. The DB has no concept of "meaning" — it just computes distance between coordinates. Meaning is baked into the vectors by the embedding model.

**Example:**

```python
query_vector = embed("how do I renew HTTPS certificates?")

results = vector_db.search(vector=query_vector, top_k=3)
# returns 3 most semantically similar document chunks
```

**Similarity metrics:**

```text
Cosine similarity   # most common for text — measures angle between vectors
Euclidean distance  # measures straight-line distance between points
```

AWS: **OpenSearch Serverless** (vector engine) or **pgvector on Aurora PostgreSQL**

---

### Chunking Strategy

**Why:** You can't embed a 200-page doc as one vector — retrieving it would blow the context window. Each chunk gets its own vector; you retrieve only the relevant chunks.

**Mental model:** Like Kubernetes resource requests — too tight (chunk too small) and you lose surrounding context; too loose (chunk too large) and the answer is buried in noise. Overlap between chunks is like TCP sequence numbers — prevents meaning loss at chunk boundaries.

**Example:**

```text
Fixed-size:
  [...500 tokens...] [...500 tokens...] [...500 tokens...]

Fixed-size with overlap (prevents boundary loss):
  [...500+50 overlap...]
                  [...50+500+50 overlap...]

Semantic (split on natural boundaries — paragraphs, headings):
  [Introduction section] [Step 1 section] [Step 2 section]
```

| Chunk size | Precision | Recall | Risk                             |
|------------|-----------|--------|----------------------------------|
| Too small  | High      | Low    | Loses surrounding context        |
| Too large  | Low       | High   | Relevant answer buried in noise  |

AWS: **Bedrock Knowledge Bases** configures chunking strategy automatically (fixed-size, semantic, or hierarchical).

---

### The RAG Pipeline

**Why:** Two distinct pipelines run at different times — ingestion (offline) and query (online). Keeping them separate is key: the LLM is only involved at the very end and has no knowledge of the vector DB. All retrieval intelligence happens before the model is called.

**Mental model:** Ingestion = CI/CD build pipeline (runs when source changes, slow, batch). Query = production API endpoint (per-request, latency-sensitive). The LLM is just a text-in/text-out function at the end — retrieval quality determines answer quality, not the model.

**Ingestion pipeline (offline — runs once or on schedule):**

```text
S3 (raw documents)
  → chunk documents
  → embed each chunk  (Titan Embeddings)
  → store vector + original text in vector DB  (OpenSearch)
```

**Query pipeline (online — runs per user request):**

```text
User question
  → embed the question  (same Titan Embeddings model)
  → similarity search in vector DB  → top-k chunks returned
  → build prompt:
      [system instruction]
      [chunk_1: retrieved context]
      [chunk_2: retrieved context]
      [user question]
  → send to LLM  (Claude on Bedrock)
  → return answer
```

**Two independent failure modes:**

```text
Retrieval failure  →  wrong chunks returned      (fix: chunk strategy, embedding model)
Model failure      →  right chunks, bad answer   (fix: prompt engineering, model choice)
```

AWS: **Bedrock Knowledge Bases** manages both pipelines end-to-end as a single managed service.

---

### AWS Architecture Choices

**Why:** On AWS you choose between managed (Bedrock Knowledge Bases) or DIY (wiring services yourself). Same trade-off as EKS Fargate vs self-managed nodes — convenience vs control.

**Mental model:** Bedrock KB = EKS Fargate (fast to ship, opaque internals, limited tuning knobs). DIY = self-managed nodes (full observability, full responsibility). The black-box nature of Bedrock KB is the key constraint — if retrieval quality is poor, you have limited levers to diagnose or fix it.

**Managed — Bedrock Knowledge Bases:**

```python
# Point at S3, call one API — chunking, embedding, indexing, retrieval handled for you
response = bedrock_agent_runtime.retrieve_and_generate(
    input={"text": "how do I renew HTTPS certificates?"},
    retrieveAndGenerateConfiguration={
        "type": "KNOWLEDGE_BASE",
        "knowledgeBaseConfiguration": {
            "knowledgeBaseId": "KB123",
            "modelArn": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-sonnet"
        }
    }
)
```

**DIY — wire services yourself:**

```text
S3
  → Lambda (chunking)
  → Titan Embeddings (Bedrock)
  → OpenSearch Serverless
  → Lambda (query handler)
  → Claude on Bedrock
```

**When to use which:**

| | Bedrock Knowledge Bases | DIY |
| --- | --- | --- |
| Use when | PoC, fast shipping, standard use case | Custom chunking, retrieval tuning, observability |
| Control over retrieval | Low | Full |
| Ops overhead | Minimal | Higher |
| Debuggability | Limited | Full |

**OpenSearch Serverless vs pgvector on Aurora:**

| | OpenSearch Serverless | pgvector on Aurora |
| --- | --- | --- |
| Best for | Pure vector search at scale | Already using Aurora; mixed SQL + vector queries |
| Scales to zero | Yes | No (Aurora has a minimum cost) |
| Operational overhead | Low | Low (managed) |

---

## Gotchas

1. **Use the same embedding model for ingestion and query** — mixing models is like compiling with different compilers; the vectors live in incompatible spaces and similarity search returns garbage.
2. **Retrieval is the #1 failure mode** — if wrong chunks are retrieved, the LLM will confidently answer from bad context. Always debug retrieval before blaming the model.
3. **Chunk size is the hardest knob to tune** — start with fixed-size + overlap as a baseline; switch to semantic chunking if retrieval quality is poor.
4. **Bedrock KB is a black box** — you can't inspect what was retrieved or tune the retrieval step. Choose DIY if observability or custom retrieval logic matters.
5. **Context window is a hard ceiling** — prompt + retrieved chunks + model response must all fit. Large chunk sizes eat into the space available for the model's response.

---

## Quick Reference

**AWS services:**

```text
S3                        # document storage — source of truth
Titan Embeddings          # embedding model — text → vector (Bedrock)
OpenSearch Serverless     # vector DB — ANN index, scales to zero
pgvector on Aurora        # vector DB — if you already use Aurora
Claude on Bedrock         # LLM — generation step
Bedrock Knowledge Bases   # fully managed RAG pipeline (ingestion + query)
```

**Chunking strategies:**

```text
Fixed-size            # simplest, split every N tokens — good default
Fixed-size + overlap  # add overlap window to prevent boundary loss — recommended starting point
Semantic              # split on natural boundaries (paragraphs, headings) — best quality
Hierarchical          # nested chunks (small for retrieval, large for context) — advanced
```

**When to use each capability:**

| Capability | Use when |
| --- | --- |
| Bedrock KB (managed) | PoC, fast shipping, standard retrieval is sufficient |
| DIY pipeline | Custom chunking, hybrid search, re-ranking, full observability |
| OpenSearch Serverless | Primary vector store, large scale, scales to zero |
| pgvector on Aurora | Already using Aurora, need SQL + vector in one store |
| Fine-tuning (not RAG) | Need to change model *behavior* or *style*, not just inject facts |

---

## Session Handoff Prompt

Copy this into a new Claude Code session when you're ready to build:

> I know RAG well. Quick context: LLMs are frozen at training time and have no access to private or recent data. RAG fixes this by chunking documents, converting them to vectors via an embedding model (Titan Embeddings), storing in a vector DB (OpenSearch Serverless), and at query time retrieving only the relevant chunks to inject into the prompt. The LLM only sees the final assembled prompt — all intelligence is in the retrieval step. On AWS: Bedrock Knowledge Bases for the managed pipeline, or DIY with Lambda + OpenSearch + Bedrock when I need custom retrieval control. I'm building: [describe your task here]

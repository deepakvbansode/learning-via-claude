# Amazon Bedrock

## The Problem It Solves

Calling AI models directly means managing separate SDKs, API keys, billing, and compliance exposure per vendor — and traffic leaving your AWS network. Bedrock is AWS's managed hosting layer for licensed AI models — it runs models on AWS infrastructure inside your region, unifies auth via IAM, consolidates billing, and lets you swap models by changing one string.

---

## Core Concepts

### What Bedrock Is

**Why:** Avoid vendor fragmentation and keep sensitive data inside your AWS network instead of routing it to external APIs.

**Mental model:** Like RDS vs. self-managed Postgres on EC2 — AWS hosts and serves the resource on their infra inside your region, billed to your account, accessible via IAM. You never touch the GPU servers. Same idea as ElastiCache for Redis, but for AI models.

**Example:**

```text
Available providers:  Anthropic (Claude), Meta (Llama), Mistral,
                      Amazon (Titan/Nova), Cohere, AI21, Stability AI
NOT available:        OpenAI, Google  →  use Azure OpenAI / Vertex AI instead
```

---

### Foundation Models

**Why:** Pre-trained on billions of tokens so you don't train from scratch. Reasoning, summarization, and code understanding come built-in — no ML team, no GPU budget, no training pipeline.

**Mental model:** Like a Docker base image (`golang:1.22`) — someone else did the heavy build work, you layer your use case on top. A foundation model is that base image, but for intelligence.

**Example:**

```text
Context window = max request payload size (prompt + response combined)
Claude 3.5 Sonnet:  ~200k tokens  (~150k words)
Llama 3 70B:        ~128k tokens
Exceed the limit → call fails
```

**Fine-tuning vs RAG:**

```text
RAG          →  give the model new facts at query time (your private docs)
Fine-tuning  →  change how the model behaves/reasons (style, format, domain logic)
```

---

### Inference / Converse API

**Why:** Converse API gives you a unified request/response contract across all models — swap `modelId` and nothing else in your code changes.

**Mental model:** Converse API = shared gRPC interface definition that all models implement. InvokeModel = calling each service's raw REST endpoint with its own proprietary schema. Use Converse unless you need model-specific features.

**Example:**

```python
response = bedrock.converse(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
    messages=[
        {"role": "user", "content": [{"text": "Summarize this contract: ..."}]}
    ]
)
# swap modelId to meta.llama3-70b-instruct-v1:0 → everything else unchanged
```

**Sync vs streaming:**

```python
bedrock.converse()        # blocking — waits for full response (batch jobs, DB writes)
bedrock.converse_stream() # streaming — tokens as generated (chat UIs, user-facing)
# Note: streaming reduces time-to-first-token, NOT total generation time
```

---

### Knowledge Bases (RAG)

**Why:** Foundation models don't know your private data, and you can't fit 50k documents into a context window. RAG retrieves only the relevant chunks at query time and injects them into the prompt.

**Mental model:** Like a Postgres index — you don't full-scan every row for every query, you index then seek. RAG indexes your document corpus so only semantically relevant chunks are fetched per query. Bedrock Knowledge Bases manages the entire pipeline as a managed service.

**Flow:**

```text
Ingest (one-time):
  S3 bucket (your docs) → chunked → embedded → stored in vector DB

Query (per request):
  1. User query → converted to embedding
  2. Vector DB → similarity search → top N chunks retrieved
  3. user query + retrieved chunks → assembled into prompt
  4. Prompt → sent to model
  5. Model responds using retrieved context

Key: retrieval happens BEFORE the model is involved.
     The model only sees the assembled prompt — it has no knowledge of the vector DB.
```

**Two independent failure modes:**

```text
Retrieval failure  →  wrong chunks returned  (fix: chunk strategy, embedding model)
Model failure      →  right chunks, bad reasoning  (fix: prompt, model choice)
```

---

### Agents

**Why:** Multi-step tasks spanning multiple systems can't be done in a single model call. Agents let you describe a goal in natural language and let the model figure out which tools to call and in what order.

**Mental model:** Like a Temporal workflow, but the model defines the execution plan at runtime instead of you writing the steps in code. You provide tools (Lambda functions, APIs, Knowledge Bases); the agent reasons about what to call and when.

**Flow:**

```text
User goal → agent reasons → picks tool → calls it → observes result
          → reasons again → picks next tool → ...  → final response
```

**When to use which:**

```text
Bedrock Agent  →  open-ended goals, flexible multi-step reasoning, natural language tasks
Temporal       →  deterministic, auditable, replayable, compliance-critical workflows
Combined       →  Temporal as outer orchestrator + Bedrock Agent as one reasoning step inside
```

---

## Gotchas

1. **OpenAI and Google are not on Bedrock** — if you need GPT-4 or Gemini, use Azure OpenAI or Vertex AI respectively; Bedrock cannot help you there
2. **Streaming reduces perceived latency, not total generation time** — the model generates at the same speed either way; don't promise "faster AI" from switching to streaming
3. **Retrieval quality determines answer quality** — wrong chunk strategy means the model never sees the right context, regardless of model capability; RAG failures are usually retrieval failures
4. **Agents are non-deterministic** — same input can produce different execution paths on different runs; don't use them for compliance-critical flows where every step must be auditable
5. **Context window is a hard ceiling** — prompt + retrieved chunks + model response must all fit; for RAG, large chunk sizes eat into space available for the response

---

## Quick Reference

**Model IDs (Converse API):**

```text
anthropic.claude-3-5-sonnet-20241022-v2:0   # Claude 3.5 Sonnet — general purpose
meta.llama3-70b-instruct-v1:0               # Llama 3 70B — open weights
mistral.mistral-large-2402-v1:0             # Mistral Large
amazon.titan-text-express-v1                # Amazon Titan — cheapest option
```

**Inference patterns:**

```python
bedrock.converse()         # sync, full response — batch jobs, DB writes
bedrock.converse_stream()  # streaming — chat UIs, user-facing features
```

**When to use each capability:**

| Capability      | Use when                                               |
|-----------------|--------------------------------------------------------|
| Plain Inference | Single-turn tasks: classify, summarize, extract        |
| Knowledge Bases | Model needs to reason over your private documents      |
| Agents          | Multi-step tasks spanning multiple APIs/systems        |
| Fine-tuning     | Need consistent behavior/style baked in, not just data |

---

## Session Handoff Prompt

Copy this into a new Claude Code session when you're ready to build:

> I know Amazon Bedrock well. Quick context: Bedrock is AWS's managed hosting layer for licensed AI models (Anthropic, Meta, Mistral — not OpenAI/Google) running inside AWS, with IAM auth and unified billing. The Converse API provides a model-agnostic interface so you can swap models by changing one string. Knowledge Bases manages the full RAG pipeline — retrieval happens before the model is involved, so failures split cleanly into retrieval vs. reasoning. Agents let the model orchestrate multi-step tool use at runtime, trading determinism for flexibility; pair with Temporal when auditability matters. I'm building: [describe your task here]

# RAG model selection — deeper explanation

### What “model capabilities” mean in RAG

- Information extraction: From retrieved chunks, identify facts, entities, numbers, decisions, and cite locations. Prefer extraction over paraphrase when the question asks for facts.
- Context understanding: Link multiple chunks, resolve pronouns and synonyms, handle contradictions, and summarize consistently with citations.
- Tool use and function calling: Decide when retrieval is insufficient, call tools such as calculators, search, APIs, or run structured functions, then combine tool outputs with text reasoning.

---

### Four steps to choosing a large model

### 1) Model size selection (start large, then right-size)

- Goal: validate feasibility and ceiling quality quickly.
- Practical picks: Frontier API model for a pilot, or a strong open model locally.
- Shrink path: distill or quantize to a smaller open model once tasks and prompts stabilize.

### Heuristics by constraint

- Laptop or small VM: 1.5B–8B quantized (Q4/Int8) for routing, extraction, and short answers.
- Single CPU server: 7B–13B quantized for RAG chat and summaries.
- Single GPU (24–48GB): 8B–14B FP16/INT8; long-context variants if needed.
- API-first: choose by eval scores, latency SLAs, and cost per 1K tokens.

### 2) Capability testing (task-focused evals)

- Build a small, high-quality test set reflecting your real questions.
- Split by skill: extraction, multi-hop reasoning, numerical reasoning, policy compliance, hallucination resistance.
- For each test, store: question, gold answer, supporting chunk IDs, allowed tools, and expected format.
- Measure: exact match or F1 for extraction. Faithfulness with citation checks. Response JSON validity for tool functions. Latency p50 and p95, and cost per query.
- RAG-specific probes: no-answer cases, conflicting docs, long-tail synonyms, and multi-chunk stitching.

### 3) Cost and hardware

- Track three dials per model: quality, latency, and price.
- Compute cost model: prompts per day × avg input tokens × price + output tokens × price + retrieval and rerank costs.
- Hardware notes: use quantized weights on CPU, enable KV cache reuse, keep context windows tight with reranking, and stream tokens for UX.

### 4) Enterprise data security

- Retrieval layer: store vectors in approved infra, encrypt at rest, restrict by namespace and user identity.
- Model access: prefer server-side calls. When using external APIs, redact PII, mask secrets, and log only hashes of sensitive fields.
- Prompt hygiene: include safety rails and “answer only from provided context; otherwise say you don’t know.”
- Evaluation: add red-team prompts for data exfiltration and prompt injection. Track refusal vs leakage rate.

---

### Minimal reference checklist

- [ ]  A baseline “big” model proves task feasibility.
- [ ]  A smaller candidate meets the target quality within latency and budget.
- [ ]  Test set with citations and no-answer cases is versioned.
- [ ]  Cost dashboard shows cost per query and monthly run-rate.
- [ ]  Security review completed for data store, prompts, and external calls.

---

### Example prompt skeleton for RAG

```
System: You are a precise enterprise assistant. Answer only with the provided context. If the context is insufficient, say "I don't have enough information." Provide citations like [doc:id:chunk].

User question: question

Context (top-k, deduplicated):
chunk_1
---
chunk_2
---
chunk_3

Output format: JSON with fields {"answer": string, "citations": [string]}
```

---

### Function-calling contract example

```json
{
  "name": "get_policy_limit",
  "description": "Lookup expense policy limit by category and role",
  "parameters": {
    "type": "object",
    "properties": {
      "category": {"type": "string"},
      "role": {"type": "string"}
    },
    "required": ["category", "role"]
  }
}
```

Model decides when to call, RAG supplies the policy excerpt; runtime validates and injects the tool result back into the answer.

---

### Next steps (suggested)

- Populate a 30–50 item eval set from your finance and policy examples.
- Try one "big" API model and one local quantized model; record quality, latency, and cost.
- Pick the smallest model that clears your quality bar, then optimize prompts and retrieval.
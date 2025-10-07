# Four Roles for Enterprise RAG — Responsibilities, Skills, and Concrete Examples

### How to use this page

This page outlines four key roles for building enterprise RAG systems. Each role has responsibilities, day‑to‑day tasks, skill checklist, anti‑patterns, and concrete examples.

---

## 1) Large Model Application Developer

- Mission: Turn business needs into working features.
- Responsibilities
    - Design prompts, retrieval pipelines, and output formats
    - Wire up tools and function calls
    - Own end‑to‑end user experience and quality bar
- Day‑to‑day tasks
    - Write prompts and unit tests for extraction, summarization, and Q&A
    - Build RAG chains with reranking and citation checks
    - Define JSON schemas for function calls and validate responses
- Skills checklist
    - Transformers basics, embeddings, vector DBs, LangChain/LlamaIndex
    - Python or Java service integration, streaming, retries, timeouts
- Anti‑patterns
    - Passing huge context without reranking
    - Letting the model answer without citations or no‑answer handling
- Concrete examples
    1. Policy Q&A: Map “travel category + role” to `get_policy_limit(category, role)`; the model must call the tool when the policy text is ambiguous.
    2. Finance insights: Combine top‑k facts from a knowledge graph + PDF tables; require the model to output {"answer","citations"}.
    3. Security: Add prompt rule “answer only from context or say you don’t know,” log refused cases for review.

---

## 2) Large Model Inference / Deployment Engineer

- Mission: Ship stable, fast, and cost‑efficient inference.
- Responsibilities
    - Choose serving stack (vLLM, TGI, TensorRT‑LLM, Ollama for CPU)
    - Capacity planning, autoscaling, monitoring, and model versioning
    - Latency and throughput tuning, token streaming
- Day‑to‑day tasks
    - Quantize open models for CPU nodes; enable KV‑cache and paged attention
    - Configure p50/p95 SLOs, circuit breakers, canary rollouts
    - Observability: tokens, latency, errors, cost per request
- Skills checklist
    - CUDA and parallelism concepts, inference frameworks, containers
- Anti‑patterns
    - Serving long contexts by default; no rate limits; missing health checks
- Concrete examples
    1. CPU‑only: Serve qwen2‑1.5b Q4 via Ollama for routing and extraction.
    2. GPU: Deploy Llama‑3‑8B on vLLM with tensor parallel 1, paged attention on, max context 8k, streaming enabled.
    3. Rollout: Blue/green switch for new prompt template with 10% traffic canary and auto‑rollback on regression.

---

## 3) Large Model Algorithm / Training Engineer

- Mission: Pretrain or fine‑tune domain models where needed.
- Responsibilities
    - Data curation, cleaning, deduplication, safety filtering
    - Choose and run LoRA/PEFT, SFT, DPO or preference optimization
    - Build evaluation suites and ablation studies
- Day‑to‑day tasks
    - Convert PDFs and HTML to structured chunks; remove boilerplate
    - Train LoRA heads for policy style and citation‑friendly outputs
    - Run evals for faithfulness, exact‑match, and JSON validity
- Skills checklist
    - PyTorch, DeepSpeed/Megatron, tokenizers, PEFT libraries
- Anti‑patterns
    - Fine‑tuning before fixing retrieval and prompting; leaking PII into training
- Concrete examples
    1. Policy tone LoRA: 2–4 hour SFT on internal policies to reduce refusal and improve citation language.
    2. Distillation: Compress API responses into a 7B student for on‑prem.
    3. Reranker training: Pairwise ranking on internal Q/A to improve top‑1 hit rate.

---

## 4) Product Manager (AI PM)

- Mission: Decide what to build and why; ensure ROI and safety.
- Responsibilities
    - Define use cases, success metrics, and acceptance tests
    - Prioritize privacy, compliance, and roll‑out stages
    - Coordinate with eng for cost and latency budgets
- Day‑to‑day tasks
    - Write problem statements and evaluation rubrics with examples
    - Shepherd red‑team tests for prompt injection and data leakage
    - Monitor adoption, costs, and incident reviews
- Skills checklist
    - LLM basics, prompt literacy, experiment design, data governance
- Anti‑patterns
    - Shipping without a measurable quality bar or a fallback UX
- Concrete examples
    1. Success metric pack: EM/F1 for extraction, faithfulness rate, p95 latency < 3 s, cost/query < target.
    2. Launch plan: Internal pilot → limited beta → GA with audit trail and opt‑out.
    3. Data governance: Define which sources are allowed into the vector store and retention windows.

---

### Cross‑role handoff example (end‑to‑end)

1) PM defines: “Expense policy Q&A” with metrics and guardrails.

2) App Dev builds a RAG chain + `get_policy_limit` tool and tests.

3) Inference Eng deploys quantized 7B for CPU and 8B on vLLM for peak.

4) Algo Eng trains a small LoRA to improve policy style and citations.

---

### Ready‑to‑use checklists

- App Dev: context ≤ 3k, dedup, rerank, cite, no‑answer path, JSON schema
- Inference: p95 latency SLO, streaming, autoscale, circuit breaker, rollback
- Algo: clean data, PEFT config, eval suite, PII redaction
- PM: metrics, rollout gates, audit logging, cost guardrail
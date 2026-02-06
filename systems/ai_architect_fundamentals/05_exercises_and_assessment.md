# Section 5: Exercises & Self-Assessment

> **Purpose:** Verify mastery. These exercises test architectural thinking, not coding ability.  
> **Instructions:** Do these AFTER completing all four sections. Attempt without looking back first.

---

## Part A: Concept Verification (30 minutes)

### Exercise 1: Trace the Path
**Without looking at notes,** trace the complete path for this input through an LLM:

Input: *"The truck battery SOC dropped to 23%"*

Draw/write:
1. How is this tokenized? (estimate the tokens)
2. What do the embeddings capture?
3. How does attention help the model understand "SOC" relates to "battery"?
4. What is the model predicting at each step?

<details>
<summary>Self-check</summary>

1. **Tokens:** ["The", " truck", " battery", " SOC", " dropped", " to", " 23", "%"] — approximately 8 tokens. Note: "SOC" might be a single token or split depending on the tokenizer.

2. **Embeddings:** Each token gets a ~768-3072 dimensional vector. "battery" and "SOC" embeddings are relatively close in the embedding space due to co-occurrence in training data. Position encodings are added so the model knows word order.

3. **Attention:** In the attention layers, "SOC" attends strongly to "battery" (learning that SOC is a battery metric). "dropped" attends to "SOC" (subject-verb relationship). "23%" attends to "dropped" and "SOC" (the value and what it measures). Multiple attention heads capture different relationships.

4. **Prediction:** If generating a response, the model predicts one token at a time. Given "The truck battery SOC dropped to 23%", it might predict next token "which" (P=0.15) or "," (P=0.25) or "." (P=0.10), building a response about low state of charge.
</details>

### Exercise 2: Cost Estimation
Your EV fleet AI system handles:
- 5,000 diagnostic queries per day (avg 300 input tokens, 500 output tokens)
- 500 report generation requests per day (avg 2,000 input tokens, 1,500 output tokens)
- 50,000 alert classification tasks per day (avg 100 input tokens, 20 output tokens)

Design a model routing strategy and estimate daily costs. Compare "all Sonnet" vs your optimized approach.

<details>
<summary>Self-check</summary>

**All Sonnet (Claude Sonnet 4 at $3/1M input, $15/1M output):**
- Diagnostics: 5K × (300×$3 + 500×$15)/1M = $0.0045 + $0.0375 per query = $42 × 5 = $210/day (correction: 5000 × $0.042 = $210)
  - Input: 5000 × 300 = 1.5M tokens × $3/1M = $4.50
  - Output: 5000 × 500 = 2.5M tokens × $15/1M = $37.50
- Reports: Input: 500 × 2000 = 1M × $3 = $3. Output: 500 × 1500 = 0.75M × $15 = $11.25
- Alerts: Input: 50K × 100 = 5M × $3 = $15. Output: 50K × 20 = 1M × $15 = $15
- **Total all-Sonnet: ~$86/day**

**Optimized routing:**
- Diagnostics → Sonnet (complex reasoning needed): Same $42/day
- Reports → Sonnet: Same $14.25/day
- Alerts → **Haiku** ($0.25/1M input, $1.25/1M output): Input $1.25, Output $1.25 = **$2.50/day** (vs $30!)
- **Total optimized: ~$59/day** — saving ~31%

**Even further:**
- Add caching for repeat diagnostic queries (20% hit rate): saves ~$8.40
- Compress system prompt for alerts (no need for long prompt): saves tokens
- **Estimated total: ~$48/day** — saving ~44%
</details>

### Exercise 3: Failure Classification
For each scenario, classify the failure type and propose a fix:

**A)** The system says "NMC811 cells can safely operate at 60°C" — but your documentation says the limit is 45°C.

**B)** A user asks "What's the warranty status for truck T-4421?" and gets an answer about a completely different truck.

**C)** The system worked perfectly last month but now gives vague, unhelpful answers to battery chemistry questions.

**D)** The system consistently formats its JSON output with an extra comma, causing parsing errors in your backend.

<details>
<summary>Self-check</summary>

**A) Hallucination (Generation failure)**
- The model invented or confabulated a temperature — 60°C is not from the context.
- Fix: Strengthen "answer only from context" instruction. Add citation requirement. Add output validation checking claimed numbers against source documents.

**B) Retrieval failure**
- The system retrieved the wrong truck's data, likely due to poor metadata filtering.
- Fix: Add vehicle_id as a metadata filter in retrieval. Don't rely on semantic search for entity-specific lookups — use exact match on vehicle ID first.

**C) Drift (likely model drift or context drift)**
- If the API model was updated silently, behavior changes. Or documents expired.
- Fix: Check if model version changed. Check document store freshness. Run eval suite to quantify degradation. Implement drift monitoring.

**D) Format failure (Generation failure)**
- The model is generating invalid JSON.
- Fix: Use structured output mode (JSON mode) if available. Add JSON schema validation on output. If invalid, retry with error message included. Consider a smaller, well-constrained output format.
</details>

---

## Part B: Architecture Exercises (60 minutes)

### Exercise 4: Design Canvas
Fill out the AI System Design Canvas (from Section 2) for this problem:

> **Problem:** A fleet manager needs an AI assistant that can answer questions about their 200 electric trucks, predict which batteries need replacement in the next 30 days, and generate weekly fleet health reports.

Complete all 10 sections of the canvas. Spend at least 5 minutes thinking before writing.

<details>
<summary>Self-check — key points to have covered</summary>

- **Problem:** Three distinct tasks (Q&A, prediction, reporting) — may need different architectures
- **User:** Fleet managers, non-technical, want fast definitive answers, acceptable latency varies by task (Q&A: 3s, report: 30s, prediction: batch is fine)
- **Data:** Vehicle telemetry (time-series), maintenance records, battery specs (documents), historical degradation data
- **Intelligence:** 
  - Q&A → RAG pipeline (Section 2)
  - Prediction → ML model (not LLM! This is a regression/classification task)
  - Reporting → Prompt chaining (gather data → analyze → format)
- **Context:** System prompt varies by task. Q&A needs vehicle-specific context. Report needs fleet-wide data.
- **Evaluation:** Q&A evals (retrieval + generation metrics), prediction evals (precision/recall on replacement predictions), report evals (completeness checklist)
- **Guardrails:** Safety-critical predictions MUST be flagged. Can't recommend ignoring a failing battery.
- **Cost:** Q&A is pay-per-use, prediction runs batch (cheap), reporting is scheduled (weekly, predictable cost)
- **Key insight:** Prediction should NOT use an LLM — it should use a traditional ML model. The LLM's role is to EXPLAIN the prediction in natural language.
</details>

### Exercise 5: Debug This System
A team built a RAG system for EV maintenance documentation. Here are the symptoms:

1. Answers about NMC811 cells are accurate 90% of the time
2. Answers about LFP cells are accurate only 60% of the time
3. When users ask in German, accuracy drops to 40% regardless of cell type
4. Every Monday, response latency doubles for 2 hours

Propose your investigation plan and likely root causes for each symptom.

<details>
<summary>Self-check</summary>

1. **NMC811 at 90%** — This is your baseline. Decent but not great. Likely some chunking issues or occasional wrong chunk retrieval.

2. **LFP at 60%** — Significantly worse. Most likely causes:
   - Less LFP documentation in the system (data gap)
   - LFP terms overlap with other chemistry terms, causing retrieval confusion
   - Chunking strategy works for NMC811 doc format but breaks on differently-structured LFP docs
   - **Investigation:** Count LFP vs NMC811 documents. Check retrieval quality for LFP queries specifically.

3. **German at 40%** — Clear multilingual failure:
   - Embedding model likely English-only or weakly multilingual
   - Documents probably only in English, but queries in German
   - **Fix:** Use multilingual embedding model (BGE-M3) OR translate queries to English before retrieval OR add German documents

4. **Monday latency doubling** — Operational issue:
   - Weekly batch job running on same infrastructure? (document re-indexing, backup)
   - Monday morning traffic spike + cold cache (weekend cleared cache)
   - Scheduled re-embedding or model warm-up
   - **Investigation:** Check infrastructure logs for Monday morning schedules. Check cache hit rates on Monday AM vs other days.
</details>

### Exercise 6: Design a System From Scratch
Design the complete architecture for this product:

> **Product:** "BatteryGPT" — An AI-powered battery diagnostic chatbot for EV fleet mechanics. Mechanics can describe symptoms in natural language, and the system provides diagnostic steps, parts needed, and estimated repair time.

Requirements:
- Must work on tablets in the workshop (mobile-friendly)
- Must handle greasy-finger typos and informal language
- Safety-critical advice must be flagged
- Must cite which manual section the advice comes from
- Should learn from resolved cases over time
- Budget: $100/day for AI costs

Design:
1. The full system architecture (all 7 layers)
2. Your context engineering strategy
3. Your evaluation plan
4. Your V0 → V1 → V2 progression

<details>
<summary>Self-check — key points</summary>

**Architecture highlights:**
- Mobile-first UI (PWA or responsive web)
- Speech-to-text option for hands-free (greasy fingers!)
- Typo-tolerant query processing (spelling correction before retrieval)
- Hybrid search (BM25 catches exact part numbers despite typos, vector catches semantic meaning)
- Model routing: Haiku for simple lookups, Sonnet for complex diagnostics
- Safety classifier on output — flag anything mentioning high voltage, thermal risk, or structural components
- Citation extraction with manual section references

**Context engineering:**
- System prompt: "You are a master EV mechanic with 20 years of experience..."
- Retrieved context: Relevant manual sections + similar past resolved cases
- Structured output: JSON with diagnosis, steps, parts, time_estimate, safety_flags, citations

**Evaluation:**
- Ground truth: 100 real diagnostic scenarios from experienced mechanics
- Metrics: Correctness (does diagnosis match expert?), completeness (all steps included?), safety (were risks flagged?), citation accuracy

**V0 → V1 → V2:**
- V0: Claude Sonnet + 10 maintenance PDFs + basic RAG. Test with 5 mechanics for 1 week.
- V1: Add hybrid search, reranking, safety flags, citation verification. 50-case eval suite.
- V2: Add resolved case learning, model routing, caching, streaming, production monitoring.

**Budget check at $100/day:**
- 500 mechanic queries/day × ~$0.05/query (routed) = ~$25/day for main queries
- Async eval (5% sampling) = ~$5/day
- Safety validation (parallel) = ~$10/day
- Total: ~$40/day — well within budget with room for growth
</details>

---

## Part C: Self-Assessment Rubric

### Rate yourself honestly (1-5) after completing the 2-day intensive:

| # | Skill | 1=Lost | 3=Understand | 5=Can Teach | Score |
|---|-------|--------|-------------|-------------|-------|
| 1 | Explain how transformers process text | ☐ | ☐ | ☐ | __ |
| 2 | Calculate token costs for a production scenario | ☐ | ☐ | ☐ | __ |
| 3 | Choose between open vs closed models with reasoning | ☐ | ☐ | ☐ | __ |
| 4 | Design a RAG pipeline with chunking and retrieval strategy | ☐ | ☐ | ☐ | __ |
| 5 | Choose the right workflow pattern (chain/route/parallel/orchestrate/eval-opt) | ☐ | ☐ | ☐ | __ |
| 6 | Decide when to use an agent vs a workflow | ☐ | ☐ | ☐ | __ |
| 7 | Classify AI failures (retrieval/generation/system) | ☐ | ☐ | ☐ | __ |
| 8 | Design an evaluation pipeline from scratch | ☐ | ☐ | ☐ | __ |
| 9 | Explain hallucination causes and mitigation strategies | ☐ | ☐ | ☐ | __ |
| 10 | Set up observability for an AI system | ☐ | ☐ | ☐ | __ |
| 11 | Design guardrails and security for AI | ☐ | ☐ | ☐ | __ |
| 12 | Architect a full AI product (7 layers) | ☐ | ☐ | ☐ | __ |
| 13 | Design cost and latency optimization strategies | ☐ | ☐ | ☐ | __ |
| 14 | Plan a V0 → V1 → V2 shipping progression | ☐ | ☐ | ☐ | __ |
| 15 | Know when NOT to use AI | ☐ | ☐ | ☐ | __ |

### Scoring Guide

| Total | Assessment | Action |
|-------|-----------|--------|
| **60-75** | Ready. You think like an architect. | Proceed to Week 1 of the roadmap with confidence |
| **45-59** | Strong foundation. Some gaps. | Re-read weak sections, redo exercises for those areas |
| **30-44** | Good start but needs more time. | Take one more day. Focus on weakest 5 skills |
| **< 30** | Foundations still building. | Re-read all sections slowly. Discuss concepts with Copilot |

---

## Part D: Architect's Quick Reference Card

Save this for daily reference during your 12-week roadmap:

### The AI Architect's Decision Tree

```
New AI task arrives
    │
    ├── Can it be solved WITHOUT AI? ──Yes──▶ Don't use AI.
    │
    ├── Single LLM call sufficient? ──Yes──▶ Optimize the prompt. Done.
    │
    ├── Needs external data? ──Yes──▶ Add RAG.
    │
    ├── Multi-step process needed?
    │   ├── Steps are fixed? ──▶ Prompt chaining
    │   ├── Different input types? ──▶ Routing
    │   ├── Independent subtasks? ──▶ Parallelization
    │   ├── Dynamic subtasks? ──▶ Orchestrator-workers
    │   └── Quality-critical? ──▶ Evaluator-optimizer
    │
    ├── Open-ended, tool-using, adaptive? ──▶ Agent
    │
    └── At each stage ask:
        ├── How do I evaluate this?
        ├── What are the guardrails?
        ├── What's the cost/latency budget?
        └── When should a human step in?
```

### The Debugging Reflex

```
AI gave wrong answer
    │
    ├── 1. Was the right info retrieved? (Check retrieval)
    ├── 2. Was the right info in the context? (Check context assembly)
    ├── 3. Did the model reason correctly? (Check generation)
    ├── 4. Did guardrails catch/cause issues? (Check guardrails)
    └── 5. Is this a new pattern or drift? (Check monitoring)
```

### The Daily Mantra

> **Context over code. Simplicity over cleverness. Evaluation over intuition. Ship over perfection.**

---

## What's Next

You now have the architectural thinking foundation. Return to the [main ROADMAP](../../ROADMAP.md) and begin **Week 1: LLM Fundamentals** — where you'll implement these concepts with actual code.

The difference: you'll now build with **purpose and architecture**, not just tutorials and hope.

---

*Completed: AI Architect Fundamentals 2-Day Intensive*  
*Return to: [README](README.md) | [ROADMAP](../../ROADMAP.md)*

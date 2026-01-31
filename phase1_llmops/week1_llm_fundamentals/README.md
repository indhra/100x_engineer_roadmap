# Week 1: LLM Fundamentals

> **Goal:** Understand how LLMs work at an architect level - not to build one, but to make informed decisions about using them.

---

## 🎯 Learning Objectives

By end of Week 1, you will:
- [ ] Explain transformer architecture without looking at notes
- [ ] Understand tokenization trade-offs (BPE vs WordPiece vs SentencePiece)
- [ ] Reason about embeddings as "meaning coordinates"
- [ ] Make context window decisions for production systems
- [ ] Estimate costs and latency for LLM deployments
- [ ] Choose between open vs closed models for different use cases

---

## 📁 Week Structure

```
week1_llm_fundamentals/
├── README.md                    # This file
├── concepts/
│   ├── 01_transformer_architecture.md
│   ├── 02_tokenization.md
│   ├── 03_embeddings.md
│   ├── 04_context_windows.md
│   ├── 05_inference_vs_training.md
│   └── 06_open_vs_closed.md
├── exercises/
│   ├── tokenization_lab.ipynb
│   ├── embeddings_lab.ipynb
│   └── cost_estimation.py
└── architect_decisions.md       # Your design decisions log
```

---

## 📅 Daily Plan

| Day | Focus | Activity |
|-----|-------|----------|
| Day 1 | Transformer Architecture | Concept deep-dive, mental models |
| Day 2 | Tokenization | BPE/WordPiece/SentencePiece + hands-on |
| Day 3 | Embeddings | Vector space intuition + similarity lab |
| Day 4 | Context Windows | Strategies, trade-offs, production patterns |
| Day 5 | Inference vs Training | Cost/latency analysis |
| Day 6 | Open vs Closed Models | Decision framework |
| Day 7 | Review & Integration | Architect questions, synthesis |

---

## 🧠 Mental Models to Build

1. **LLM as "Compressed Internet + Reasoning Engine"**
   - Not a knowledge base, but a pattern recognizer
   - Trained on internet-scale text, learned to predict next token
   - "Reasoning" emerges from scale + training

2. **Tokens as "Atoms of Language"**
   - The fundamental unit LLMs see
   - Not words, not characters - subwords
   - Different tokenizers = different "atomic structure"

3. **Embeddings as "Meaning Coordinates"**
   - Each token/phrase = point in high-dimensional space
   - Similar meanings = nearby in space
   - Foundation of semantic search

---

## ✅ Completion Checklist

- [ ] All 6 concept documents read and understood
- [ ] Hands-on exercises completed
- [ ] Architect questions answered in `architect_decisions.md`
- [ ] Can explain concepts to someone else

---

*Started: January 31, 2026*

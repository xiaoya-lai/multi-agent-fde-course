# Submission — Implement Semantic Search → Moment Search

**Student:** Jessie Lai
**Module:** 3 — Production Agentic RAG
**Video chosen:** [System Design Course](https://www.youtube.com/watch?v=C842vFY5kRo) (`C842vFY5kRo`) — 125 min, 3,173 transcript cues, ~24k tokens.
**Why this video:** long enough that fixed-size chunking visibly hurts (a 2-hour talk), densely sectioned (scaling, caching, databases, career advice), so "which *moment* answers this" is a real question — exactly where Moment RAG should beat flat semantic search.

---

## Deliverables (both options submitted)

| # | Deliverable | Where | Covers |
|---|---|---|---|
| **Option 1** | Slide deck | [`Semantic_to_Moment_Search_deck.html`](Semantic_to_Moment_Search_deck.html) | Video choice, codebase understanding, Part 1, Part 2, comparison, findings |
| **Option 2** | Demo walkthrough (video) | https://youtu.be/RvJIrtoGF3Y | Live demo of moment-level retrieval + the multi-source/ARGUS realization |
| Working notebook | End-to-end, runnable | [`Semantic_to_Moment_Search.ipynb`](Semantic_to_Moment_Search.ipynb) | Both pipelines + scored evaluation, reproducible |
| Comparison data | Numbers behind the claims | [`systemdesign_data/summary.json`](systemdesign_data/summary.json), `comparison.png` | Baseline vs Moment, same video, same questions |

---

## Part 1 — Baseline semantic-search RAG

Flat pipeline: transcript → fixed-size chunks (≈255 tokens, 145 chunks) → OpenAI embeddings → similarity search → top-k context → generated answer. Notebook **Section 2**. This is the "simple transcript chunking" baseline the assignment asks for — one dense vector per chunk, no query transformation, no reranking.

## Part 2 — Moment RAG (full port of the module codebase)

Ported the module's `src/selfbuilt` pipeline to this single video (notebook **Section 3**), keeping its four moving parts:

- **Semantic chunking** → moment-shaped segments (125 chunks, ≈195 tok, tighter spans) instead of fixed windows (3a).
- **Enrichment / HyDE-at-ingest** — each chunk pre-tagged with the questions it answers, embedded as a third retrieval branch (3b–3c).
- **Hybrid retrieval in Qdrant** — dense kNN + BM25 sparse + question multi-vector, RRF-fused (3d).
- **Query side** — self-query filters → decompose into ~4 sub-queries → retrieve → cross-encoder rerank → resolve to **moments** → cited synthesis with YouTube deep-links (3e–3f).

The key difference: retrieval resolves to a **moment** (a timestamped span you can cite and jump to), not a fixed chunk.

## Comparison — baseline vs Moment RAG (same video, same 4 questions)

| Metric | Baseline | Moment RAG |
|---|---|---|
| Chunks | 145 (avg 255 tok) | 125 (avg 195 tok) |
| Avg top-hit span | **75.5 s** | **59.8 s** (tighter → more precise citation) |
| Avg latency | 1,624 ms | 6,100 ms |
| Sub-queries / question | 1 | ~4 |

Scored, LLM-labeled evaluation on a held-out split is in notebook **Section 4b**.

**Reading:** Moment RAG trades latency for **precision and citeability** — it decomposes the question into ~4 facets and reranks, so it lands on a ~60 s moment you can deep-link to, versus the baseline's broader ~75 s chunk. On a 2-hour talk, "take me to the exact 30 seconds" is the win; the ~4× latency is the cost of decomposition + reranking.

## Findings, limitations, improvements (notebook Section 5)

- **Finding:** moment-level retrieval sharpens *where* the answer is, which matters more as the source gets longer; flat chunking blurs it.
- **Limitation:** higher latency (decompose + rerank), and quality depends on chunk/moment boundaries + transcript accuracy (no speaker diarization here).
- **Improvements:** cache sub-query decomposition, parallelize the rerank, and tune moment boundaries per content type.

---

## Beyond the assignment — multi-form, then production (ARGUS)

The single-video port above is Part 2 of this assignment. I then scaled the same idea along two axes:

1. **Multiple source forms, one index.** Generalized ingestion so **PDF papers, slide decks, and video** land in *one shared vector index* with a `kind` tag and the right locator per type (page / slide / timestamp) — one grounded answer can cite a slide, a paper page, and a video moment together.
2. **ARGUS — production deployment.** Async ingestion (Prefect), crash-safe recovery, cross-source grounded citations, deployed on Fly:
   - **Live app:** https://argus-moment-search-jlai.fly.dev/ (`/demo` is the cross-source search surface)
   - **Evaluation:** [`../../../FDE-01-assignments/Assignment_3_Moment_Search_Scaled/PRODUCT_EVAL.md`](../../../FDE-01-assignments/Assignment_3_Moment_Search_Scaled/PRODUCT_EVAL.md) — recall@10 = 1.0, accept latency p95 = 4.7 ms, zero loss under a mid-ingest crash; the one failing SLA (search latency during concurrent ingest) is documented honestly, not loosened.
   - **Demo:** https://youtu.be/RvJIrtoGF3Y

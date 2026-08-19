# Design Proposal 1

## TL;DR

- **Your current pipeline underperforms because it throws away the visual signal twice**: a VLM collapses each figure/table/chart into a lossy text caption, and a text-only embedder (Titan Text V2) then encodes that caption — so charts, layouts, and dense tables never reach the index as pixels. Switch to true multimodal embeddings that encode page/region images directly, and pass the original images to Gemini at answer time.
- **Recommended architecture: a single-vector multimodal embedding (Cohere Embed v4 on Bedrock) with hybrid BM25 + k-NN search in your existing OpenSearch, a Cohere Rerank 3.5 stage, and retrieved page/region images injected into Gemini Flash 2.5.** This keeps your OpenSearch + AWS + Gemini stack, avoids the 100–1000× vector blow-up of ColPali-style late interaction, and is operationally simplest at 1M+ pages. Reserve a ColPali/ColQwen late-interaction tier (in a multi-vector engine such as Vespa/Qdrant, or via OpenSearch rescoring) for the subset of visually hardest queries where recall is the binding constraint.
- **Cost is dominated by one-time embedding and steady-state Gemini image tokens, not vector storage.** A single 1024-dim float index for 1M pages needs only \~4.6 GB of HNSW RAM; the multimodal embedding backfill is a modest one-time API spend; the recurring cost driver is image tokens fed to Gemini per query (≈258 tokens per 768×768 tile).

---

## Key Findings

1. **Text-only retrieval has a hard ceiling on figure/table/chart-rich docs.** On the ViDoRe visual-retrieval benchmark, an "OCR + BM25 (chunk/embed pipeline)" averages just **17.7 nDCG@5** versus **89.3** for ColQwen2-v1.0 (ColiVara-Eval ViDoRe results) — roughly a 5× gap on exactly the document types you have. Your VLM-caption approach is a variant of "caption-and-index," which the field regards as lossy and prone to "caption hallucination becoming index pollution."
2. **Single-vector multimodal embedders are now strong and API-available on Bedrock.** Cohere Embed v4 (multimodal, 128K context, Matryoshka dims 256/512/1024/1536, native int8/binary output) has been GA on Amazon Bedrock since Oct 2, 2025, "natively processing documents with tables, graphs, diagrams, code snippets, and even handwritten notes… eliminating the need for time-consuming data cleanup" — available on-demand in US East (N. Virginia), Europe (Ireland), and Asia Pacific (Tokyo) plus cross-region inference. Voyage multimodal-3/3.5, Amazon Nova Multimodal Embeddings, Amazon Titan Multimodal G1 (256/384/1024 dims), and Google/Gemini multimodal embeddings are alternatives.
3. **Late-interaction (ColPali/ColQwen) wins on raw retrieval quality but is expensive to store.** On ViDoRe v1, ColQwen2.5-v0.2 scores **89.5 nDCG@5**, ColQwen2-v1.0 **89.2**, and ColPali-v1.3 **84.8** (MINER paper table); the current ViDoRe v1&v2 leaders are Nemotron ColEmbed V2 / ColQwen3 variants (\~84–85 avg). But each page produces \~1,000+ patch vectors: 1M pages × \~1,024 patches × 128 dims × 4 bytes ≈ 524 GB uncompressed, vs \~3–4 GB for a single-vector bi-encoder index. Binary quantization (32×), token pooling ("with a pool factor of 3, the total number of vectors is reduced by 66.7% while 97.8% of the original performance is maintained" — ColPali paper), and PLAID/HPC compression bring this down 30–60×.
4. **OpenSearch can do most of this today, but multi-vector late interaction is a rescoring feature, not first-stage.** OpenSearch supports hybrid search (BM25 + k-NN with normalization/RRF via search pipelines), FAISS/Lucene/nmslib engines, FP16/byte/binary quantization, disk-based (`on_disk`) vector search, and out-of-the-box Bedrock neural/multimodal ingest pipelines. Late-interaction (ColBERT/ColPali) is supported only as a **rescoring/rerank-by-field** step (painless script or Lucene 10.3 `LateInteractionField`), not as scalable first-stage ANN. True first-stage multi-vector retrieval at scale is better served by Vespa, Qdrant, or Milvus.
5. **Feeding images to Gemini Flash 2.5 is cheap per image but adds up per query.** Gemini charges 258 tokens for images ≤384px, and tiles larger images into 768×768 tiles at 258 tokens each; a full page rendered at \~1500×2000 costs roughly 6 tiles (\~1,548 tokens). At Gemini 2.5 Flash input pricing of **$0.30/M input tokens ($2.50/M output, 1,048,576-token context)**, that is a fraction of a cent per page-image, but retrieving 5–10 page images per query is the dominant recurring cost. (Note: industry trackers report Gemini 2.5 Flash scheduled for shutdown around Oct 2026 — plan a migration path to Gemini 3.x.)
6. **Rerankers are the cheapest quality lever and are natively integrated.** Cohere Rerank 3.5 is available on Bedrock (`cohere.rerank-v3-5:0`) and wired into OpenSearch via a `rerank` search-pipeline processor; a VLM-as-reranker (Gemini scoring page images) is a stronger but pricier alternative.

---

## Details

### A. Diagnosis: why the current approach underperforms

Your pipeline does: scanned PDF → VLM (Gemini Flash 2.5) writes a text description of each image → both OCR text and the image description are embedded with Titan Text Embeddings V2 → OpenSearch. There are four compounding failure modes:

- **Double lossy compression of visual content.** A chart contains dozens of quantitative relationships (axis scales, trends, small-multiples, legends). A VLM caption is a single natural-language sentence or paragraph; it discards most of that structure, and then a text embedder discards more. The answer to "what drove the Q3 margin decline?" often lives in a waterfall chart that no sentence in the document states.
- **Caption hallucination becomes index pollution.** When the VLM guesses at an ambiguous figure, the wrong description is embedded as ground truth and is irreversibly baked into the index.
- **OCR linearization destroys 2-D layout.** On scanned documents, OCR merges table columns into nonsense, detaches footnotes from referents, and separates captions from figures. Titan Text V2 embeds this scrambled text.
- **No visual grounding at answer time.** Even when the right page is found, Gemini answers from text, not from the actual pixels — so it fabricates numbers it cannot see.

The fix is architectural, not a matter of tuning chunking: embed the pixels directly (multimodal embeddings), retain page/region images in object storage, and put those images into the generation prompt.

### B. Landscape: single-vector multimodal vs. late interaction

**Single-vector multimodal embedders (one dense vector per page or region).**

| Model | Availability | Dims | Notes / benchmark |
| --- | --- | --- | --- |
| Cohere Embed v4 | Bedrock, SageMaker, Azure | 256/512/1024/1536 (Matryoshka) | Native int8 + binary output; 128K context; \~65.2 MTEB text; strongest managed multilingual (cross-lingual 0.955). No directly published ViDoRe v2 figure; a value of \~53.6 nDCG@5 is inferred from Amazon's relative claim (see Caveats). |
| Voyage multimodal-3 / 3.5 | Voyage API, MongoDB | 1024 | voyage-multimodal-3 averaged \~0.550 nDCG@5 on ViDoRe v2 (ViDoRe v2 paper); voyage-3.5-multimodal \~65.5 nDCG@10 (Gemini Embedding 2 paper). $0.12/M text tokens + per-pixel image charge. |
| Amazon Nova Multimodal Embeddings | Bedrock | 3072/1024/384/256 (Matryoshka) | 58.7 nDCG@5 (3072-dim) on ViDoRe v2 per Nova tech report; 60.6 nDCG@10 per Gemini paper. |
| Amazon Titan Multimodal G1 | Bedrock | 256/384/1024 | Older/weaker; \~$0.00006 per input image + \~$0.0008/1K text tokens; no published ViDoRe v2 score. |
| Google/Gemini multimodal embeddings | Vertex AI | up to 3072 | Gemini Embedding 2 scored 64.9 nDCG@10 on ViDoRe v2; legacy multimodalembedding@001 far weaker (28.9). |
| Jina CLIP v2 / Jina embeddings v4 | API + open weights | up to 2048 | v4 supports single- and multi-vector modes; multilingual + Matryoshka. |
| SigLIP 2, Nomic Embed Vision | open weights | varies | Baselines; weaker on document-style visuals than the above. |

**Late-interaction visual retrievers (multi-vector, ColBERT-style MaxSim).**

- **ColPali v1.3** (PaliGemma backbone): 84.8 nDCG@5 ViDoRe v1. **ColQwen2/2.5** (Qwen2/2.5-VL): ColQwen2.5-v0.2 89.5, ColQwen2-v1.0 89.2 nDCG@5 ViDoRe v1. **ColSmol** (256M/500M) for tight VRAM. Newer leaders (early 2026): Nemotron ColEmbed V2 (Qwen3-VL, 320-dim ColBERT vectors), ColQwen3/Ops-Colqwen3-4B (\~84.8 avg ViDoRe v1&v2), colnomic-embed-multimodal-3b.
- **Storage implications at 1M pages:** ColPali projects to D=128 → \~257.5 KB/page → \~257 GB for 1M pages; ColQwen at 128-dim \~179 GB; higher-dim variants exceed 1 TB. Mitigations: token pooling (pool 3 → 66.7% fewer vectors, 97.8% quality retained), binary quantization + hamming MaxSim (32× storage cut, \~4× compute cut, near-parity after float rerank), and PLAID/HPC-ColPali (up to 32–57× compression).
- **OpenSearch fit:** first-stage multi-vector ANN is not natively scalable; late interaction is supported as a **rescore/rerank-by-field** stage (painless script; Lucene 10.3 `LateInteractionField` maxSim). To use ColPali as *first-stage* retrieval at 1M+ scale you need Vespa (documented "scaling ColPali to billions" with binary + hamming + phased ranking), Qdrant, or Milvus (native multi-vector + MaxSim).

**Verdict:** Late interaction gives the best raw nDCG, but at 1M+ pages it forces a second engine and a 30–500× larger index. Single-vector multimodal embeddings capture the visual signal you are currently losing, fit your existing OpenSearch, and — combined with a reranker — close most of the quality gap at a fraction of the operational cost.

### C. OpenSearch capabilities and constraints (verified)

- **Engines & methods:** FAISS, Lucene, nmslib; HNSW and IVF. HNSW RAM ≈ `1.1 × (4 × dim + 8 × M)` bytes/vector for float (Amazon OpenSearch Service formula). For 1M × 1024-dim, M=16: ≈ `1.1 × (4096 + 128)` × 1M ≈ **4.6 GB**.
- **Quantization:** FP16 (FAISS SQ), byte/int8, and binary (32× compression, hamming space). GPU-accelerated index build (OpenSearch 3.0+; FP16/byte/binary from 3.2).
- **Disk-based vector search:** `mode: on_disk` with scalar (default from 3.6) or binary quantization; big memory savings for slightly higher latency, with full-precision rescoring via `oversample_factor`.
- **Hybrid search:** `hybrid` query combining BM25 `match` + `neural`/`knn` clauses, fused by a normalization-processor search pipeline (min-max/L2) or RRF (k=60 baseline). This directly reuses your existing BM25 + a new multimodal vector field.
- **Bedrock integration:** ML connectors + `text_embedding`/multimodal ingest processors call Bedrock (Titan multimodal, Cohere Embed) on-cluster; documented Cohere-on-Bedrock semantic-search and compressed-embedding (int8/binary) tutorials exist. Rerank via a `rerank` response processor calling Cohere Rerank 3.5 (`cohere.rerank-v3-5:0`) on Bedrock.
- **Late interaction:** rerank-by-field with a SageMaker-hosted ColBERT/ColPali model (`ml_inference` processor generates multi-vectors + a pooled single vector; k-NN retrieves, maxSim rescores).

### D. Ingestion pipeline patterns for scanned documents

1. **Page rendering:** rasterize every PDF page to an image (e.g., \~150–200 DPI, long side \~1,500–2,000 px) → store in S3 with `{doc_id, page_no}` keys. This is the canonical asset for both embedding and generation.
2. **Layout/region extraction (optional but valuable for chart/table-dense docs):** run a layout model to crop figures, tables, and charts as separate regions. Options: **Docling** (DocLayNet layout + TableFormer table structure), **Surya** (detection/layout/reading-order, 90+ languages), **PaddleOCR-Structure**, **MinerU/Marker**, or **AWS Textract** (managed, tables/forms). Embed each region image in addition to the full page for finer-grained retrieval and citation.
3. **Text layer:** keep OCR text (Textract/Surya) for BM25 and as a complementary text vector — hybrid lexical matching still helps for names, IDs, and exact terms.
4. **Embedding:** call Cohere Embed v4 (multimodal) on Bedrock for page images and region crops; optionally keep the existing Titan Text V2 text vector during migration for A/B and fallback.
5. **Indexing:** write one OpenSearch document per page (and optionally per region) containing: OCR text, `page_image_uri` (S3), `mm_embedding` (knn\_vector), provenance metadata, and modality tag.

**Page-level vs. region-level:** page-level is simpler and is what ViDoRe-style models are trained for; region-level (figure/table crops) improves precision and yields tighter citations and smaller images to send to Gemini. Recommended: index page-level as the primary unit and add region crops for table/figure-heavy pages.

### E. Query pipeline

1. Embed the query with Cohere Embed v4 (text input) → query vector.
2. **Hybrid first stage** in OpenSearch: BM25 over OCR text + k-NN over `mm_embedding`, fused via normalization pipeline or RRF; retrieve top \~100–200.
3. **Rerank** top-k with Cohere Rerank 3.5 (Bedrock) using OCR text + captions; optionally a **VLM-as-reranker** stage where Gemini scores the top \~10–20 page images for the hardest queries.
4. **Assemble generation prompt:** fetch the top N (typically 3–8) `page_image_uri` from S3, insert the actual images plus their OCR text and citations into the Gemini Flash 2.5 prompt, instruct the model to ground answers in the visuals and cite `{doc, page}`.

**Gemini image budgeting:** 258 tokens for ≤384px; larger images tile into 768×768 at 258 tokens/tile (tiles ≈ ⌈w/768⌉×⌈h/768⌉). A 1500×2000 page ≈ 6 tiles ≈ 1,548 tokens. Use Gemini's `media_resolution` (LOW≈64 / MEDIUM≈256 tokens per image on 2.5) to trade OCR fidelity for cost; use HIGH only for small-text/OCR-critical pages. Keep to \~3–8 images per prompt for latency and cost; note the image-generation-oriented **Gemini 2.5 Flash Image variant caps at 3 images per prompt** — the standard 2.5 Flash text+vision model accepts many more within its 1M-token context.

### F. Cost & scale analysis (1M pages, order-of-magnitude)

- **One-time multimodal embedding backfill:** dominated by per-image/pixel charges. With Cohere Embed v4 on Bedrock (billed per token; images consume pixel-derived tokens) or Titan Multimodal G1 (\~$0.00006/image → \~$60 per 1M page images) the backfill is a modest one-time spend in the tens-to-low-hundreds of dollars for page-level embedding; region crops multiply by the average regions/page.
- **Index RAM (single-vector):** 1M × 1024-dim float HNSW ≈ 4.6 GB; int8 ≈ \~1.3 GB; binary ≈ \~0.15 GB + rescore. Comfortably fits a modest OpenSearch data-node tier; use `on_disk` + binary if you also index region crops (3–5× more vectors).
- **Index RAM (if late interaction):** 179–524 GB uncompressed for 1M pages; \~6–30 GB with binary+pooling — an order of magnitude more infrastructure and a second engine. This is the core reason to keep late interaction as an optional reranker, not the primary index.
- **Recurring query cost:** Gemini 2.5 Flash input $0.30/M tokens (output $2.50/M). At \~1,548 image tokens/page × 5 pages ≈ 7,740 tokens ≈ $0.0023 per query in image tokens, plus reranker and embedding query costs. This scales linearly with images-per-prompt — the main knob for cost control.
- **Incremental ingestion:** single-vector indexes support cheap CRUD (add/update/delete a doc = one vector). Re-embedding on a model upgrade is a full backfill; mitigate by versioning the embedding field and dual-indexing during cutover. Late-interaction indexes are heavier to update but still CRUD-friendly with token pooling.

### G. Evaluation methodology

- **Build a golden set from your own corpus:** 100–300 real queries (from logs/support tickets) each labeled with the correct page(s)/region(s). Include a dedicated slice of "visual-only" queries whose answers live in charts/tables/diagrams — this is where the new architecture should win most.
- **Retrieval metrics:** Recall@k (is the gold page in top-k — the single most important gate), nDCG@5/@10 (graded, position-weighted), MRR. Evaluate retrieval separately from generation.
- **Generation metrics:** faithfulness/groundedness, citation coverage, hallucination rate, answer correctness (LLM-as-judge with a rubric + human spot-checks).
- **A/B vs. current pipeline:** run the golden set through (i) current Titan-text pipeline, (ii) multimodal single-vector + hybrid + rerank, (iii) + late-interaction rerank. Compare Recall@k/nDCG and downstream answer correctness, plus latency and cost per query.
- **Regression gate:** wire the golden set into CI; block changes to chunking/embedding/rerank that regress Recall@k beyond a preregistered budget. Optionally track ViDoRe v2 as an external sanity check, but prioritize your own domain golden set.

---

## Recommendations

**Stage 0 — Instrument first (1–2 weeks).** Before changing models, build the golden-set eval harness and measure the current Titan-text pipeline's Recall@k and answer correctness, with a visual-only query slice. You cannot justify or tune the migration without this baseline.

**Stage 1 — Single-vector multimodal + hybrid + rerank (primary recommendation).**

- **Embedder:** Cohere Embed v4 on Bedrock, output **1024-dim** (Matryoshka; drop to 512/256 only if eval shows negligible loss). Store int8 for the index, keep float for optional rescore. Rationale: native Bedrock availability, native int8/binary output, 128K context, strong multilingual, no parsing pipeline required.
- **OpenSearch index:** one doc per page with fields — `ocr_text` (text, BM25), `mm_embedding` (knn\_vector, dim 1024, FAISS HNSW, M=16, ef\_construction 256, int8 or `on_disk`+binary if adding region crops), `page_image_uri` (keyword, S3 URI), `doc_id`/`page_no`/`section` (provenance). Add a parallel region-crop index for table/figure-dense pages.
- **Ingestion:** rasterize pages → S3; Docling (or AWS Textract for a fully managed path) for OCR + table/figure crops; Bedrock multimodal ingest processor for embeddings.
- **Query:** hybrid BM25 + k-NN (RRF or normalization pipeline) → Cohere Rerank 3.5 → fetch top 3–8 page images → Gemini Flash 2.5 with images + OCR + citation instructions, `media_resolution` MEDIUM (HIGH for small-text pages).
- **Benchmark to advance:** ship if Recall@5 on the visual-only slice improves materially over baseline (target a large jump given the \~5× text-vs-vision gap on ViDoRe-style tasks — 17.7 → 89.3 nDCG@5 for BM25 vs ColQwen2 on ViDoRe is the illustrative ceiling).

**Stage 2 — Add a late-interaction reranker only where Stage 1 falls short.**

- If the visual-only slice still misses, add ColQwen2.5/ColQwen3 (or Nemotron ColEmbed V2) as a **rerank-by-field** stage in OpenSearch (SageMaker-hosted, multi-vector + pooled vector, maxSim rescore) over the top \~100 hybrid candidates. Use token pooling (factor 2–3) + binary quantization to contain storage.
- **Escalate to a dedicated multi-vector engine (Vespa/Qdrant/Milvus) only if** late interaction must be *first-stage* (i.e., hybrid recall itself is inadequate). Vespa's binary+hamming+phased-ranking recipe is the proven path at 1M–1B scale. This is a significant operational commitment — treat it as a last resort.

**Stage 3 — Optimize cost/latency.** Tune images-per-prompt and `media_resolution`; move to `on_disk`+binary vectors if index RAM grows with region crops; consider Gemini Batch API for offline re-embedding/eval; plan a migration off Gemini 2.5 Flash ahead of its announced \~Oct 2026 shutdown to a Gemini 3.x model.

**Thresholds that change the recommendation:**

- If \>30% of answers depend on fine-grained chart/table reading *and* Stage 1+rerank can't clear your Recall@5 target → commit to late interaction (Stage 2/3).
- If corpus is largely text with occasional figures → Stage 1 alone likely suffices; skip region crops.
- If multilingual/cross-lingual retrieval matters → Cohere Embed v4 is the strongest managed choice; if you can self-host GPUs and want max quality, ColQwen3/Nemotron ColEmbed.

---

## Caveats

- **Benchmark metric mismatch.** Published ViDoRe v2 numbers mix nDCG@5 (Nova report) and nDCG@10 (Gemini Embedding 2 paper) across sources, so cross-model comparisons are not apples-to-apples. Cohere Embed v4 has **no directly published ViDoRe v2 score**; the \~53.6 nDCG@5 figure is back-calculated from Amazon Nova's claim that Nova MME (58.7 nDCG@5 at 3072-dim) beats Cohere Embed 4 by "5.1pps" — treat it as approximate. Amazon Titan Multimodal G1 has no published ViDoRe v2 score at all. Validate on your own golden set — vendor and third-party benchmark rankings do not reliably predict domain performance.
- **ViDoRe scores reflect fine-tuned models on their own query style.** ColQwen's high scores partly reflect fine-tuning on ViDoRe query-page pairs; expect lower absolute numbers on your idiosyncratic corpus.
- **Model availability/pricing move fast.** Gemini 2.5 Flash pricing ($0.30/M in, $2.50/M out) and its scheduled shutdown, Bedrock regional availability for Embed v4 (initially US East N. Virginia, Europe Ireland, Asia Pacific Tokyo + cross-region inference), and per-image charges should be re-verified at implementation time.
- **OpenSearch late-interaction support is nascent.** Native rescoring via Lucene `LateInteractionField` and community plugins exist but are newer and less battle-tested than single-vector k-NN; validate on your OpenSearch version before depending on it.
- **Region extraction adds a failure surface.** Layout models (Surya/Docling/Textract) are imperfect on low-quality scans; budget for OCR pre-processing (grayscale, denoise, thresholding) and treat table-structure extraction as best-effort.
- **Cost estimates are order-of-magnitude.** Actual embedding backfill and query costs depend on average pixels/page, regions/page, images-per-prompt, and Bedrock/Gemini list prices at the time of build.

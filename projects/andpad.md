---
layout: page
title: Andpad — Blueprint Knowledge and 3D Representation
permalink: /projects/andpad/
---

# 🏗️ Andpad

### Blueprint Knowledge and 3D Representation

Multimodal retrieval over scanned drawings, and a hybrid CV + VLM pipeline that turns floor plans into IFC/BIM.

**Date:** 2026

---

## Overview

Two pieces of work, one thesis: **the moment you flatten a visual document into text, you throw away the answer — and the moment you let a language model produce coordinates, you throw away correctness.**

A chart encodes dozens of quantitative relationships. A floor plan encodes topology, dimensions, and drafting conventions. Both are routinely fed to AI systems that immediately collapse them into prose — and then wonder why the answers are vague or fabricated. Both workstreams below attack that same failure from opposite ends: one keeps pixels in the *retrieval* path, the other keeps the vision model out of the *geometry* path.

| Workstream | Status | What it is |
| --- | --- | --- |
| **1. Blueprint knowledge** | Architecture diagnosis + design proposal, with staged rollout and eval plan | Replacing a caption-and-index pipeline with true multimodal embeddings at 1M+ page scale on OpenSearch/Bedrock |
| **2. 3D representation** | Built PoC, demo-complete end to end; licensing and accuracy gaps named up front | Scanned floor plan → IFC4 model via deterministic CV geometry plus a VLM semantic layer, wrapped in an agent UX with narrated conversion, approval-gated edits, renders, and quantity takeoff |

Three convictions run through both:

- **Keep the visual signal.** Embed pixels, and put the actual image in front of the model at answer time — don't let a caption stand in for the page.
- **Confine the model to what it is actually good at.** In the BIM pipeline the vision model may write a room label or an opening type and **never a coordinate**; deterministic code owns every number. Judgment to the LLM, arithmetic to plain code.
- **Name the gap.** Workstream 1 is built around an evaluation harness before any migration. Workstream 2 states plainly what is *not* yet proven — no formal accuracy metrics, a non-commercial model license, a test suite not yet in CI — because a PoC that reports only its successes hasn't tested anything.

---

## Part 1 — Blueprint Knowledge: Multimodal RAG for Visually-Dense Documents

### The diagnosis

The existing production pipeline was: scanned PDF → a VLM writes a text description of each figure → OCR text *and* that description are embedded with a **text-only** embedder (Titan Text V2) → OpenSearch. Retrieval quality on chart- and table-heavy documents was poor, and the instinct in the room was to tune chunking.

The problem isn't chunking. It's four compounding failure modes:

- **Double lossy compression.** A waterfall chart holds dozens of quantitative relationships; a caption is one sentence. The text embedder then discards more. The answer to *"what drove the Q3 margin decline?"* often lives in a figure that no sentence in the document states.
- **Caption hallucination becomes index pollution.** When the VLM guesses at an ambiguous figure, the wrong description is embedded *as ground truth* and irreversibly baked into the index.
- **OCR linearization destroys 2-D layout.** On scans, table columns merge into nonsense, footnotes detach from referents, captions separate from figures — and the embedder faithfully encodes the scramble.
- **No visual grounding at answer time.** Even when the right page is retrieved, the generator answers from text rather than pixels, so it fabricates numbers it cannot see.

So the fix is architectural: **embed the pixels, keep page images as first-class assets, and inject them into the generation prompt.**

### The size of the gap

This isn't a marginal tuning question. On the ViDoRe visual-retrieval benchmark, an OCR + BM25 chunk-and-embed pipeline averages **17.7 nDCG@5** against **89.3** for ColQwen2-v1.0 — roughly a **5× gap on exactly the document class in question**. That number is the argument for spending the migration effort.

### The design space, and why the obvious winner loses

| Approach | Retrieval quality | Index cost at 1M pages | Fits existing OpenSearch? |
| --- | --- | --- | --- |
| Text-only (current) | 17.7 nDCG@5 on ViDoRe-style tasks | ~3–4 GB | Yes |
| **Single-vector multimodal** (Cohere Embed v4, Nova MME, Voyage multimodal-3) | Strong; closes most of the gap with a reranker | **~4.6 GB** HNSW RAM at 1024-dim float | **Yes** |
| Late interaction (ColPali / ColQwen2.5 / Nemotron ColEmbed) | Best raw quality — 84.8–89.5 nDCG@5 on ViDoRe v1 | **179–524 GB** uncompressed (~1,000 patch vectors per page) | No — first-stage multi-vector ANN needs Vespa/Qdrant/Milvus |

Late interaction wins the benchmark and loses the architecture review. At 1M+ pages it forces a **second search engine** and a 30–500× larger index. HNSW RAM is `1.1 × (4 × dim + 8 × M)` bytes per vector, so a single 1024-dim float index is ~4.6 GB — int8 ~1.3 GB, binary ~0.15 GB with rescore. The multi-vector equivalent is two orders of magnitude more infrastructure for a quality delta a cross-encoder reranker largely recovers.

Mitigations exist and are worth knowing precisely — token pooling at factor 3 removes **66.7% of vectors while retaining 97.8% of performance**, binary quantization with hamming MaxSim cuts storage 32×, PLAID/HPC-ColPali reach 32–57× — but they reduce a 500 GB problem to a 6–30 GB problem plus a new engine, not to zero.

### Recommended architecture

Single-vector multimodal embeddings, hybrid retrieval in the OpenSearch cluster that already exists, a reranker, and real page images in the generation prompt:

<p align="center">
  <img src="/assets/andpad/retrieval-architecture.svg" width="100%" alt="Retrieval architecture: ingest rasterizes pages to S3 and embeds page and region images with a multimodal embedder into OpenSearch alongside OCR text; queries run hybrid BM25 plus kNN, rerank, then send the top page images to the generator for a cited answer">
</p>

Deliberate choices worth defending:

- **Cohere Embed v4 on Bedrock at 1024 dims.** Native Bedrock availability, Matryoshka dims (drop to 512/256 only if eval shows negligible loss), native int8/binary output, 128K context, strongest managed multilingual option — and no parsing pipeline required, since it natively ingests tables, diagrams, and handwriting.
- **Page-level as the primary retrieval unit, region crops as a secondary index.** Page-level is what ViDoRe-style models are trained for; region crops buy precision, tighter citations, and smaller images to send to the generator.
- **Keep BM25.** Hybrid isn't ceremony — lexical matching still wins on names, IDs, and exact terms, and it's already in the cluster.
- **Keep the old text vector during migration.** Dual-index and A/B rather than cut over blind; a versioned embedding field makes model upgrades a backfill instead of a rebuild.
- **Late interaction stays a rerank-by-field stage, not first-stage.** OpenSearch supports ColBERT/ColPali-style MaxSim only as rescoring (painless script, or Lucene 10.3 `LateInteractionField`). Escalating to a dedicated multi-vector engine is a last resort, gated on hybrid recall actually being inadequate.

### Where the money actually goes

The instinct is to worry about vector storage. The real cost profile inverts that:

| Cost | Magnitude at 1M pages | Notes |
| --- | --- | --- |
| One-time embedding backfill | Tens to low hundreds of dollars for page-level | Region crops multiply by regions/page |
| Index RAM (single-vector) | 4.6 GB float · ~1.3 GB int8 · ~0.15 GB binary | `on_disk` + binary if region crops land |
| **Recurring: generator image tokens** | **~$0.0023/query** at 5 page images | The dominant steady-state driver |

A page rendered at ~1500×2000 tiles into ~6 tiles of 768×768 at 258 tokens each ≈ 1,548 tokens. So **images-per-prompt is the cost knob**, with `media_resolution` (LOW ≈ 64 / MEDIUM ≈ 256 tokens per image) as the fidelity dial — HIGH reserved for small-text, OCR-critical pages.

### Evaluation, which comes first

**Stage 0 is instrumentation, not modeling.** Before changing an embedder, build the harness and measure the *current* pipeline:

- **A golden set from the real corpus** — 100–300 queries pulled from logs and support tickets, each labeled with the correct page(s), including a dedicated **visual-only slice** whose answers live exclusively in charts, tables, or diagrams. That slice is where the new architecture should win most, and it's the only honest way to size the win.
- **Retrieval measured separately from generation** — Recall@k as the primary gate, plus nDCG@5/@10 and MRR. Generation scored on faithfulness, citation coverage, hallucination rate, and correctness (LLM-as-judge with a rubric, plus human spot-checks).
- **Three-arm A/B** — current text pipeline vs. multimodal + hybrid + rerank vs. + late-interaction rerank — compared on quality *and* latency *and* cost per query.
- **The golden set becomes a CI regression gate**, blocking chunking/embedding/rerank changes that regress Recall@k past a preregistered budget.

### Decision thresholds

The proposal names in advance what would change the recommendation, so the decision isn't relitigated on vibes:

- **>30% of answers depend on fine-grained chart/table reading** *and* Stage 1 + reranker can't clear the Recall@5 target → commit to late interaction.
- Corpus largely text with occasional figures → Stage 1 alone; skip region crops.
- Cross-lingual retrieval matters → Cohere Embed v4 is the strongest managed option; self-hosted GPUs available and max quality wanted → ColQwen3 / Nemotron ColEmbed.

### Caveats I'd state before anyone acts on it

Being the person who flags this is part of the deliverable:

- **Published benchmark numbers are not apples-to-apples** — sources mix nDCG@5 and nDCG@10 across models. Cohere Embed v4 has **no directly published ViDoRe v2 score**; figures circulating for it are back-calculated from a competitor's relative claim and should be treated as approximate.
- **ViDoRe leaders are partly fine-tuned on ViDoRe-style query/page pairs**, so expect lower absolute numbers on an idiosyncratic corpus. Vendor and third-party rankings do not reliably predict domain performance — which is the whole reason Stage 0 exists.
- **Layout/region extraction adds a failure surface** on low-quality scans; treat table-structure extraction as best-effort and budget for OCR pre-processing.
- **Model availability moves fast** — regional Bedrock availability, per-image pricing, and generator lifecycles (Gemini 2.5 Flash has an announced ~Oct 2026 shutdown) all need re-verification at build time, with a migration path planned rather than discovered.

---

## Part 2 — 3D Representation: Floor Plan → BIM with a Hybrid CV + VLM Pipeline

### The question, and the architectural answer

**Can a scanned residential floor plan become a usable BIM model automatically — and what does an agent-driven UX around that conversion feel like?** Both are demonstrated end to end on real plans.

The answer that actually matters is *where the AI belongs*.

The obvious design is to ask a vision model for wall coordinates and extrude them. We built that, measured it, and **rejected it permanently**: coordinates came back with **5–15% error**, along with hallucinated walls and broken topology. A BIM file is a geometric contract, not a draft — 5% on a wall position isn't "close", it's unusable downstream.

So the architecture is a **hybrid split**:

- A **deterministic computer-vision engine owns all geometry** — wall centerlines and thicknesses, room polygons, opening positions.
- A **vision-language model sits on top as a semantic layer** — labeling rooms, classifying doors vs. windows, reading scale annotations, reviewing quality.

That buys the reliability of classical CV with the flexibility of an LLM confined to the judgments it's genuinely good at. **The VLM never writes a coordinate**, and that invariant is enforced by a test rather than by code-review discipline.

<p align="center">
  <img src="/assets/andpad/hybrid-split.svg" width="100%" alt="Comparison: the rejected approach asked a vision model for coordinates and produced 5 to 15 percent error, hallucinated walls and broken topology; the adopted hybrid split gives all geometry to a deterministic CV engine and confines the VLM to room labels, opening types, scale reading and advisory QA">
</p>

Scope was locked at kickoff — single-storey, orthogonal ("Manhattan"), clean residential plans; IFC4 output with walls, doors/windows, slab, and spaces — then deliberately *extended* mid-PoC (via ADR-0001) to cover **model enrichment**, because the demo target is a finished-looking home rather than bare walls. Enrichment follows the same rule: every placement is computed deterministically, and the LLM never invents geometry.

### The pipeline

Eight stages that communicate **only through files on disk** (`masks.npz → plan.json → plan.scaled.json → plan.ifc`), which makes every stage independently inspectable and re-runnable:

<p align="center">
  <img src="/assets/andpad/f2b-pipeline.svg" width="100%" alt="The eight-stage conversion pipeline: parse, vectorize, vlm-enrich, overlay, scale, furnish, ifc and vlm-qa, colour-coded so the six deterministic CV stages own all geometry while the two VLM stages write only labels and types, with file artifacts masks.npz, plan.json, plan.scaled.json and plan.ifc passing between stages">
</p>

Two of the eight stages are the VLM's, and neither of them may touch a number. The other detail worth pulling out of the diagram: **`furnish` is the one deliberate exception to pixel-authority** — furniture is mm-authoritative and back-projected to px for the overlay — and the parse stage uses one pretrained multi-task model (CubiCasa5K) to get wall masks, room segmentation, and door/window heatmaps from a single inference.

### The VLM semantic layer

Four structured-output roles, each returning schema-validated JSON (`messages.parse` + Pydantic) — never free text, never coordinates:

| Role | Stage | What it may write |
| --- | --- | --- |
| **Room labeling** | vlm-enrich | `room.label` only — corrects the CV model's regionally-biased labels using the source image plus an ID-annotated render |
| **Opening classification** | vlm-enrich | `opening.type` only — door / window / generic, from image crops |
| **Scale reading** | scale (opt-in) | A scale *reference* (two pixel endpoints + real mm) fed into the same deterministic converter as manual input — a backstop, never the primary |
| **QA review** | vlm-qa | **Nothing in the model.** An advisory discrepancy report (`qa_report.md`) comparing the overlay against the original drawing |

Three operating principles, each enforced by a test:

- **Degrade, never crash.** Any API or schema failure leaves the CV result in place. With no credentials at all, the whole pipeline runs CV-only.
- **Adapter seam.** Every call goes through a `VlmClient` adapter, so a local or self-hosted VLM can replace the hosted one for confidential deployments without touching role logic. That path is designed in, not bolted on later.
- **Two credential paths.** A first-party API key (default model `claude-opus-5`, switchable by env var) or an AWS Bedrock inference profile, which keeps traffic inside our own account and is usually the easier compliance story. Bedrock takes precedence when set.

### Two deliverables, one demo

| Deliverable | Stack | What it does |
| --- | --- | --- |
| **Floor plan → BIM pipeline** (`f2b` CLI + HTTP API) | Python 3.11 · PyTorch · OpenCV · Shapely · IfcOpenShell · FastAPI | Image → masks → vector model → scaled model → IFC4, exposed as a CLI and as sync and async (SSE-narrated) HTTP endpoints |
| **BIM viewer module** | TypeScript pnpm monorepo · Fastify · ThatOpen Fragments · React · three.js | Stateless IFC → `.frag` conversion service (worker-thread pool) plus embeddable React viewer components — viewer, properties panel, screenshot API |

```
image → [f2b pipeline] → plan.ifc → [converter POST /jobs] → plan.frag → [<BimViewer>]
```

The converter is **stateless and unauthenticated by design**, with an in-memory job registry — a restart loses jobs and clients resubmit — so it must sit behind the host product's gateway. All config is environment variables, and internal paths never leak into responses. The version-coupled viewer dependencies (`three` / `web-ifc` / `@thatopen/*`) are pinned centrally in the workspace catalog because they have to move together.

### The agent experience

Built in phases, each merged to main:

| Phase | Capability | How it works |
| --- | --- | --- |
| **1** | **Narrated conversion** | An async endpoint returns an SSE stream; a read-only Narrator maps pipeline checkpoints to agent events (tool chips, artifacts, metrics), replayable via `Last-Event-ID`. Conversions lacking a scale park and later resume |
| **2a** | **Natural-language edits behind approval gates** | The model runs a tool loop over *deterministic* edit operations — it picks the operation and parameters symbolically, Shapely computes the geometry, and **every mutation requires explicit user approval in the UI** |
| **2b** | **Wall topology** | Room merge and split with adjacency plus sliver/notch guards, on the same approval-gated pattern |
| **3** | **Photoreal rendering** | A `render_image` tool requests a viewer screenshot through a capture gate, then a Gemini image model restyles it. Degrades to a friendly message without a key |
| **4a** | **Estimates** | A pure, LLM-free quantity takeoff over the scaled plan × unit costs → cost chip, estimate panel, and a read-only `get_takeoff` tool |
| **5a** | **Interior enrichment** | Drawing-detected fixtures, deterministic furniture layout, materials, finish floors, door/window detail — plus furniture edit tools where touched items become **pinned** and survive re-layout |
| **5b** | **Exterior enrichment** | Hip roof with a viewer toggle, siding, garage door, porch and walkway aprons |

**Why the edit design matters:** every edit the agent can make is a **named, deterministic, approval-gated operation**. The LLM chooses *what* to do; plain code computes *where*. That keeps the no-VLM-geometry invariant intact even on the interactive path, and makes every mutation auditable.

### Engineering quality

- **356 Python tests, all hermetic** — no model weights, no API keys, no network. The full suite runs in about **5 seconds**, with VLM calls dependency-injected and faked. Fast and offline is what makes a test suite actually get run.
- **JavaScript side:** unit and integration tests per package plus **12 Playwright end-to-end specs** covering the real demo flows — upload → narrate → scale-park → resume → edit → estimate.
- **Invariants are executable.** An `invariants.md` document pairs each golden rule — VLM never touches geometry, degrade-never-crash, pixel-authority, the licensing boundary, takeoff purity — with the specific test or grep that enforces it. A rule nothing checks is a comment, not an invariant.
- **Known gap, stated:** CI runs the JS build, tests, and e2e, but the **Python suite is not in CI yet** — it runs locally per merge.

### Licensing is a first-class engineering constraint

The most consequential finding in the PoC isn't geometric, it's legal — and it was surfaced early rather than at commercialization:

| Component | License | Verdict |
| --- | --- | --- |
| **CubiCasa5K** (code, dataset, weights) | CC BY-NC 4.0 | **Demo/PoC only — not commercially shippable.** Vendored behind a single-importer boundary so it can be swapped |
| MitUNet (production candidate) | Code MIT · checkpoint CC BY-NC | Architecture is clean; the **checkpoint must be retrained** on ~200–500 self-annotated or licensed plans |
| IfcOpenShell · OpenCV · Shapely · scikit-image · PyTorch · FastAPI | LGPL / Apache / BSD / MIT | Clean for commercial use (keep IfcOpenShell dynamically linked) |
| Viewer stack (three.js, ThatOpen) | MIT | Clean |
| web-ifc | MPL-2.0 | File-level copyleft — safe consumed unmodified via npm; flag in client-facing notes |
| Ultralytics YOLO | AGPL | Deliberately avoided, with a repo guard against reintroduction |

The parser being non-commercial is a blocking constraint on shipping, so the architecture **anticipates the swap**: one importer boundary, one retraining task, no rewrite. Naming that in the executive summary rather than burying it is the difference between a PoC that informs a roadmap and one that produces an unpleasant surprise.

**Data egress**, likewise scoped: two external calls exist, both acceptable here because the PoC uses **public plans only** — floor plan images go to the model API (or to Bedrock inside our own account), and viewer screenshots (renders of the 3D model, not the source drawing) go to an image model for restyling. A deployment on confidential client drawings must reconfirm policy or swap in a local VLM through the adapter seam.

### Limitations and known risks

- **Regional bias.** The parser is trained on Finnish plans, so accuracy degrades on other drawing conventions. Client-style plans need testing early; the production fix is retraining on client-domain data.
- **Geometry scope.** Single-storey, orthogonal walls only — curved and diagonal walls and multi-storey buildings are out of scope.
- **Scale is load-bearing.** No IFC without a resolved scale. The UX handles this gracefully by parking and asking, but *fully automatic* conversion depends on the VLM scale reader finding a readable annotation.
- **Revit acceptance is unproven.** Revit is stricter than IFC viewers — wall joins and opening booleans are the classic failures. Output is validated continuously in viewers, but Revit import needs its own pass before any client commitment.
- **No formal accuracy metrics yet.** The planned side-by-side — percentage of walls and openings recovered, dimensional error, measured against a commercial service as benchmark — hasn't been executed. Quality evidence so far is *visual*: overlay alignment plus VLM QA reports on the hero plans. This is the largest evidentiary gap and it is stated as such.
- **Costs are placeholders.** The estimate feature demonstrates the mechanism, not real pricing.

### Commercial context, and the honest build-vs-buy question

| Product | Pricing | Output | Notes |
| --- | --- | --- | --- |
| Plans2BIM (WiseBIM) | €15 / plan | IFC + DXF | 10 s – 3 min per plan |
| MakeaBIM | €0.10 / BIM object | IFC | API on Enterprise tier |

These set the bar: roughly **50% modeling-time reduction with mandatory human cleanup**. The build-vs-buy question is genuine — if a commercial service clears the accuracy bar on *our clients'* plans, then buying the conversion step and building the experience layer on top is a legitimate strategy. What this PoC uniquely demonstrates is that experience layer, plus full control of the pipeline.

### Production roadmap

1. **Replace the parser** — retrain an MIT-licensed architecture on client-style plans; multi-model fusion later.
2. **Confidentiality path** — local or self-hosted VLM behind the existing adapter seam; the Bedrock path already works today.
3. **Accuracy evaluation** — execute the deferred metrics table on client plans, including the commercial-service benchmark and a Revit acceptance pass.
4. **Feature backlog** — construction sheets, prompt-started designs, real cost data for takeoff, multi-storey and non-Manhattan support.
5. **Hardening** — Python suite into CI, self-hosted Fragments worker, single-parse optimization.

---

## 🛠 Technical Stack

- **Retrieval:** Amazon OpenSearch (hybrid BM25 + k-NN, FAISS HNSW, int8/binary quantization, `on_disk` mode), Cohere Embed v4 + Rerank 3.5 on Bedrock, S3, Gemini Flash 2.5 vision, Docling / Surya / Textract for layout + OCR.
- **CV geometry engine:** Python 3.11, PyTorch, OpenCV, scikit-image, Shapely, IfcOpenShell (IFC4), FastAPI with SSE.
- **VLM semantic layer:** Claude (`claude-opus-5`) via first-party API or AWS Bedrock inference profile, structured output with Pydantic schemas, behind a swappable client adapter.
- **Viewer & demo:** TypeScript pnpm monorepo, Fastify, ThatOpen Fragments, web-ifc, three.js, React, Vite.
- **Evaluation & tests:** golden-set retrieval harness (Recall@k / nDCG / MRR), LLM-as-judge groundedness rubrics, VLM QA overlay reports, 356 hermetic Python tests, 12 Playwright e2e specs, executable invariants doc.

---

## Lessons I took away

- **Lossy preprocessing is invisible in the logs and fatal in the results.** A caption pipeline looks healthy end to end — documents ingested, vectors written, answers returned. Nothing surfaces the fact that the information needed to answer the question was destroyed at step two. You find it by measuring retrieval on a visual-only query slice, or you don't find it at all.
- **Don't let a language model produce geometry.** Asking a VLM for coordinates gave 5–15% error, hallucinated walls, and broken topology, so that approach was rejected permanently rather than tuned. The durable split is judgment to the model, arithmetic to deterministic code — and it holds even in the interactive path, where the agent picks a named operation and Shapely computes the result.
- **An invariant nothing checks is just a comment.** Pairing every golden rule with the specific test or grep that enforces it is what kept "the VLM never touches geometry" true as the system grew a chat interface, an edit loop, and an enrichment stage.
- **Pick the architecture that fits the operations budget, not the one that wins the benchmark.** Late interaction is genuinely better at retrieval and genuinely wrong at 1M pages on one team's existing cluster. Naming that tradeoff explicitly — with the storage arithmetic and the threshold that would reverse the decision — beats picking the leader off a leaderboard.
- **Licensing is architecture.** A CC BY-NC parser makes an otherwise-working pipeline unshippable. Finding that in week one and designing the swap boundary around it is cheap; finding it during commercialization is not.
- **Publish what isn't proven.** The most useful lines in the PoC write-up are the ones admitting no formal accuracy metrics yet, unvalidated Revit import, placeholder costs, and a test suite still outside CI. That list is what a stakeholder actually needs in order to decide what to fund next.

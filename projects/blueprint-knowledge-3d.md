---
layout: page
title: Blueprint Knowledge and 3D Representation
permalink: /projects/blueprint-knowledge-3d/
---

# 📐 Blueprint Knowledge and 3D Representation

### Multimodal retrieval over scanned drawings, and a VLM agent that turns floor plans into IFC/BIM

**Date:** 2026

---

## Overview

Two pieces of work, one thesis: **the moment you flatten a visual document into text, you throw away the answer.**

A chart encodes dozens of quantitative relationships. A floor plan encodes topology, dimensions, and drafting conventions. Both are routinely fed to AI systems that immediately collapse them into a sentence of prose — and then wonder why the answers are vague or fabricated. Both workstreams below attack that same failure from opposite ends: one keeps pixels in the *retrieval* path, the other keeps the vision model out of the *geometry* path.

| Workstream | Status | What it is |
| --- | --- | --- |
| **1. Blueprint knowledge** | Architecture diagnosis + design proposal, with staged rollout and eval plan | Replacing a caption-and-index pipeline with true multimodal embeddings at 1M+ page scale on OpenSearch/Bedrock |
| **2. 3D representation** | Built PoC, quality measured rather than asserted | Raster 間取り図 → real IFC model, refined by bilingual chat, opens in Revit |

Three convictions run through both:

- **Keep the visual signal.** Embed pixels, and put the actual image in front of the model at answer time — don't let a caption stand in for the page.
- **The model produces a schema, not an artifact.** Constrain generation to a validated structure and let deterministic, unit-tested code produce the final output.
- **Measure the claim.** Both workstreams are built around an evaluation harness, and both report the axis where they're weak instead of hiding it.

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

```
INGEST                                    QUERY
scanned PDF                               user query
  ├─ rasterize page (150–200 DPI) → S3      ├─ embed (Cohere Embed v4, text)
  ├─ layout model → figure/table crops      │
  │    (Docling · Surya · Textract)         ├─ HYBRID first stage (top 100–200)
  ├─ OCR text layer  ─────────────┐         │    BM25(ocr_text) ⊕ kNN(mm_embedding)
  └─ Cohere Embed v4 (multimodal) │         │    fused by RRF or normalization pipeline
        on page + region images   │         ├─ Cohere Rerank 3.5 (Bedrock)
                                  ▼         │    [+ optional VLM-as-reranker on hard queries]
        OpenSearch doc per page/region      └─ fetch top 3–8 page images from S3
        { ocr_text, mm_embedding(knn),           → Gemini Flash 2.5 with IMAGES + OCR
          page_image_uri, doc_id, page_no }        + cite {doc, page}
```

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

## Part 2 — 3D Representation: Floor Plan → BIM with a VLM Agent

### The question, and the honest answer

**Can an AI agent convert a residential floor plan image into a real, correctable BIM model — with nobody touching a CAD tool?**

Yes, with one measured caveat.

Upload a raster 間取り図. In about a minute you get a finished-looking house — colored walls on the Japanese module grid, a gable roof with eaves, glazed windows, wood floors, furniture in the rooms — as a genuine **IFC file that opens in Revit with its colors and millimeter units intact**. You then refine it by chatting, in Japanese or English (*"外壁を濃紺に塗って"*, *"delete the wall between the LDK and the bedroom"*, *"total floor area?"*), and every edit regenerates the same IFC.

**The caveat is the PoC's central research finding:** extraction is *strong on structure* — walls, topology, room labels, dimensions — and *weak on opening placement*. Door and window positions jitter run-to-run by roughly **±20% along their wall**, and one evaluation run **hallucinated an interior door** that wasn't on the plan. That's measured by a built-in eval script that renders overlays for direct comparison, not asserted from vibes. A known, quantified limitation beats a hidden defect.

### Why the naive approach fails

"Ask a vision model for coordinates and extrude them" produces a wobbly mess: walls inset by half a thickness, dimensions ~1% short, lines near-but-not-quite orthogonal. Three choices turn that into credible geometry.

**a) A semantic intermediate representation, not a mesh.** The AI never produces geometry. It produces a **Plan Model** — and everything downstream is deterministic and unit-tested:

```
PlanModel
└── storeys: [Storey]                    # 1 today; a 2nd storey is additive
    ├── name, ceiling_height_mm
    ├── walls:     [Wall{id, start(x,y), end(x,y), thickness_mm}]   # centerlines, mm
    ├── openings:  [Opening{id, wall_id, kind: door|window,
    │                       offset_mm, width_mm, sill_mm, height_mm}]
    ├── rooms:     [Room{id, label, polygon: [(x,y)]}]
    ├── furniture: [Furniture{id, kind, position, rotation_deg}]     # 9-kind catalog
    ├── finishes:  Finishes{wall_colors, floors}
    └── roof:      Roof{pitch_deg, overhang_mm, color}
```

Millimeters everywhere, origin at the building's bottom-left exterior corner. **Walls are centerlines and openings are offsets along them**, so moving a wall moves its openings for free. Missing finishes fall back to Palette defaults at generation time, so operations never have to remember to paint things — a small convention that removes a whole class of bug.

**b) Deterministic post-processing that exploits domain structure.** Japanese residential plans sit on the 910mm 尺 module, so coordinates snap to its 455mm half-module — and that converts jitter into *correctness*. Raw model output of `60 / 7220 / 5400` reliably becomes exactly `0 / 7280 / 5460`. Three passes, in order:

| Pass | Rule | Why |
| --- | --- | --- |
| **Orthogonalize** | Walls within 7° of an axis become axis-aligned | Near-orthogonal is always a drafting artifact |
| **Join** | Endpoints within 250mm cluster to their centroid | Walls that should share a corner, don't quite |
| **Snap** | Every coordinate to the 455mm half-module | Turns jitter into exact modular dimensions |

Opening offsets are then remapped to **preserve absolute plan position** while their wall moves underneath them, clamped inside the wall, and dropped if wider than it.

**c) A bounded operation set for edits.** The refinement agent cannot write arbitrary code or emit coordinates freely. It calls **18 validated pure functions** (`PlanModel → PlanModel`). An invalid edit returns a domain error the agent can read and retry against — *"No wall with id 'w-x'. Walls in the model: …"*.

The result: **two LLM calls in the entire system.** Everything else is deterministic, tested Python.

### The pipeline

| Stage | What happens | Notable decision |
| --- | --- | --- |
| **Extraction** | One vision call via `messages.parse` with a Pydantic schema; calibrates scale from dimension text → 帖 labels → the 910mm module, in that reliability order | The schema *is* the domain model — one definition, no drift, and **zero JSON parsing or repair code** |
| **Post-processing** | Orthogonalize → join → snap → opening remap | The 455mm snap is the highest-leverage line in the system |
| **Palette** | Exterior walls (centerline on the footprint perimeter — post-snap this test is *exact*) get siding; floors from bilingual label heuristics (和室/畳 → tatami, 浴/トイレ/WC → tile, else wood) | Named `IfcMaterial` records are the join key for the viewer's texture layer *and* independently useful to external BIM tools |
| **Auto-furnish** | Rooms the plan didn't furnish get filled from a 9-item catalog, placed against a wall, clear of door/window spans and other furniture | Rule-based, and nothing outside the catalog can exist |
| **IFC generation** | `IfcOpenShell` authoring: walls with `IfcOpeningElement` cuts, `IfcDoor`/`IfcWindow` via `IfcRelFillsElement`, `IfcCovering` floor plates, `IfcSlab`, `IfcRoof` gable prism, correctly-typed `IfcFurniture`/`IfcSanitaryTerminal`, deduplicated `IfcSurfaceStyle` | Real IFC authoring, not a text templater |
| **Staged Reveal** | The UI draws findings back over the original image — walls → openings → rooms → 2D fades to 3D — driven by NDJSON **Narration** events | Narration is **computed from Plan Model data, not generated**: no extra LLM call, no latency, no hallucination risk. It never enters the agent's context |

The **#1 trap for future contributors**, documented because it cost real time: two unit conventions coexist inside IfcOpenShell — `ifcopenshell.api.geometry` helpers take SI **meters** and convert internally, while `ShapeBuilder` takes project units (**mm**) raw. Both were verified empirically with probe scripts, and tests pin the behavior.

### The refinement agent

A **manual tool-use loop** rather than the SDK's runner — deliberate, because the model client is the faking seam for tests, and a manual loop keeps every request and response visible to them:

```
history += user message (+ Plan Model JSON snapshot in <plan_model> tags;
                         + the original plan image, first turn only)
loop (max 8):
    response = model(system, tools, history)
    if stop_reason != tool_use: break
    for each tool_use: apply operation  (errors → tool_result is_error, agent reacts)
if anything mutated: regenerate IFC
```

Keeping the original plan image in the conversation is what makes *"図面の通りに戻して"* ("restore it as drawn on the plan") work without re-running extraction. The **operation registry** is the single place an operation exists — description + JSON schema + pure-function handler + `mutates` flag — so the agent's tool list derives from it and adding a capability is one entry.

| Group | Operations |
| --- | --- |
| Walls | `add_wall`, `delete_wall`, `move_wall`, `set_wall_thickness` |
| Openings | `add_opening`, `delete_opening`, `move_opening`, `resize_opening`, `set_opening_kind` |
| Rooms & storey | `relabel_room`, `set_ceiling_height` |
| Appearance | `paint_wall`, `set_room_floor`, `configure_roof` |
| Furniture | `add_furniture` (auto-places if no position given), `move_furniture`, `delete_furniture` |
| Read-only | `get_model_summary` — areas in m² and 帖, counts, finishes, roof, furniture per room |

### The four load-bearing decisions

Each recorded as an ADR, because these are the choices everything else implements:

- **ADR-0001 — The viewer renders the IFC itself.** There is no separate 3D scene built from the Plan Model; the browser loads the exact bytes the backend generated. One geometry path, so the screen cannot drift from the exported file — which makes the BIM claim *literal*. The price, stated: a mandatory Python backend and a sub-second IFC round trip on every mutating edit.
- **ADR-0002 — Appearance is IFC-native; polish is viewer-side.** Anything describing the *building* (roof, door panels, glazing, paint, floor materials) lives in the IFC as surface styles so it survives into Revit. Anything describing the *presentation* (background, shadows, roof toggle) is viewer-side.
- **ADR-0003 — Textures keyed by the file's Material names.** Run **spike-first with an explicit timebox and a written go/no-go**: the spike found the viewer's batched geometry carries no UV attribute (so `material.map` was impossible) and same-colored elements batch into one mesh (so per-element override was impossible). What worked was `onBeforeCompile` injection of world-space **triplanar** sampling, verified with a checkerboard before any real texture was drawn. The ADR knowingly relaxes ADR-0001 for appearance only — and records the resulting limitation, that elements recolored away from Palette defaults render flat.
- **ADR-0004 — Model spend runs through a Bedrock application inference profile.** Organizational policy requires spend attributable to a cost centre, which only an application inference profile carries. The consequence worth recording: **there is deliberately no default model id.** A missing `BIM_AGENT_MODEL` refuses to start the server, because the tempting fallback to a public profile is exactly the wrong failure mode — it would keep working while quietly running a different model on untagged spend.

### Findings

1. **Structure extraction is strong** — 5/5 walls with correct topology, all rooms correctly labeled, and after grid snapping the exact 7280×5460mm dimensions. Western-style and English-labeled plans also work via the bilingual heuristics.
2. **Opening placement is the weak axis** — ~±20% run-to-run jitter along the wall, plus one hallucinated interior door. The eval overlay makes such errors visible at a glance.
3. **The 455mm grid snap is load-bearing** — the single highest-leverage piece of code in the system.
4. **Chat refinement is dependable** — the agent resolves natural-language references ("exterior walls", room names) to element operations correctly, works bilingually, and plan-grounded restore requests reproduce geometry from the image.
5. **The IFC round trip holds** — colors authored as IFC surface styles render in the browser *and* persist in the downloaded file.
6. **Structured output eliminated an entire error class** — because the Pydantic schema is enforced by the SDK, there is no JSON repair code anywhere. The historical failure mode of LLM-to-structured-data pipelines simply doesn't exist here.

### Engineering quality

**136 automated tests, all passing** — roughly 0.8 lines of test per line of backend source. The strategy is **seam-based**, with no mocking below one boundary:

```
┌──────────────────────────────────────────────────────────────────┐
│ HTTP seam (primary): FastAPI TestClient drives real routes.      │
│ The ONLY fake is the model client. Post-processing, Palette,      │
│ auto-furnish, operations, IFC generation and session state all    │
│ run for real.                                                     │
├──────────────────────────────────────────────────────────────────┤
│ Pure-function seams: post-processing, every operation, palette    │
│ classification, quantities, roof geometry, furniture placement.    │
├──────────────────────────────────────────────────────────────────┤
│ IFC parse-back seam: generated bytes are re-parsed with           │
│ IfcOpenShell and asserted structurally — element counts, fill     │
│ relationships, style colors, extrusion depths, mm units.          │
├──────────────────────────────────────────────────────────────────┤
│ LLM quality: NOT a CI gate. An opt-in eval script runs real       │
│ Extraction over samples/ and writes a report + overlays.          │
└──────────────────────────────────────────────────────────────────┘
```

Two techniques worth stealing:

- **A negative assertion made cheap and exact.** *"No IFC regeneration happened"* is asserted by byte-identical `GET /api/ifc` responses — any regeneration would mint new random GlobalIds. That's how read-only chat turns are proven side-effect-free.
- **IFC asserted structurally by re-parsing, never by golden file.** Random GlobalIds would make byte comparison flap forever.

And a deliberate gap, named as such: **LLM output quality is not a CI gate.** Correct for a PoC — it would make the suite slow, costly, and flaky — but it means prompt changes are currently unguarded, which is why wiring the existing eval script into a regression gate is the top next step.

### Limitations and scope boundaries

Out of scope by choice: cost estimates and construction documents; hip roofs (寄棟), ceilings, per-face wall painting; stairs, columns/beams, site, multi-storey (the schema is storey-additive, so a second floor is additive work, not a rewrite); CAD input (DWG/DXF/JWW) and hand-drawn sketches; robustness on uncurated uploads — *graceful failure* on bad input is in scope, *good results* are not; persistence, auth, deployment, multi-user.

Behaviors worth knowing before a demo: the session is in memory, so a restart clears it; extraction takes ~30–60s and chat turns ~10–60s, both live model calls; re-uploading the same plan can produce a slightly different model; wall paint applies per wall to both faces, so painting a room with an exterior wall changes its outside face too (documented in the operation description so the agent warns the user).

**Next, ordered by value-to-effort:** make the eval script a regression gate for prompt changes; attack opening-placement variance directly (extract twice and reconcile, or add a self-check pass); second storey and stairs; broaden the sample set.

---

## 🛠 Technical Stack

- **Retrieval:** Amazon OpenSearch (hybrid BM25 + k-NN, FAISS HNSW, int8/binary quantization, `on_disk` mode), Cohere Embed v4 + Rerank 3.5 on Bedrock, S3, Gemini Flash 2.5 vision, Docling / Surya / Textract for layout + OCR.
- **BIM agent:** Python 3.12, FastAPI, Uvicorn, IfcOpenShell 0.8.5, Pydantic v2, `anthropic[bedrock]` structured output + streaming, Amazon Bedrock application inference profiles.
- **Frontend:** React 19, TypeScript, Vite 8, ThatOpen Components / web-ifc / three.js.
- **Evaluation:** golden-set retrieval harness (Recall@k / nDCG / MRR), LLM-as-judge groundedness rubrics, extraction-overlay eval reports, 136-test seam-based backend suite.

---

## Lessons I took away

- **Lossy preprocessing is invisible in the logs and fatal in the results.** A caption pipeline looks healthy end to end — documents ingested, vectors written, answers returned. Nothing surfaces the fact that the information needed to answer the question was destroyed at step two. You find it by measuring retrieval on a visual-only query slice, or you don't find it at all.
- **Don't let the model produce the artifact.** Both workstreams put a validated schema between the model and the output — a Plan Model in one, a retrieval unit with provenance in the other. Constrained generation plus deterministic post-processing beats a bigger prompt, and it's the difference between geometry that looks hand-wobbled and geometry that snaps to the exact module.
- **Pick the architecture that fits the operations budget, not the one that wins the benchmark.** Late interaction is genuinely better at retrieval and genuinely wrong at 1M pages on one team's existing cluster. Naming that tradeoff explicitly — with the storage arithmetic and the threshold that would reverse the decision — is more useful than picking the leader off a leaderboard.
- **Instrument before you migrate.** The eval harness is the cheapest stage and the one most often skipped. Without a baseline you can neither justify the change nor tell whether it worked.
- **Publish the weak axis.** The BIM PoC's most valuable output isn't the house that appears in a minute — it's the measured ±20% opening jitter and the one hallucinated door, because that's what tells a stakeholder what to fix next. A feasibility PoC that reports only its successes hasn't tested anything.

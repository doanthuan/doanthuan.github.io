# 🚀 LayerProof

### Multi-Agent AI System for Generating Professional Presentations

**Date:** 2026

**URL**: [https://layerproof.app/](https://layerproof.app/)
---

<p align="center">
  <img src="/assets/layerproff/sample1.webp" width="32%" alt="LayerProof generated sample 1">
  <img src="/assets/layerproff/sample2.webp" width="32%" alt="LayerProof generated sample 2">
  <img src="/assets/layerproff/sample3.webp" width="32%" alt="LayerProof generated sample 3">
</p>

## Overview

This platform turns a user prompt plus a handful of reference assets into
finished creative — social-media campaigns across TikTok / Instagram /
Threads / Facebook / LinkedIn, full slide decks, blog posts, RFC/PRDs,
mindmaps, and more. Each generation is a multi-step pipeline: outline →
research → image composition → per-platform variants → publish. A single
campaign can fan out into dozens of LLM calls and image-generation jobs
across several minutes, so the hard part was never the prompts — it was
making the orchestration **survive real infrastructure** (pod restarts,
rate limits, GPU OOMs, flaky upstreams) without duplicating work or
corrupting state.

I owned three pillars of the system end to end:

1. **The agentic workflow engine** — an event-sourced durable-workflow
   runtime that lets a pipeline resume from any point after a failure, with
   a strict determinism contract that prevents replay corruption.
2. **The image generation pipeline** — a two-path image system built on
   Gemini 3.1 Flash Image with a vision-model prompt builder, iterative
   prompt engineering (v1 → v5), and an evaluation harness that lets me
   A/B prompts offline before shipping.
3. **The image-processing worker** — a custom Python worker on ECS Fargate
   that co-hosts remote-API processors (Gemini, Textract, OpenAI), heavy
   CPU processors (Playwright, vtracer), and GPU models (CyberAgent LayerD,
   BiRefNet matting) on a single weighted-semaphore scheduler.

Technical audience below — this is a deep-dive, not a sales sheet.

## Stack at a glance

- **Backend:** Kotlin 2.0.21, Micronaut, Exposed ORM, PostgreSQL, Redis
  (Redisson cluster), RabbitMQ, SQS.
- **Workers:** Python 3.12, `uv`, multiprocessing + ThreadPoolExecutor,
  ONNX Runtime, PyTorch + Transformers, OpenCV, Playwright, vtracer,
  `ddtrace` for observability.
- **AI:** Gemini (Vision + Image Edit), OpenAI, Anthropic Claude. Multi-
  provider abstraction so workflows don't know which model they got.
- **Real-time:** Centrifugo WebSocket server pushing typed live-object
  updates to the browser — no polling.
- **Infrastructure:** AWS ECS Fargate (CPU + GPU pools), SQS, S3,
  LocalStack for local dev, Firebase Auth.

---

## Pillar 1 — Agentic Workflow Engine

### The problem

A "generate one social-media campaign" request spawns 30+ activities:
topic brainstorm, web search, chart rendering, reference-image download,
canvas composition, vision analysis, image generation, caption generation,
aspect-ratio variants, publish per platform. Any of them can fail
transiently. Three things had to be true:

1. If a pod dies mid-campaign, the workflow must resume — not restart.
2. No side effect (API call, DB write, S3 upload) may run twice.
3. The user must see progress in real time, without the orchestration
   layer caring about the transport.

Temporal would have worked but would have dragged in a non-trivial
operational surface. I built `module-durable-job` instead — a lean,
event-sourced workflow engine backed by the stack we already run.

<p align="center">
  <img src="/assets/layerproff/mermaid-diagram.png" width="90%" alt="Durable workflow engine architecture">
  <br/>
  <em>Durable workflow engine — workflow ↔ activity ↔ queue topology.</em>
</p>

### Design

**Event-sourced activity log.** Every activity transition (started,
delayed, completed, failed) is appended to a Postgres event log keyed by
`(activity_id, sequence_no)`. Workflows replay themselves by re-running
the orchestrator function and short-circuiting at each `context.run(...)`
by reading the cached outcome from the log. On replay, a workflow is a
pure function of its input and its event log — which is why the
determinism rules below are not optional.

**Pluggable queue backends.** Postgres is the authoritative state; the
job queue is swappable — `PostgresActivityQueue`, `SqsActivityQueue`,
`RabbitMQActivityQueue`, `RedisActivityQueue`. Each supports the same
two-stage protocol: `peek` (lease a message, start a visibility timeout)
and then `remove` (on success) or `returnToQueue` (on failure or
no-capacity). Distributed Redisson fair locks prevent two workers from
racing the same activity.

**Parent-child scheduling.** A workflow that launches N child activities
doesn't busy-wait; it suspends until children complete. Each child,
on completion, atomically enqueues its parent for re-entry. This is how
a `SocialCampaignGenerationWorkflow` can fan out into 20 platform-post
generations and only resume when the last one lands.

### The determinism contract

This is the rule set I enforce codebase-wide — violations *look* like
they work until a pod dies mid-flight and you get duplicated DB rows:

| Concern        | Not allowed                    | Required                              |
| -------------- | ------------------------------ | ------------------------------------- |
| Timestamps     | `Instant.now()`                | `context.now()`                       |
| UUIDs          | `UUID.randomUUID()` in workflow | Generated inside a leaf activity     |
| Delays         | `kotlinx.coroutines.delay()`   | `context.delay()`                     |
| Side effects   | Direct service calls from workflow | Wrapped in an idempotent activity |
| Logging        | `Logging` on workflow classes  | Logging only inside activities        |
| Exception flow | Catching `Exception`/`Throwable`/`ActivityPendingException` | Let pending exceptions propagate |

Workflows have **zero dependencies injected** — they are pure
orchestrators. Activities are where side effects live, and every
activity starts with a check-before-act idempotency guard. "Only
workflows, never activities, get replayed against a frozen log" was the
single insight that made the whole system coherent.

### Live state

Workflow progress reaches the browser via typed `LiveObjectUpdate` data
classes flowing through Centrifugo. I explicitly banned `mapOf("status"
to "IN_PROGRESS", "progres" to 50)` — one typo used to silently drop
progress for days — and replaced it with a `LiveObjectUpdateMapper` that
turns a strongly-typed update class into a patch map at the last
possible moment. Compile-time type checking on every status update.

### The agent layer on top

Once the durable runtime was stable, it became the substrate for real
agents. `ChatModelInvocationActivity` wraps an LLM call (OpenAI / Claude /
Gemini) with tool-use orchestration: the model proposes a tool call,
the workflow dispatches a subagent or a tool workflow, the result is
appended to the conversation, the model is re-invoked. Tools are full
workflows themselves:

- `RunCodeToolWorkflow` — dispatches Python execution to a sandboxed
  Lambda.
- `FetchWebpageToolWorkflow` — scrapes with a worker processor.
- `DispatchSubagentToolWorkflow` — spawns another agent with its own
  identity and toolset.
- `GenerateImageToolWorkflow` / `EditImageToolWorkflow` — the image
  pipeline described in Pillar 2, invoked as agent tools.

Because tools are workflows, **agent behavior is fully replayable**: a
model's tool-call decisions are recorded alongside their outcomes, so a
crashed agent resumes exactly where it left off, same decisions, same
intermediate state. This is the part I'm most proud of — agentic loops
that are not "best-effort retries" but provably-deterministic
continuations.

### Workflows shipped (selected)

`SocialCampaignGenerationWorkflow`, `SocialOutlineGenerationWorkflow`,
`BatchSocialPostImageGenerationWorkflow`, `GeneratePlatformPostWorkflow`,
`GenerateAspectRatioVariantWorkflow`, `GenerateImageVariationsWorkflow`,
`PublishTikTokPostWorkflow`, `PublishThreadsPostWorkflow`,
`PublishInstagramPostWorkflow`, `PublishFacebookPostWorkflow`,
`PublishLinkedInPostWorkflow`, `SlideDeckGenerationWorkflow`,
`SlideImageEditAgenticWorkflow`, `LayerDecompositionWorkflow`,
`AgenticGenerationWorkflow`, `MindmapAgentWorkflow`, `RunCodeToolWorkflow`,
`DispatchSubagentToolWorkflow`, and dozens more.

---

## Pillar 2 — Image Generation Pipeline

### The problem

Gemini's raw text-to-image output is impressive in isolation, but for a
brand it fails in predictable ways: the hero product morphs, reference
images leak unintended subjects, 9:16 Story layouts put text where
Instagram's UI covers it, and "premium feel" gets interpreted as
"random gold gradient." I needed image generation that **preserves a
hero, respects platform safe zones, and is reproducible enough to A/B
test offline.**

### Two-path strategy

The pipeline branches based on whether a request has reference assets:

- **Path A — Canvas-based (with references).** Reference images are
  packed into a `data_canvas` (logo, product shot, mood board, charts
  from `code_execution`). The canvas is sent to Gemini Vision with a
  structured prompt produced by `CanvasAnalysisPromptBuilder`, which
  extracts concrete visual detail (materials, lighting direction, colour
  palette, logo position) and emits a single image-editing prompt. That
  prompt + source image feed `GeminiImageEditActivity`.
- **Path B — Direct text-to-image.** No references → straight to
  `GeminiImageGenerationActivity`.

The selection is a single conditional at the top of
`BatchSocialPostImageGenerationWorkflow`. Most creative work goes through
Path A because that's where brand identity is preserved.

### `CanvasAnalysisPromptBuilder`

The prompt builder is a Kotlin object assembled from four sections:
`Header` (target platform + aspect ratio), `Context` (topic / core
message / visual direction), `Body` (the analyst role instructions), and
`OutputFormat` (strict single-paragraph prompt contract). The current
production body is the V5 variant — the fifth iteration of the prompt,
developed against an offline eval harness described below.

Three prompt-engineering decisions that mattered:

1. **Hero preservation contract.** The first reference image is declared
   the HERO — it must be preserved pixel-faithfully. Images 1+ are
   "style cues only": the model extracts palette, lighting, and texture
   but **never imports their subjects**. Before this rule, product shots
   were routinely contaminated by mood-board elements.
2. **Aspect-ratio-specific safe zones.** 9:16 Stories enforce a strict
   central zone (top 15% excluded for the header, bottom 25% excluded
   for captions, 10% left / 20% right excluded for UI icons). 1:1 and
   16:9 get full-frame with 3–5% edge bleed. The constraint is compiled
   into the prompt, not post-processed — the model generates safe
   layouts instead of us cropping around bad ones.
3. **Concrete style language.** "Premium" and "luxury" are banned
   vocabulary. The model must emit specifics: *"soft diffused sidelight
   from upper-left, warm neutral palette of cream and oat tones, shallow
   depth of field, polished marble countertop."* Abstract adjectives
   give the model too much latitude and wreck reproducibility.

### The evaluation harness

Prompt engineering without measurement is vibes. I built an offline
harness — `workers/python-worker/notebooks/image_generation_experiments.ipynb`
plus a ~700-line `image_gen_helpers.py` that mirrors the production code
path — so I can sweep two axes at once:

- **Aspect-ratio matrix:** 1:1, 9:16, 4:5, 16:9, 3:2, 21:9.
- **Prompt variant:** canvas_analysis_v1 → v5, image_edit_v1 → v5.

For each (topic × ratio × prompt) cell, the harness generates N outputs
and logs them side-by-side. Crucially the helper module reuses the same
canvas composition, resolution-matching, and retry logic as production,
so an offline win translates directly. The "prompt-variant sweep" commit
on the current branch is that harness being extended.

### Image-generation mechanics

Inside `gemini_image_edit.py` (~1.5k lines, the workhorse processor):

- **Resolution matching.** `find_nearest_resolution()` picks the
  Gemini-supported resolution closest to the request's aspect ratio to
  minimise wasted pixels. The source is padded to the target (not
  stretched), and padding offsets are tracked so we can crop cleanly on
  the way out.
- **Upscale-then-crop.** Output is upscaled to the preferred resolution
  *then* cropped back to the requested aspect ratio, which preserves
  sharpness better than the naive downscale path.
- **Reference packing.** Gemini accepts ≤ 6 images per request. With
  more references, `ComposeDataCanvasProcessor` composites overflow into
  labelled data canvases before calling the model.
- **Inpainting (masked edits).** A red-marker overlay indicates the
  region to edit; after generation, red markers are detected and
  stripped from the final image. Lets users regenerate specific areas
  without regenerating the entire composition.
- **Safety + retry.** All five safety categories set to MEDIUM; retry
  with exponential backoff (5s → 120s, jittered) for rate limits and
  transient 5xx.

### Social-post pipeline end to end

`SocialCampaignGenerationWorkflow` drives: outline → user confirmation →
`BatchSocialPostImageGenerationWorkflow` (Path A or B, fanned out per
post) → caption generation → aspect-ratio variants → per-platform
publish workflow. Every step is an activity; the campaign survives pod
restarts; progress streams to the browser via Centrifugo.

---

## Pillar 3 — Image Processing with Custom Models

### The problem

Hosted APIs can't do everything. I needed:

- **Layer decomposition** — splitting a flat slide image into editable
  layers (background, hero, text block) with bounding boxes. No hosted
  API does this.
- **OCR inpainting** — removing text from an image without regenerating
  the whole thing. Gemini can't target a region cleanly enough.
- **Raster → SVG** — fast vectorisation for slide templates.
- **HTML rendering** — headless Chromium for charts, tables, rich
  content.

And I needed to do this **without running an always-on GPU for every
processor** — most of the time the heavy GPU processors are idle while
Gemini calls dominate the queue.

### Worker architecture

The Kotlin backend produces work onto SQS directly as
`SqsActivityMessage` JSON — no Python-native broker protocol is needed,
so the producer stays language-agnostic. On the Python side I built a
custom multiprocess orchestrator:

```
Main process
 ├─ N poll processes (one per SQS queue)  → mp.Queue
 └─ 1 processing process
      ├─ WeightedSemaphore (capacity = 10)
      └─ ThreadPoolExecutor (max_workers = 10)
            ├─ Worker: acquire(weight) → run processor → release
            ├─ On success: POST /resume → delete SQS message
            └─ On no-capacity: reset SQS visibility (return to queue)
```

The key primitive is the **weighted semaphore**. Each processor class
declares its own resource weight (1 for a cheap LLM call, up to 8 for a
GPU layer-decomposition job). Capacity is a single shared pool (default
10 slots). A poll process only fetches an SQS message if
`available_capacity >= processor_weight` — otherwise it leaves the
message on the queue for another pod or the next poll tick. This makes
GPU + CPU + remote-API workloads co-exist on one fleet without fighting
for RAM, and the poll-skip behaviour is KEDA-friendly for horizontal
autoscaling.

### Processors

Around sixteen processors ship today. The distinguishing ones:

- **`layer_decomposition`** — CyberAgent's **LayerD** model plus
  **BiRefNet** matting on GPU (~2 GB VRAM). Loads at process start (30–
  120 s cold), then decomposes a flat slide into per-element PNGs plus
  an optional SVG. Capped at 2048×2048 input to prevent CUDA OOM. Lazy-
  imported — only pods whose SQS queues include `layer_decomposition`
  pay the PyTorch load cost.
- **`ocr_in_painting`** — Hybrid: AWS Textract for text detection
  (remote), OpenCV's Telea inpainting algorithm for removal (local,
  CPU-only). Far cheaper than a full generative pass.
- **`raster_to_svg`** — `vtracer` (C++ binding) for fast, deterministic
  vectorisation.
- **`render_html`** — Playwright + Chromium for arbitrary HTML → PNG /
  WebP. Explicit restart logic in the supervisor for Chromium's fork-
  safety quirks.
- **`compose_data_canvas`** / **`compose_brand_board`** — the reference-
  image packers that feed Pillar 2's Path A.
- **`gemini_image_edit`**, **`gemini_text_to_speech`**, **`icon_search`**,
  **`code_execution`**, **`scrape_webpage`**, **`bundle_typescript_chart`**
  — remote-API and sandboxed-execution processors.
- **`background_removal`** — U²-Net via ONNX Runtime (~800 MB). Stubbed
  in the current registry pending reimplementation; the slot (type,
  weight, queue) is preserved.

### Callback protocol

When a processor finishes, the worker POSTs to the Kotlin backend's
`/api/v1/durable-job/resume` endpoint with the activity output and an
internal API key. **Only then** is the SQS message deleted. A crash
between "upload to S3" and "POST /resume" leaves the message for
redelivery, where the next attempt's check-before-act idempotency guard
(Pillar 1) resolves it cleanly. The Python worker is a first-class
citizen of the durable workflow event log without speaking Kotlin.

### Deployment

AWS ECS Fargate, split into a CPU pool (most processors) and a GPU pool
(`layer_decomposition`). One recent change: bumped ephemeral storage on
the Fargate pods to accommodate multi-gigabyte intermediate artefacts
(SVG + per-element PNGs for a single slide can exceed 1 GB).

### Scaling roadmap

The current co-located design is optimal at 1–2 GPU models. Once a third
lands, or when GPU utilisation on the dedicated pool dips below ~40%,
the next move is **NVIDIA Triton Inference Server** behind gRPC — worker
becomes CPU-only (cheap to scale), Triton handles model batching and
multi-model hosting. The clearest first extraction is
`background_removal` when it's reintroduced: single ONNX model, no
custom Python, trivially batchable.

---

## What I'm proud of

- **A determinism contract that survives production.** Not "retries that
  usually work" but a replay model with compile-time and review-time
  enforcement. Every side effect has one home; every agent decision is
  recoverable; every progress update is type-checked.
- **Prompt engineering with an eval harness.** Five numbered prompt
  iterations (v1 → v5) with an offline sweep matrix is unglamorous but
  it's why v5 is measurably better than v3 — I can show you, not tell
  you.
- **The weighted-semaphore worker.** One Python process cleanly hosts a
  2 GB GPU model, a 500 MB headless browser, and a flurry of
  remote-API calls, with no static per-workload pool sizing.
- **The Kotlin ↔ Python durable bridge.** SQS in, HTTP `/resume` out,
  idempotent on both sides — the Python fleet participates in the
  durable event log without pretending to be Kotlin.

## Lessons I took away

- **Event-sourced replay forces good architecture.** Once "only
  activities do side effects" is non-negotiable, testing the
  orchestration gets trivial — every workflow is a pure function of its
  input and its event log.
- **Prompt work needs measurement infrastructure.** It is otherwise
  indistinguishable from vibes. Number your prompts; log your outputs;
  sweep your axes.
- **Capacity scheduling beats queue-per-class.** One flexible scheduler
  with explicit weights is cheaper to operate than a static worker
  matrix, and it degrades gracefully under load instead of cliff-
  falling.
- **Write the contract, then the code.** The determinism rules, the
  hero-preservation rule, the idempotency guard — the cheapest code is
  the code that can't be written wrong.

---

## Code pointers

- Workflow engine — `service/core/module-durable-job/`
- Determinism rules — `service/.claude/rules/` and `.agent/system/durable-workflows.md`
- Image prompt builder — `service/app/module-workflow/module-socialpost/src/main/kotlin/com/c0x12c/activity/socialpost/CanvasAnalysisPromptBuilder.kt`
- Social-post workflows — `service/app/module-workflow/module-socialpost/`
- Slide workflows — `service/app/module-workflow/module-slide/`
- Agent tool workflows — `service/app/module-workflow/module-tool/`
- Python worker — `workers/python-worker/src/python_worker/`
- Evaluation harness — `workers/python-worker/notebooks/image_generation_experiments.ipynb`
- Prompt iterations — `workers/python-worker/notebooks/prompts/canvas_analysis_v*.txt`, `image_edit_v*.txt`

## Links

_(add GitHub links, demo video, screenshots here)_

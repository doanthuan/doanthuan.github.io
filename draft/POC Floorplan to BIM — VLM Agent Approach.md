# 4\. POC Floorplan to BIM — VLM Agent Approach

## 1. Executive summary

**The question we set out to answer:** can an AI agent credibly convert a residential floor plan image (間取り図) into a real, correctable BIM model — with nobody touching a CAD tool?

**The answer: yes, with one honestly-measured caveat.**

Upload a raster floor plan. In roughly a minute you get a finished-looking house — colored walls on the Japanese module grid, a gable roof with eaves, glazed windows, wood floors, furniture in the rooms — as a genuine IFC file that opens in Revit with its colors and millimeter units intact. You then refine it by chatting, in Japanese or English ("外壁を濃紺に塗って", "delete the wall between the LDK and the bedroom", "total floor area?"), and every edit regenerates the same IFC.

**The caveat, which is the PoC's central research finding:** extraction is *strong on structure* (walls, topology, room labels, dimensions) and *weak on opening placement* — door and window positions jitter run-to-run by roughly ±20% along their wall, and one evaluation run hallucinated an interior door that wasn't on the plan. This is measured by a built-in evaluation script that renders overlays for eyeball comparison, not asserted from vibes. It is a known, quantified limitation rather than a hidden defect.

**One-sentence demo:** *upload a floor plan → watch it become a house, step by step → tell the agent what to change → download the IFC and open it in Revit.*

---

## 2. Why this is hard, and what makes it work

Builders and stakeholders routinely work from floor plan images — that is the artifact customers actually have. Turning one into a usable BIM model today means manual re-modeling in CAD/BIM software.

The naive approach — "ask a vision model for coordinates and extrude them" — produces a wobbly mess. Raw model output has walls inset by half a thickness, dimensions \~1% short, and near-but-not-quite-orthogonal lines. Three design choices are what turn that into credible geometry:

**a) A semantic intermediate representation, not a mesh.** The AI never produces geometry. It produces a *Plan Model*: walls as centerlines, openings as offsets along a wall, rooms as labeled polygons. Everything downstream is deterministic and unit-tested.

**b) Deterministic post-processing that exploits domain structure.** Japanese residential plans are built on the 910mm 尺 module. Coordinates snap to its 455mm half-module — and this converts jitter into *correctness*: raw model output of `60 / 7220 / 5400` reliably becomes the exact `0 / 7280 / 5460`. This step is load-bearing; without it the model looks hand-wobbled.

**c) A bounded operation set for edits.** The refinement agent cannot write arbitrary code or emit coordinates freely. It calls 18 validated pure functions (`PlanModel → PlanModel`). An invalid edit returns a domain error the agent can read and retry against — *"No wall with id 'w-x'. Walls in the model: …"*.

The result: **two LLM calls in the entire system.** Everything else is deterministic, tested Python.

---

## 3. What it does — the user's path

1. **Upload** a floor plan (PNG/JPEG), by drag-and-drop, file picker, or one-click from a bundled sample gallery.
2. **Extraction** — a single vision-model call interprets the image into a Plan Model with real-world millimeter dimensions, calibrated from cues on the drawing (dimension text → 帖 labels → the 910mm module, in that order of reliability).
3. **Staged Reveal** — the UI draws what was found back over the original image, stage by stage: walls → doors/windows → labeled rooms → then the 2D overlay fades into 3D. Each stage arrives with a *Narration* message in the chat panel.
4. **The house rises, already finished** — a curated default *Palette* applies automatically: siding on exterior walls, white interiors, floors matched to each room's label (和室 → tatami, 浴室 → tile, else wood), a dark gable roof with eaves, wood door panels, translucent glazing. Rooms the plan didn't furnish get *auto-furnished* from a 9-item catalog.
5. **Refinement chat** — a tool-use agent applies bounded edits and answers read-only questions. The original plan image stays in the conversation, so *"図面の通りに戻して"* ("restore it as drawn on the plan") works without re-running extraction.
6. **Download IFC** — opens in external BIM tools with correct geometry, millimeter units, and the same colors the demo showed.

The viewer presents the model on a dark grid stage with a roof on/off toggle (exterior view ↔ "dollhouse" view for editing).

---

## 4. Architecture

Two components, one repo, one server. The backend serves the built frontend, so the whole demo runs from a single process.

**Three lanes to keep straight:**

- **Lane ① runs once per uploaded plan** and ends with a fully-finished Plan Model. It streams Narration events as it goes (NDJSON), so the user watches progress rather than a spinner.
- **Lane ② runs on every change.** The Plan Model is regenerated into IFC bytes the viewer loads verbatim.
- **Lane ③ is the chat loop**, where tool calls run bounded pure operations producing a new Plan Model, which flows back through lane ②.

Only the two LLM calls are non-deterministic. A rendered pipeline diagram lives at `docs/images/pipeline-flow.png` (regenerate with `backend/scripts/generate_pipeline_diagram.py`) for Confluence embedding.

### Technology choices

| Layer | Choice | Note |
| --- | --- | --- |
| Backend | Python 3.12, FastAPI, Uvicorn | IfcOpenShell is Python-only — this dictated the stack |
| IFC | IfcOpenShell 0.8.5 (`ifcopenshell.api`) | Real IFC authoring, not a text templater |
| Model SDK | `anthropic[bedrock]` ≥ 0.118 | Structured output via `messages.parse`; streaming |
| Domain model | Pydantic v2 | Doubles as the LLM's output schema — one definition, no drift |
| Frontend | React 19, TypeScript, Vite 8 |  |
| Viewer | ThatOpen Components 3.4 / web-ifc / three.js 0.185 | Renders the IFC bytes directly |

---

## 5. The Plan Model — the spine of the system

One Pydantic tree (`plan_model.py`) is the single source of truth: produced once by Extraction, edited only by Refinement operations, consumed by IFC generation. It is also the vision model's output schema, so there is exactly one definition of the domain.

```
PlanModel
└── storeys: [Storey]                    # always 1 today; a 2nd storey is additive
    ├── name, ceiling_height_mm
    ├── walls:     [Wall{id, start(x,y), end(x,y), thickness_mm}]      # centerlines, mm
    ├── openings:  [Opening{id, wall_id, kind: door|window,
    │                       offset_mm, width_mm, sill_mm, height_mm}]
    ├── rooms:     [Room{id, label, polygon: [(x,y)]}]
    ├── furniture: [Furniture{id, kind, position, rotation_deg}]        # 9-kind catalog
    ├── finishes:  Finishes{wall_colors: {wall_id: hex},
    │                       floors: {room_id: {material, color}}}
    └── roof:      Roof{pitch_deg, overhang_mm, color}                  # always present
```

**Conventions everything depends on:**

- **Millimeters everywhere.** Origin at the building's bottom-left exterior corner, y up.
- **Walls are centerlines.** Openings are located by `offset_mm` from their wall's start point — so moving a wall moves its openings for free.
- **All coordinates land on the 455mm half-module** after post-processing.
- **Missing finishes fall back to Palette defaults at generation time**, so a newly added wall is never unpainted.

That last convention is a small but significant piece of design: it means operations never have to remember to paint things.

---

## 6. Pipeline stage by stage

### 6.1 Extraction — a function, not an agent

One vision call via `client.messages.parse` with a Pydantic schema (`ExtractedStorey`) — the SDK enforces output structure, so there is **no JSON parsing or repair code**. Adaptive thinking is on; the call streams (a long thinking phase on a non-streaming request gets killed by idle-connection limits on Bedrock).

The prompt teaches four things: the coordinate system, the scale-calibration priority (dimension text → 帖 labels → 910mm module), the drawing conventions (a gap with a quarter-circle arc is a door; a gap bridged by parallel thin lines is a window), and the furniture symbol vocabulary.

The schema also requires `image_calibration` — the pixel position of the plan origin plus millimeters-per-pixel — which is what lets the frontend draw extraction results *aligned on the uploaded image* during the Staged Reveal.

Extraction is deliberately **purely geometric and semantic**: it never guesses colors, and it is **never re-run** during refinement. It reports only furniture the plan actually draws; inferring furniture from labels is a separate deterministic step.

Failures map to an explicit error with diagnostics — `stop_reason`, block types, and output token count — because the single most common failure is thinking consuming the entire token budget, and a bare "no parseable model" message is undebuggable.

### 6.2 Post-processing — deterministic cleanup

Three passes over raw LLM geometry, in order:

| Pass | Rule | Why |
| --- | --- | --- |
| **Orthogonalize** | Walls within 7° of an axis become axis-aligned | Near-orthogonal is always a drafting artifact |
| **Join** | Endpoints within 250mm cluster and move to their centroid | Walls that should share a corner, don't quite |
| **Snap** | Every coordinate to the 455mm half-module | Converts jitter into exact modular dimensions |

Opening offsets are then remapped to **preserve their absolute plan position** while their wall moves underneath them, clamped inside the wall, and dropped if wider than it. Degenerate zero-length walls are removed.

### 6.3 Palette — the default look

Applied once at upload; fills only *unset* finishes, so chat overrides survive regeneration.

- **Exterior classification:** a wall whose centerline lies on the footprint bounding rectangle's perimeter is an Exterior Wall (post-snap this test is exact, not fuzzy) → siding color; everything else → interior white.
- **Floor materials** from bilingual label heuristics: 和室/畳/TATAMI → tatami; 浴/風呂/トイレ/WC/BATH/… → tile; else wood.
- Named `IfcMaterial` strings (`siding`, `flooring`, `tatami`, `tile`, `roofing`, …) are attached alongside colors — these are the join key for the viewer's texture layer, and are independently valuable to external BIM tools.

### 6.4 Auto-furnish

Rooms the plan drew no furniture in get filled from a 9-kind catalog (bed, wardrobe, sofa, low table, dining set, kitchen counter, bathtub, toilet, washbasin), chosen by room label in placement-priority order. Placement is rule-based: items go against a wall, inset, clear of door and window spans (projected onto each wall side) and clear of other furniture, skipping anything that doesn't fit. Nothing outside the catalog can exist in the model.

### 6.5 IFC generation — Plan Model → bytes

Built with `ifcopenshell.api` (project, units, contexts, spatial tree) plus:

| Element | How |
| --- | --- |
| Walls | Box representation placed by rotation matrix along the centerline; openings cut via `IfcOpeningElement` |
| Door / window fills | `IfcDoor` panel and `IfcWindow` translucent pane placed in the opening, related via `IfcRelFillsElement` |
| Floor plates | One `IfcCovering` (FLOORING) per room, extruded from the room polygon |
| Base slab | `IfcSlab` under the footprint |
| Roof | `IfcRoof` (GABLE\_ROOF) — a triangle profile across the short span extruded along the ridge; rise = half-span × tan(pitch); overhang widens the prism |
| Furniture | One correctly-typed product per piece — `IfcFurniture` (BED / SOFA / TABLE / SHELF / KITCHENCOUNTER) and `IfcSanitaryTerminal` (BATH / TOILETPAN / WASHHANDBASIN) — as a multi-box representation with per-part colors, not generic proxies |
| Rooms | `IfcSpace` per room |
| Colors | One deduplicated `IfcSurfaceStyle` per color/transparency pair, assigned per representation |
| Materials | `IfcMaterial` records via `IfcRelAssociatesMaterial` |

**The #1 trap for future contributors:** two unit conventions coexist inside IfcOpenShell — `ifcopenshell.api.geometry` helpers take **SI meters** and convert internally, while `ShapeBuilder` takes **project units (mm)** raw. Both were verified empirically with probe scripts before use, and the tests pin the behavior.

### 6.6 Narration — computed, not generated

The step-by-step chat messages during conversion are **templates computed from Plan Model data** — no extra LLM calls, no latency, no cost, no hallucination risk. They stream as NDJSON events, one per Staged Reveal beat.

Two deliberate properties: the events are **semantic, not temporal** (the geometry events fire at the same instant, each carrying only its copy-relevant slice — the frontend owns pacing), and Narration **never enters the Refinement agent's context**. It is presentation only.

### 6.7 Refinement — the agent

A **manual tool-use loop** rather than the SDK's tool runner — deliberate, because the Anthropic client is the faking seam for tests, and a manual loop keeps every request and response visible to them. Per chat turn:

```
history += user message (+ Plan Model JSON snapshot in <plan_model> tags;
                         + the original plan image, first turn only)
loop (max 8):
    response = model(system, tools, history)
    history += assistant content
    if stop_reason != tool_use: break
    for each tool_use: apply operation to Plan Model
                       (errors → tool_result with is_error, agent can react)
    history += tool_results
if anything mutated: regenerate IFC
```

The **operation registry** (`OPERATIONS` in `operations.py`) is the single place an operation exists: description + JSON schema + pure-function handler + `mutates` flag. The agent's tool list derives from it, so adding a capability is one registry entry.

**The 18 operations:**

| Group | Operations |
| --- | --- |
| Walls | `add_wall`, `delete_wall`, `move_wall`, `set_wall_thickness` |
| Openings | `add_opening`, `delete_opening`, `move_opening`, `resize_opening`, `set_opening_kind` |
| Rooms & storey | `relabel_room`, `set_ceiling_height` |
| Appearance | `paint_wall` (one / all\_exterior / all\_interior), `set_room_floor`, `configure_roof` |
| Furniture | `add_furniture` (auto-places if no position given), `move_furniture`, `delete_furniture` |
| Read-only | `get_model_summary` — areas in m² and 帖, counts, finishes, roof config, furniture per room |

Read-only turns skip IFC regeneration — verified by byte-identical output. Every mutating operation is a validated pure function with domain error messages the agent can act on.

### 6.8 Frontend

| File | Responsibility |
| --- | --- |
| `App.tsx` | Phase machine `idle → extracting → revealing → viewing`; upload + gallery; session rejoin on refresh; roof-toggle state reapplied across IFC reloads |
| `Reveal.tsx` | Staged Reveal — SVG stages over the uploaded image, positioned via the extraction's image calibration, advanced by Narration events |
| `Chat.tsx` | Chat panel; reloads the viewer only when the response says `mutated` |
| `viewer.ts` | ThatOpen/web-ifc setup on the dark grid stage; `loadIfc` (disposes previous model, re-applies textures), `setRoofVisible`, `clear` |
| `textures.ts` | The ADR-0003 texture layer (see below) |

The web-ifc wasm binary is copied from `node_modules` by a postinstall script and served locally — the auto-fetched wasm can mismatch the bundled JS and fail cryptically.

---

## 7. The four architectural decisions

Each is recorded as an ADR in `docs/adr/`. These are the load-bearing choices; everything else is implementation.

### ADR-0001 — The viewer renders the IFC itself

There is no separate 3D scene built from the Plan Model. The backend generates an IFC file; the browser loads **those exact bytes** with web-ifc.

*Why it matters:* one geometry path, so the on-screen model cannot drift from the exported file. It makes the BIM claim literal — what you see is the file you open in Revit. **The price:** a mandatory Python backend and a sub-second IFC round trip on every mutating edit.

### ADR-0002 — Appearance is IFC-native; rendering polish is viewer-side

Everything describing the *building* (roof, door panels, glazing, wall paint, floor materials) lives in the IFC as elements with flat-color surface styles, so colors survive into Revit. Everything describing the *presentation* (background, shadows, roof visibility) is viewer-side rendering of that same file.

### ADR-0003 — The viewer renders textures keyed by the file's Material names

Flat colors were judged insufficient after two rounds of review. Elements now carry named `IfcMaterial` records, and the viewer multiplies procedurally drawn canvas detail maps — plank grain, tatami weave with mat borders, tile joints, siding laps, roof standing seams — over the file's flat colors.

This ADR is worth reading as a **process artifact**: it was run spike-first with an explicit timebox and a written go/no-go. The spike found that the viewer's batched geometry carries no UV attribute, so `material.map` was impossible; per-element material override was also impossible because same-colored elements batch into one mesh. The mechanism that worked was `onBeforeCompile` injection of world-space **triplanar** sampling, verified with a checkerboard before any real texture was drawn.

It also *knowingly relaxes* ADR-0001 for appearance only, and records the resulting limitation: elements recolored away from Palette defaults render flat.

### ADR-0004 — Model spend runs through a Bedrock application inference profile

Organisational policy requires model spend to be attributable to a cost centre, which means invoking through an application inference profile — the only Bedrock identifier carrying cost-allocation tags. A system-defined `jp.anthropic.*` cross-region profile does not.

The consequence worth recording: **there is deliberately no default model id for the Bedrock backend.** The ARN is account-specific and can't be checked in, and the tempting fallback to a public profile is exactly the wrong failure mode — it would keep working while quietly running a different model on untagged spend. A missing `BIM_AGENT_MODEL` refuses to start the server instead.

---

## 8. What is deliberately real — the credibility claims

For a feasibility PoC, the thing being tested is the *claim*, so each is falsifiable:

| Claim | How it's guaranteed |
| --- | --- |
| **The viewer renders the actual IFC file**, not a lookalike scene | ADR-0001; single geometry path; the download endpoint serves the same bytes the viewer loaded |
| **Appearance lives in the file** and survives into Revit | ADR-0002; colors are IFC surface styles, verified by re-parsing generated bytes |
| **The AI is real end to end** | Extraction is a live vision call; chat is a live tool-use agent. Nothing is pre-baked or recorded |
| **Extraction quality is measured, not asserted** | An opt-in eval script runs real extraction over every bundled sample and writes a report with overlay renders |

---

## 9. Engineering quality

**136 automated tests, all passing.** Roughly 0.8 lines of test per line of backend source. The strategy is **seam-based** — there is no mocking below one boundary:

```
┌──────────────────────────────────────────────────────────────────┐
│ HTTP seam (primary): FastAPI TestClient drives real routes.      │
│ The ONLY fake is the Anthropic client — parse() returns a canned  │
│ Plan Model, streaming create() plays scripted tool-call           │
│ responses. Post-processing, Palette, auto-furnish, operations,     │
│ IFC generation and session state all run for real.                │
├──────────────────────────────────────────────────────────────────┤
│ Pure-function seams: post-processing, every operation, palette    │
│ classification, quantities, roof geometry, furniture placement.   │
├──────────────────────────────────────────────────────────────────┤
│ IFC parse-back seam: generated bytes are re-parsed with           │
│ IfcOpenShell and asserted structurally — element counts, fill     │
│ relationships, style colors, extrusion depths, mm units — never   │
│ golden-file byte comparison (GlobalIds are random).               │
├──────────────────────────────────────────────────────────────────┤
│ LLM quality: NOT asserted in CI. The opt-in eval script runs      │
│ real Extraction over samples/ and writes a report + overlays.     │
├──────────────────────────────────────────────────────────────────┤
│ Frontend: manual demo-script walkthrough in the browser.          │
└──────────────────────────────────────────────────────────────────┘
```

| Test file | Tests | Covers |
| --- | --- | --- |
| `test_operations.py` | 36 | Every bounded operation, valid and invalid |
| `test_api.py` | 14 | HTTP routes, upload stream, error paths, session lifecycle |
| `test_chat.py` | 13 | Refinement loop, tool dispatch, mutation flags |
| `test_furniture.py` | 12 | Catalog, placement rules, auto-furnish |
| `test_postprocessing.py` | 11 | Orthogonalize / join / snap / opening remap |
| `test_furniture_operations.py` | 10 | Add / move / delete furniture |
| `test_ifc_generation.py` | 9 | Structural parse-back assertions |
| `test_roof.py` | 6 | Gable prism geometry |
| `test_palette.py` | 6 | Exterior classification, label heuristics |
| `test_narration.py` · `test_materials.py` · `test_fills.py` | 5 each | Copy templates · IfcMaterial records · door/window fills |
| `test_quantities.py` | 4 | Shoelace areas, m²/帖 conversion |

**Two techniques worth stealing:**

- *"No IFC regeneration happened"* is asserted by byte-identical `GET /api/ifc` responses — any regeneration would mint new random GlobalIds. A negative assertion made cheap and exact.
- IFC output is asserted **structurally by re-parsing**, never by golden file. Random GlobalIds would make byte comparison flap.

**A deliberate gap:** LLM output quality is not a CI gate. That is correct for a PoC — it would make the suite slow, costly, and flaky — but it means prompt changes are currently unguarded. See next steps.

---

## 10. Findings

1. **Structure extraction is strong.** Across runs on the fixture plan: 5/5 walls with correct topology, all rooms correctly labeled, and — after grid snapping — the exact 7280×5460mm dimensions. Western-style and English-labeled plans also work via the bilingual heuristics.
2. **Opening placement is the weak axis.** Positions jitter run-to-run (\~±20% along the wall), and one eval run hallucinated an extra interior door. The eval overlay makes such errors visible at a glance.
3. **The 455mm grid snap is load-bearing.** It is the single highest-leverage line of code in the system: raw coordinates like 60/7220/5400 become exactly 0/7280/5460.
4. **Chat refinement is dependable.** The agent resolves natural-language references (rooms, "exterior walls") to element operations correctly, works bilingually, and plan-grounded restore requests reproduce geometry from the image.
5. **The IFC round trip holds.** Colors authored as IFC surface styles render in the browser and persist in the downloaded file.
6. **Structured output eliminated a whole error class.** Because the Pydantic schema is enforced by the SDK, there is no JSON repair code anywhere — the historical failure mode of LLM-to-structured-data pipelines simply doesn't exist here.

---

## 11. Limitations and scope boundaries

**Deliberately out of scope:**

| Excluded | Why |
| --- | --- |
| Cost estimates, construction documents | The PoC stops at quantity and finish Q&A |
| Hip roofs (寄棟), ceilings, per-face wall painting | Geometry/rendering complexity beyond PoC value |
| Stairs, columns/beams, site, multi-storey | Schema is storey-additive; a second floor is a next-revision candidate, not a rewrite |
| CAD input (DWG/DXF/JWW), hand-drawn sketches | Raster images only |
| Robustness on arbitrary uploads | Curated samples. *Graceful failure* on bad input is in scope; *good results* on bad input are not |
| Persistence, auth, deployment, multi-user | In-memory single session; local demo |

**Known behaviours to be aware of when demoing:**

- **Session is in memory.** A backend restart clears it; the flow is built for that (start over / re-upload).
- **Real extraction takes \~30–60s**; chat turns \~10–60s. Both are live model calls.
- **Extraction variance:** re-uploading the same plan can produce a slightly different model.
- **Wall paint is per wall, both faces.** "Paint the bedroom blue" paints the room's bounding walls; if one is exterior, its outside face changes too. Documented in the operation description so the agent warns the user.
- **Relabeling a room does not re-derive its floor material** (deliberate; use `set_room_floor`).
- **Elements recolored away from Palette defaults render flat** — the documented cost of ADR-0003.
- **Window frame geometry is omitted** — the wall reveal visually frames the pane; a solid frame box would occlude it.
- **No ambient occlusion / postproduction:** ThatOpen's PostproductionRenderer drew black silently in this setup; the viewer ships on the plain renderer.

**Two rough edges found while writing this report** (small, worth fixing):

- The upload error handler catches all exceptions and narrates them with *"Try another floor plan image, or the same one again."* For configuration and credential failures — a dead SSO session, a missing key — retrying can never help. Worth splitting transient failures from config failures in the narration.
- `.env` is read once at import, so changing the model provider silently has no effect until the server is restarted. This is easy to trip over.

---

## 12. Suggested next steps

Ordered by ratio of value to effort, none committed:

1. **Make the eval script a regression gate for prompt changes.** The measurement infrastructure already exists; it just isn't wired to anything. This is the cheapest way to stop the known weak axis from silently getting worse.
2. **Attack opening-placement variance directly** — extract twice and reconcile, or add a self-check pass. This is the one substantive quality finding, and it is a bounded experiment.
3. **Second storey and stairs.** The schema is already storey-additive, so this is additive work rather than a rewrite.
4. **Broaden the sample set.** Findings currently rest on a small curated set; more plans would either strengthen or usefully complicate them.
5. **Split narration of transient vs configuration failures** (see rough edges above).

**If this were to become a product, the honest gap list is:** persistence and multi-user sessions, authentication, a deployment story, concurrency (the backend holds exactly one in-memory session), extraction robustness on uncurated input, and a frontend test suite.

---

## Appendix A — HTTP API

| Endpoint | Purpose |
| --- | --- |
| `POST /api/upload` (multipart image) | Run the conversion pipeline, streaming Narration as NDJSON; the terminal `complete` event carries the plan and calibration. Pipeline failures end the stream with an `error` event rather than an HTTP error, because the stream has already begun |
| `POST /api/chat` `{message}` | One Refinement turn. Returns `{reply, plan, mutated}` |
| `GET /api/ifc` | Current IFC bytes (viewer load + download). 404 until a session exists |
| `GET /api/samples`, `GET /api/samples/{name}` | Gallery listing (globbed per request — drop a file in `samples/` and it appears) and traversal-safe file serving |
| `POST /api/reset` | Clear the session |
| `GET /` (mount) | The built frontend (`frontend/dist`) |

## Appendix B — Module map

```
backend/src/bim_agent/
├── app.py             FastAPI routes, Session dataclass, NDJSON upload stream, static mount
├── config.py          Backend/model/effort resolution; fail-fast client construction
├── plan_model.py      Pydantic domain model + fixture
├── extraction.py      Vision LLM call (messages.parse), calibration, prompt
├── postprocessing.py  Orthogonalize / join / 455mm snap / opening remap
├── palette.py         Default finishes, exterior classification, label heuristics, Material names
├── furniture.py       Catalog, placement rules, auto-furnish
├── narration.py        Narration copy templates (no LLM)
├── operations.py      18 bounded ops (pure functions) + OPERATIONS registry
├── refinement.py      Manual tool-use agent loop
├── quantities.py      Areas (shoelace, m²/帖), counts, finish/roof/furniture summary
└── ifc_generation.py  Plan Model → IFC bytes (IfcOpenShell), style cache, roof prism

backend/scripts/
├── eval_extraction.py          Opt-in extraction quality report + overlays
├── generate_fixture_plan.py    Synthetic 間取り図 generator (Pillow)
├── generate_pipeline_diagram.py Pipeline diagram for docs
└── probe_bedrock.py            Provider connectivity probe

frontend/src/
├── App.tsx   Phase machine, upload, gallery, roof toggle
├── Reveal.tsx Staged Reveal SVG overlay
├── Chat.tsx   Chat panel
├── viewer.ts  ThatOpen/web-ifc viewer
└── textures.ts ADR-0003 triplanar texture layer
```

## Appendix C — How to run

```bash
cd frontend && npm install && npm run build
cd ../backend && uv run uvicorn bim_agent.app:create_app --factory --port 8000
```

Open `http://localhost:8000`. Requires `uv`, Node 20+, and model credentials in a `.env` at the repo root.

**Model provider** — configured in `.env` (see `config.py`):

- **Amazon Bedrock** (default): set `BIM_AGENT_MODEL` to the application inference profile ARN and `AWS_REGION` to its region, then run the backend under AWS credentials — `aws-sso exec -p <profile> -- uv run uvicorn …`. See ADR-0004.
- **Anthropic API**: set `BIM_AGENT_BACKEND=anthropic` and `ANTHROPIC_API_KEY`, and leave `BIM_AGENT_MODEL` unset. A local-development convenience — spend on this path is not attributable to a cost centre.

`BIM_AGENT_EFFORT` (low/medium/high/max) tunes thinking depth. Thinking counts against extraction's token budget, so drop to `medium` if a model overruns. `.env`** is read once at startup — restart the server after changing it.**

To measure extraction quality: `cd backend && uv run python scripts/eval_extraction.py` (spends real tokens; writes `eval-report/report.md` with overlays).

## Appendix D — Related documents

| Document | Contents |
| --- | --- |
| `CONTEXT.md` | The domain glossary (Plan Model, Extraction, Palette, Narration, …) — the canonical vocabulary used in code, issues, and this report |
| `docs/architecture.md` | Technical architecture (v2-era; this report supersedes it where they disagree) |
| `docs/project-overview-prd.md` | Product requirements and user stories (v2-era) |
| `docs/adr/0001…0004` | The four architectural decision records |

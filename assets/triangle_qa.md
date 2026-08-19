# Triangle Health — Interview Prep Q&A

**Project:** Agentic Graph-Retrieved Reasoning Framework for biomedical decision support (oncology + neurology)
**Stack:** Python, FastAPI, Celery, Airflow, Neo4j, vector DB, GraphRAG, BioBERT/PubMedBERT, Tx-Gemma, BioMni, vLLM, Google ADK, MCP

---

## 0. How to use this document

Three tiers of questions, roughly how a real loop will go:

| Round | What they probe | Sections |
|---|---|---|
| Recruiter / hiring manager screen | Story, scope, ownership, impact | 1, 12 |
| Technical deep dive | KG, retrieval, agents, evals | 2–8 |
| Architecture / staff-level | Scale, cost, failure modes, tradeoffs | 9–11 |
| Behavioral | Decisions, conflict, ambiguity | 12 |

**Critical prep warning before you read further:** your project page claims *"significantly outperforms traditional biomedical RAG"*, *"reduced hallucination"*, and *"improved retrieval accuracy"* with **zero numbers**. Every competent interviewer will attack exactly this. Section 8 tells you what to fill in. Do not walk into the interview without concrete figures and the eval setup that produced them — an unquantified claim on a resume reads as a red flag, not a strength.

---

## 1. Project Overview & Framing

### Q1.1 — "Walk me through this project."

**Answer (~90 seconds, the version you memorize):**

Triangle Health is a biomedical decision-support system. The core problem: clinicians and researchers asking questions like *"what emerging treatments exist for a EGFR-mutant NSCLC patient who progressed on osimertinib?"* get answers from standard LLMs that are fluent, plausible, and unverifiable. In medicine, unverifiable is unusable.

I designed and built an Agentic Graph-Retrieved Reasoning Framework with three layers:

1. **A provenance-aware biomedical knowledge graph** built from ClinicalTrials.gov, PubMed, Semantic Scholar, OpenAlex, and PrimeKG. Every node and edge is traceable back to a source document and a specific extraction step, and entities are canonicalized against UMLS, MeSH, OMIM, and DrugBank so "NSCLC," "non-small cell lung carcinoma," and the C-code all resolve to one node.

2. **A hybrid retrieval engine** that runs dense semantic retrieval with domain embeddings alongside symbolic graph traversal, then fuses the two with a scoring function over similarity, graph relevance, extraction confidence, and node centrality. The output isn't a bag of text chunks — it's a subgraph plus metadata plus a reasoning trace.

3. **A multi-agent reasoning layer** — an orchestrator that decomposes a clinical question into subtasks, routes to a deep-research subagent (ReAct, web + KG tools) and a medical subagent (ReAct backed by self-hosted Tx-Gemma and Therapeutic Data Commons tools), then a tournament-style pairwise ranker that scores candidate treatments on evidence strength, safety, guideline support, patient fit, and mechanistic plausibility.

The through-line is auditability. Every claim in the final answer resolves to a source document, an extracted relation, an agent reasoning step, and a versioned KG entity.

**Coaching note:** lead with the *problem*, not the stack. The stack list is the least interesting thing you can say first.

---

### Q1.2 — "Why a knowledge graph? Why not just vector RAG over PubMed?"

Vector RAG fails on three things this domain needs constantly:

- **Multi-hop relational queries.** "Which drugs target a pathway downstream of a gene mutated in this patient's tumor?" is a graph traversal — drug → target → pathway → gene. There's no single passage in PubMed that contains that chain. Cosine similarity over chunks cannot compose it.
- **Aggregation and negation.** "How many Phase III trials exist for this indication, and which were terminated?" is a query over structured facts, not a similarity search.
- **Provenance and explainability.** A vector store gives you "here are 8 similar chunks." A KG gives you "here is the specific edge `drug --INHIBITS--> target`, extracted from PMID 12345678, confidence 0.91, extraction model v2.3." For clinical use, that difference is the whole product.

The honest answer is that it's *hybrid* for a reason — graphs are bad at fuzzy, unstructured, novel-phrasing queries, and dense retrieval covers that. Neither alone is sufficient. That's why the retrieval layer runs both and fuses.

---

### Q1.3 — "Who is this for, and what does a user actually do with it?"

Three personas, deliberately different surfaces:

- **Researcher:** literature exploration, mechanism discovery, "what's the evidence landscape for X."
- **Clinician:** trial matching and treatment options for a specific patient profile — mutation, stage, prior lines of therapy, comorbidities.
- **Patient:** grounded, plain-language explanation with citations.

The system is decision *support*, not a decision maker. It surfaces evidence and ranked options with justification; a clinician decides. That distinction matters for regulatory scope (see Q10.1) and I designed to it from the start.

---

### Q1.4 — "Why oncology and neurology first?"

Oncology is the highest-value case for this architecture: mutation-driven treatment selection is inherently a graph problem (variant → gene → pathway → drug → trial), the trial landscape moves faster than any human can track, and precision medicine means the answer is genuinely patient-specific. Neurology second because it shares the mechanistic-reasoning shape but has sparser, noisier evidence — a good stress test for whether the retrieval fusion degrades gracefully.

Narrow scope also made the ontology tractable. Canonicalization across all of medicine is a multi-year problem; across two specialties it's achievable with high precision.

---

## 2. Knowledge Graph Construction

### Q2.1 — "Walk me through the ingestion-to-graph pipeline."

Airflow DAGs, one per source, running on a schedule matched to source update cadence (ClinicalTrials.gov daily, PubMed daily incremental, OpenAlex/Semantic Scholar weekly snapshots, PrimeKG as a versioned bulk import).

Stages:

1. **Fetch & normalize** — pull raw records via API/bulk, normalize to a common document schema (id, title, abstract/body, source, publication date, type, source-native IDs).
2. **Dedupe** — cross-source dedup on DOI/PMID/NCT ID, then fuzzy title+author matching for records missing identifiers. OpenAlex and Semantic Scholar overlap heavily with PubMed; without this you triple-count evidence and corrupt any confidence weighting.
3. **Semantic chunking** — split on structural + semantic boundaries rather than fixed token windows (details in Q2.4).
4. **LLM summarization** — structured summaries of papers and trials into a fixed schema (population, intervention, comparator, outcome, effect direction, limitations for papers; phase, status, eligibility, arms, endpoints for trials).
5. **Entity extraction** — diseases, drugs, genes, mutations, trials, phenotypes.
6. **Canonical linking** — map surface forms to UMLS CUIs, MeSH descriptors, OMIM, DrugBank IDs.
7. **Relation extraction** — typed edges with confidence scores and provenance.
8. **Write** — nodes/edges into Neo4j, chunk embeddings into the vector store, both keyed by the same canonical entity IDs so retrieval can join across them.

Celery handles the fan-out — per-document extraction is embarrassingly parallel, and decoupling it from Airflow means a slow LLM call doesn't block a DAG slot. Airflow owns orchestration and dependencies; Celery owns throughput.

---

### Q2.2 — "How do you do entity extraction, and how accurate is it?"

Layered, because pure-LLM extraction is expensive and pure-dictionary extraction misses novel entities:

- **Dictionary/gazetteer pass** against UMLS, MeSH, DrugBank, HGNC — high precision on known entities, near-zero cost, catches the long tail of standard vocabulary.
- **LLM extraction pass** for entities the gazetteer misses: novel compounds, unusual mutation notations, newly-named phenotypes. Constrained output schema, few-shot with domain examples.
- **Reconciliation** — the two are merged; gazetteer hits get high base confidence, LLM-only entities get flagged lower and are candidates for review.

*[FILL IN: your actual entity-level precision/recall on a held-out annotated set, and the set's size and provenance.]* If you don't have this, say so plainly and describe the spot-check protocol you used instead — an honest "we validated on a 200-document manual sample, precision was roughly X" beats a confident number you can't defend.

---

### Q2.3 — "Canonical linking is hard. How did you handle ambiguity?"

This is the single hardest part of the KG and I'd flag it as such.

Problems and handling:

- **Polysemy** — "ALS" is amyotrophic lateral sclerosis or advanced life support. Resolved with context: candidate CUIs are scored against the surrounding chunk embedding and the document's MeSH terms, not the mention alone.
- **Abbreviations** — resolved with a document-local abbreviation table built from the standard "long form (SHORT)" pattern, applied before global lookup.
- **Gene vs protein vs drug name collisions** — type constraints from the extraction step restrict the candidate space before linking.
- **Multiple valid CUIs for one concept** — UMLS is not clean; I maintain a preferred-source ordering and a small manually-curated override table for high-frequency oncology/neurology concepts.

Unresolvable mentions are kept as nodes with a `canonical: false` flag rather than dropped. Dropping them silently loses evidence; flagging them lets retrieval down-weight rather than ignore.

**If pushed on failure rate:** be honest. Something like "clean linking on high-frequency concepts, meaningful degradation on the tail" is credible; "99% accuracy" is not, and any interviewer with UMLS experience will know it.

---

### Q2.4 — "What is 'semantic chunking' and why not fixed-size?"

Fixed-size chunking splits mid-sentence and mid-argument, which in biomedical text routinely severs a finding from its qualifier — you get "treatment X improved survival" in one chunk and "in patients without brain metastases, though not statistically significant" in the next. That is actively dangerous in this domain.

My approach: split on document structure first (abstract sections, IMRaD headings, trial arms/eligibility blocks), then within long sections split at points of low semantic similarity between adjacent sentence embeddings, with a max-size backstop.

**Contextual embedding:** each chunk is prefixed at embedding time with a compact context header — document title, section, study type, and the canonical entities present. This keeps a chunk retrievable when it uses a pronoun or shorthand for the thing you're searching for. It's the same idea as Anthropic's contextual retrieval work, adapted so the context is drawn from the KG rather than generated freeform.

---

### Q2.5 — "What does 'provenance-aware' mean concretely? What's on an edge?"

Every edge carries:

- `source_doc_ids` — the documents supporting it (a relation asserted by five papers is not the same as one)
- `extraction_method` + `model_version` — dictionary, LLM (which model, which prompt version)
- `confidence` — extraction confidence, calibrated where possible
- `evidence_spans` — the specific text that produced the assertion
- `assertion_type` — asserted, negated, hypothesized, contradicted (see Q2.6)
- `valid_from` / `valid_to` — for versioning (see Q2.7)

This is what makes the "every inference is auditable" claim real rather than aspirational. When the UI shows a justification, it's rendering these fields, not a post-hoc LLM explanation of its own answer.

---

### Q2.6 — "How do you handle contradictory evidence? Two papers disagree."

The graph does not force resolution — collapsing a genuine scientific disagreement into one edge is a correctness bug, not a cleanliness win.

Contradictions are represented explicitly: edges carry assertion type and are grouped, so `drug --EFFECTIVE_FOR--> disease` can coexist with a negated assertion from another source. Downstream:

- **Retrieval** returns both and marks the conflict.
- **The agent layer** is required to surface disagreement rather than pick silently — this is a large part of what the multi-agent debate structure buys.
- **Ranking** weights by evidence hierarchy: meta-analysis and Phase III RCT above Phase II above case reports above preclinical. A single in-vitro contradiction does not cancel a Phase III result, and the ranking criteria encode that.

Retracted papers are a special case — I flag against retraction databases and exclude or hard-down-weight, because a retracted finding in a KG is worse than a missing one.

---

### Q2.7 — "You said 'versioned KG.' What does versioning mean here and why?"

Two reasons it's non-negotiable:

1. **Auditability over time.** If a clinician got an answer in March, I need to reconstruct exactly the graph state that produced it. Without versioning, "auditable" is a marketing word.
2. **Evidence changes.** A trial moves from recruiting to terminated. A drug gets a new label warning. A paper is retracted. The graph must represent *when* something was true.

Implementation: bitemporal edges — `valid_from`/`valid_to` on the assertion, plus ingestion timestamp. Updates are soft — a superseded edge is closed, not deleted. Queries default to "as of now" but can be run as-of any timestamp. Snapshot tags mark releases used for evaluation so eval results stay reproducible.

**Likely follow-up — "doesn't that blow up graph size?"** Yes, monotonically. Mitigations: close-and-archive cold edges to object storage past a retention window, and only version assertion edges, not high-churn derived properties like centrality (recomputed, not versioned).

---

### Q2.8 — "Why Neo4j?"

Mature Cypher, good tooling, native support for the traversal patterns I needed, and an ecosystem (APOC, GDS for centrality) that meant I wasn't implementing PageRank myself.

**Honest tradeoffs I'd name:** single-writer scaling limits on the community edition, memory-hungry for large graphs, and licensing cost at scale. If I were re-deciding at 10x the graph size I'd seriously evaluate a distributed option, and if the workload had shifted more toward analytics than traversal I'd have looked at a property-graph-on-columnar approach. For this workload — deep traversal, moderate write volume, heavy read — Neo4j was right.

---

## 3. Hybrid Retrieval

### Q3.1 — "Walk me through what happens between question and retrieved evidence."

1. **Query understanding** — extract and canonicalize entities from the question (same linking pipeline as ingestion, so query entities and graph entities share an ID space). Classify intent: trial matching vs mechanism vs comparison vs general.
2. **Dense retrieval** — embed the query with the domain encoder, ANN search over the chunk index, filtered by any hard constraints (date, study type, indication).
3. **Graph retrieval** — anchor on the canonicalized query entities, traverse typed paths bounded by depth and relation type according to intent. Trial matching traverses differently than mechanism explanation.
4. **Fusion** — merge into a single candidate set scored by the fusion function.
5. **Rerank** — cross-encoder over top-k.
6. **Assemble** — return a subgraph, the supporting chunks, provenance metadata, and the traversal path as a reasoning trace.

The key design point: dense and graph retrieval share the canonical entity ID space. That's what makes fusion meaningful rather than two unrelated result lists stapled together.

---

### Q3.2 — "Explain the fusion scoring. How did you set the weights?"

Score combines four signals:

- **Semantic similarity** — dense retrieval score
- **Graph relevance** — a function of path length, relation type weight, and number of distinct paths connecting the candidate to query anchors (multiple independent paths is much stronger than one)
- **Confidence** — extraction confidence on the supporting edges, aggregated
- **Centrality** — precomputed node centrality as a weak prior on the entity's importance in the graph

**On weights, be honest about method.** The strong answer: normalize each signal to a comparable scale (raw cosine and path counts are not commensurable — this is where naive fusion breaks), then tune on a labeled query set with a retrieval metric as objective. The weak answer you should avoid: "I picked weights that seemed reasonable."

*[FILL IN: how you actually tuned — grid search, Bayesian opt, or manual on a dev set. And the dev set size.]*

**Strong follow-up you should volunteer:** centrality is a double-edged signal. It biases toward well-studied entities, which is exactly wrong when the user is asking about *emerging* treatments. I'd weight it low or make it intent-conditional — down-weighted for novelty-seeking queries. Raising this yourself signals real depth.

---

### Q3.3 — "Why RRF vs. weighted score fusion?"

Reciprocal Rank Fusion is rank-based, so it sidesteps score normalization entirely and is very robust — a good default. Weighted score fusion retains magnitude information, which matters when a signal is genuinely calibrated (extraction confidence is meaningful; the gap between rank 1 and rank 2 in RRF is fixed regardless of how much better rank 1 actually is).

*[FILL IN which you used.]* If you used weighted fusion, be ready to explain your normalization. If RRF, be ready to explain how you injected confidence, since RRF alone discards it.

---

### Q3.4 — "Why BioBERT/PubMedBERT instead of a modern general embedding model?"

Domain pretraining matters most where general models have the least signal: gene symbols, mutation notation, drug names, and biomedical acronyms are effectively out-of-distribution for general web-trained encoders. `EGFR T790M` and `EGFR C797S` are near-identical strings with completely different clinical implications — a domain encoder separates them, a general one often doesn't.

**Caveat I'd raise unprompted:** these are relatively old, BERT-scale, 512-token models. Modern general-purpose embedding models have closed much of the domain gap and handle longer context. The right answer is empirical — I'd benchmark domain vs. general vs. a fine-tuned general model on my own retrieval eval set rather than assume. *[FILL IN if you ran this comparison — it's a strong thing to have done.]*

---

### Q3.5 — "How do you bound graph traversal? Won't it explode?"

Yes, without controls — biomedical graphs are dense and hub nodes like `cancer` or `TP53` connect to enormous neighborhoods.

Controls:
- **Depth limits**, typically 2–3 hops; useful biomedical reasoning chains are short and beyond 3 hops relevance collapses.
- **Typed path templates** per intent — trial matching follows `patient_profile → mutation → gene → trial_eligibility`, not arbitrary edges.
- **Hub penalties** — traversal cost scales with node degree, so paths through super-hubs are discouraged.
- **Beam search over paths** rather than full expansion, scored incrementally, keeping top-k partial paths.
- **Edge confidence thresholds** to prune low-confidence relations early.

---

### Q3.6 — "What's the reranker and why do you need one after fusion?"

Fusion is cheap and approximate — it operates on precomputed representations. A cross-encoder reads query and candidate jointly and is substantially more accurate at the top of the list, which is what actually reaches the agent's context. Standard cascade: cheap and broad, then expensive and narrow over top-k.

Rerank on the top ~50–100 candidates, not the full set — the whole point is confining the expensive step.

---

### Q3.7 — "How do you handle a query with no good match?"

Explicit abstention path. If fused scores fall below threshold and graph anchors are unresolvable, the system says the evidence is insufficient rather than returning the least-bad chunks.

This matters more here than in most RAG systems: in consumer search, a mediocre result is a minor annoyance; in clinical decision support, a confidently-presented weak match is a safety issue. Designing for "I don't know" as a first-class output was deliberate.

---

## 4. Multi-Agent Architecture

### Q4.1 — "Why multi-agent? Isn't this just added complexity and latency?"

Fair challenge, and the honest answer is that multi-agent is often over-applied. Here it earned its place for specific reasons:

- **Genuinely heterogeneous tools.** The deep-research subagent needs web + literature search. The medical subagent needs Tx-Gemma and TDC tools. Different tool sets, different prompting, different failure modes. One agent with 20 tools degrades — tool selection accuracy falls off as the tool count grows.
- **Parallelism.** Independent subtasks run concurrently, so the multi-hop investigation is wall-clock-bounded by the slowest branch, not the sum.
- **Context isolation.** Each subagent gets a clean, focused context. A single agent doing a long multi-hop investigation accumulates a polluted context where early irrelevant results degrade later reasoning.
- **Independent verification.** Two agents reaching a conclusion via different evidence paths is meaningfully stronger than one agent asserting it twice.

**The cost, stated plainly:** latency, token spend, and much harder debugging. For a simple factual lookup, the orchestrator should and does route to a fast path rather than spinning up the full pipeline. Not every question deserves a research team.

---

### Q4.2 — "How does the orchestrator decompose a question? What if it decomposes badly?"

Decomposition is an LLM planning step with a structured output schema — subtasks typed by which subagent handles them, with declared dependencies so independent ones parallelize.

Bad decomposition is the dominant failure mode of the whole system, so:
- **Plan validation** before execution — schema checks, subtask count bounds, entity resolvability checks. A subtask referencing an entity not in the KG and not resolvable on the web is caught before burning an agent run.
- **Replanning on failure** — if subagents return low-confidence or contradictory results, the orchestrator can revise the plan rather than synthesizing garbage. Bounded replan count to prevent loops.
- **Plan visibility** — the plan is part of the audit trail. When output is wrong, the first debugging question is almost always "was the decomposition right?", and I can answer it directly.

---

### Q4.3 — "Explain ReAct. Why ReAct for the subagents?"

ReAct interleaves reasoning and acting: the model reasons about what it needs, calls a tool, observes the result, and reasons again with that observation in context. The loop continues until it can answer or hits a budget.

It fits here because biomedical research is genuinely iterative — you find a paper that names a mechanism you didn't know about, which changes what you search next. A plan-then-execute agent can't do that; it commits to a search strategy before it knows what's out there.

**Controls I put on the loop:** max iterations, per-agent token budget, and repetition detection — ReAct agents fall into loops where they re-issue near-identical searches, and without detection they burn budget going nowhere.

---

### Q4.4 — "Tell me about Tx-Gemma and why you self-hosted it."

Tx-Gemma is a Gemma-derived model family from Google specialized for therapeutic development tasks — property prediction, drug-target interaction, trial outcome prediction — trained on Therapeutic Data Commons data. It's the specialist that a general LLM can't replicate: general models have no reliable grasp of molecular property prediction.

Self-hosting via vLLM for four reasons:
1. **Data control** — clinical queries can contain sensitive context; keeping inference in-infrastructure avoids third-party data flow entirely.
2. **Cost** — high-volume specialist calls in an agent loop; per-token API pricing dominates fast.
3. **Latency** — no network round trip, and no shared-tenancy variance in an already-multi-step pipeline.
4. **Version pinning** — a silently-updated hosted model invalidates my evals. Pinned weights mean reproducible results.

vLLM specifically for continuous batching and PagedAttention — with bursty agent traffic, throughput per GPU was the binding constraint and vLLM's batching is what made it economical.

---

### Q4.5 — "What is BioMni and how does it fit?"

BioMni is a biomedical AI agent framework providing a large library of tools and datasets for biomedical research tasks. It gave the research subagents a ready set of domain tool integrations rather than my hand-building each one.

**If you're not deeply familiar with its internals, say what you used it for and don't oversell.** "I used it for X specific capability" is a strong answer; a vague claim to have architected on top of it invites questions you can't answer.

---

### Q4.6 — "Why Google ADK, and what does MCP do here?"

**ADK** provides the agent scaffolding — agent definitions, tool registration, session and state management, and multi-agent composition patterns including the orchestrator/subagent structure I needed. It saved me from writing orchestration plumbing, and it plays well with the Google model ecosystem I was already using for Tx-Gemma.

**MCP** is the tool interface layer. Rather than bespoke integrations per tool, tools are exposed over a standard protocol, so adding a data source doesn't mean touching agent code. Practically this meant the KG query interface, TDC tools, and web search all present the same shape to the agent, and I could swap or add tools without redeploying agent logic.

---

### Q4.7 — "How do agents share state? What stops them from duplicating work?"

Shared session state managed by the orchestrator — subagents write findings back to a common scratchpad keyed by subtask, and the orchestrator reads it for synthesis. Subagents do *not* share full context; they see their subtask, relevant prior findings, and their tools. That isolation is the point (Q4.1).

Duplicate work: the orchestrator dedupes at plan time by design, and retrieved-document IDs are tracked in shared state so a second agent retrieving the same paper recognizes it rather than re-processing.

---

### Q4.8 — "How do you debug this when it gives a wrong answer?"

Full trace persistence — plan, every subagent's tool calls and observations, retrieved doc IDs, the KG snapshot version, and the final synthesis. Debugging walks backward: was the answer a synthesis error, a subagent reasoning error, a retrieval miss, or a bad KG edge? Each has a different fix, and without the trace you're guessing.

This is the operational payoff of the auditability design. It wasn't built for compliance alone — it's what made the system debuggable at all.

---

## 5. Tournament-Style Ranking

### Q5.1 — "Why pairwise tournament instead of just scoring each treatment 0–10?"

LLMs are substantially better at relative than absolute judgment. Ask for a 0–10 score and you get clustering around 7–8 with poor discrimination and no stability across runs. Ask "which of these two is better on evidence strength" and you get a far more reliable signal.

Pairwise comparisons then aggregate into a global ranking. It also produces the artifact I actually want: for any two treatments, an explicit recorded justification for why one ranked above the other. That's directly presentable to a clinician in a way that "7.5 vs 7.2" is not.

---

### Q5.2 — "Pairwise is O(n²). How do you handle that?"

For typical candidate counts (roughly 5–20 treatments) full round-robin is fine and gives the most reliable ranking.

Beyond that: Swiss-style pairing or an Elo-style approach where candidates are matched against similarly-rated opponents, which converges to a stable ranking in O(n log n) comparisons. You lose some precision in the middle of the table, but the top of the ranking — the part anyone reads — stabilizes quickly.

---

### Q5.3 — "LLM judges have position bias. How did you address it?"

Real and well-documented — the same pair in reverse order can flip the verdict. Mitigations:

- **Order randomization** across matchups.
- **Dual evaluation** for high-stakes pairs — run A/B and B/A; disagreement is itself signal, recorded as low-confidence rather than resolved by coin flip.
- **Structured criteria** — the judge scores against the five explicit dimensions (evidence strength, safety, guideline support, patient fit, mechanistic plausibility) rather than giving a holistic preference. Decomposed judgments are more stable and more auditable.
- **Verbosity control** — LLM judges favor longer responses, so candidate descriptions are length-normalized before comparison.

---

### Q5.4 — "How do you weight the five criteria? Isn't safety not commensurable with mechanistic plausibility?"

Correct, and I don't treat them as commensurable. Safety and contraindications act as a **gate**, not a weighted term — a contraindication for the specific patient profile disqualifies rather than deducting points. You cannot let strong mechanistic plausibility outweigh a hard contraindication; that's a patient-harm bug.

The remaining criteria are weighted, with evidence strength and guideline support dominant, and mechanistic plausibility deliberately lowest — it's the criterion most vulnerable to plausible-sounding LLM confabulation and the one with the weakest link to clinical outcome.

---

### Q5.5 — "Isn't intransitivity a problem? A beats B, B beats C, C beats A."

Yes, it happens, and I treat it as signal rather than noise to smooth away. A cycle means the candidates are genuinely close or the criteria conflict across the pairs. The system detects cycles and surfaces those candidates as a tier — "these three are comparable on current evidence" — rather than manufacturing false precision in the ordering.

Presenting an arbitrary total order over genuinely ambiguous evidence is exactly the kind of overconfidence this project exists to avoid.

---

## 6. Hallucination & Grounding

### Q6.1 — "You claim reduced hallucination. How do you measure it?"

*[This needs your real numbers. Here's the framework and what to fill in.]*

Measurement approach:
- **Claim-level groundedness** — decompose the answer into atomic claims, check each against retrieved evidence. Report the proportion of claims with adequate support. Automated with an NLI-style entailment check, validated against human annotation on a sample.
- **Citation accuracy** — does the cited source actually support the claim? Cited-but-unsupported is a distinct and more insidious failure than uncited, because it *looks* grounded.
- **Human expert review** on a held-out clinical question set — the ground truth, expensive, so sampled.

*[FILL IN: baseline system, dataset, n, and the actual delta. "Reduced hallucination" without a baseline comparison is not a result.]*

---

### Q6.2 — "What specifically in your architecture reduces hallucination?"

Four mechanisms, in rough order of impact:

1. **Retrieval grounding with hard provenance** — the synthesis step is constrained to cite from retrieved evidence; a claim without an attachable source ID is flagged in post-processing.
2. **KG constraint** — relations asserted in the answer are checkable against graph edges. If the model asserts drug X targets gene Y and no such edge exists, that's detectable programmatically, which is not true of free-text RAG.
3. **Multi-agent cross-checking** — independent agents reaching conclusions via different evidence paths. Disagreement surfaces rather than hides.
4. **Abstention path** — a system that can say "insufficient evidence" hallucinates less than one forced to always answer.

**Honest caveat worth volunteering:** none of this eliminates hallucination. It reduces it and, more importantly, makes remaining errors *detectable* by a reviewing clinician, because every claim carries a checkable source. Detectability may matter more than raw rate for clinical safety.

---

### Q6.3 — "What if the retrieved evidence itself is wrong?"

Then the system is confidently wrong with a citation, which is arguably worse than being uncited. Mitigations: source quality weighting (peer-reviewed and trial registry above preprint above web), retraction checking, evidence-hierarchy weighting in ranking, and requiring multiple independent sources for high-stakes claims.

But I'd state the limit clearly: the system's ceiling is the quality of the biomedical literature, and that literature has real replication problems. This is a fundamental reason the product is decision *support* with visible sources — the clinician can see that the support is a single small study and weight it accordingly. The alternative, hiding the evidence base behind a confident answer, is what makes these systems dangerous.

---

## 7. Engineering & Infrastructure

### Q7.1 — "Walk me through the serving architecture."

FastAPI as the API layer — async, which matters when most request time is waiting on model and DB calls. Long-running research requests are submitted as jobs rather than held open: FastAPI accepts, enqueues to Celery, returns a job ID, and results stream back over SSE/websocket as agents complete. A full agentic research run is far beyond any reasonable HTTP timeout.

Celery workers in separate queues by workload class — ingestion, extraction, agent execution — so a batch ingestion backlog can't starve interactive queries. Neo4j for the graph, vector DB for embeddings, vLLM serving self-hosted models on GPU nodes.

---

### Q7.2 — "What's your latency, and how do you make it acceptable?"

*[FILL IN actual numbers — p50/p95 for simple queries vs. full research runs.]*

Design for it:
- **Tiered routing** — simple factual lookups take a fast path (retrieval + single synthesis, seconds). Only genuine research questions get the full agentic pipeline (minutes).
- **Streaming and progressive disclosure** — show the plan immediately, then subagent findings as they land. Perceived latency is what users experience; a visible working system tolerates far more wall clock than a spinner.
- **Parallel subtask execution** across independent branches.
- **Caching** at multiple layers — embeddings, frequent subgraph queries, and semantically-similar query results with a KG-version-aware key so cache doesn't serve stale evidence after an update.

**Framing that lands:** for a clinician researching treatment options, a three-minute thorough answer beats a five-second shallow one. I optimized for the right latency target per query class rather than uniformly for speed.

---

### Q7.3 — "How do you control cost?"

Multi-agent pipelines are token-expensive by nature. Levers:
- **Model tiering** — small fast models for routing, classification, and extraction; frontier models only for planning and final synthesis. This is the single biggest lever.
- **Self-hosting** the high-volume specialist calls (Tx-Gemma via vLLM).
- **Aggressive caching** — ingestion-time extraction is the bulk of LLM spend and is fully cacheable by document hash.
- **Budget enforcement** — per-request token ceilings, so a pathological ReAct loop has a bounded blast radius.
- **Routing** — not every question triggers the full pipeline.

---

### Q7.4 — "How would you scale this to 10x the corpus?"

Ingestion scales horizontally — it's embarrassingly parallel, so more Celery workers, with LLM inference throughput as the real constraint. Vector search scales with sharding and is well-understood.

Neo4j is where it gets interesting: write throughput and memory become the pressure points. Options in order of preference — better indexing and query optimization first (most "graph scaling problems" are unindexed traversals), then read replicas for the read-heavy workload, then partitioning by domain (oncology and neurology subgraphs are largely separable, which is a genuine advantage of the narrow scope), and only then a distributed graph store.

I'd also expect the KG quality bar to become the harder problem before infrastructure does. 10x the documents at the same extraction precision means 10x the bad edges, and graph quality degrades non-linearly with noise because errors compound along traversal paths.

---

### Q7.5 — "How do you test a nondeterministic system?"

Layered:
- **Deterministic unit tests** for everything non-LLM — chunking, canonicalization, graph queries, fusion math. This is more of the system than people assume, and it's where most bugs actually live.
- **Golden-set regression** for retrieval — fixed queries, expected documents, tracked metrics. Retrieval is largely deterministic given a fixed index, so this is a real regression gate.
- **LLM-as-judge evals** for generation quality, run against a versioned KG snapshot so changes are attributable.
- **Trace assertions** for agent behavior — not "did it produce this exact text" but "did it call the KG tool," "did it stay within iteration budget," "did every claim carry a source."
- **Canary + human review** on a sampled slice of production traffic.

Pinning model versions and KG snapshots is what makes any of this interpretable. Without both pinned, you can't tell whether a metric moved because of your change.

---

## 8. Evaluation & Results ⚠️ HIGHEST-RISK SECTION

### Q8.1 — "You say hybrid retrieval 'significantly outperforms traditional biomedical RAG.' Outperforms what, by how much, on what?"

**This is the question most likely to damage you, and your project page currently gives you nothing to answer with.** Prepare this before anything else in this document.

You need four things:

| Component | What you must be able to state |
|---|---|
| **Baseline** | The specific comparison system — naive chunk-and-embed RAG over the same corpus? BM25? An off-the-shelf biomedical RAG? |
| **Dataset** | Which queries, how many, where the ground truth came from, who labeled it |
| **Metrics** | Recall@k, nDCG@k, MRR for retrieval; groundedness/citation accuracy/expert rating for answers |
| **Delta** | Actual numbers, with a sense of whether the difference is meaningful given the sample size |

Public benchmarks worth mentioning if you used them or would: BioASQ, PubMedQA, MedQA, MIRAGE/MedRAG. Using a recognized benchmark alongside your internal set is a strong signal — it makes your numbers comparable to something.

**If you genuinely don't have rigorous numbers, the recoverable answer is:** "I evaluated on an internal set of N queries with expert-labeled relevant documents and saw [directional result]. I want to be careful not to overstate it — it wasn't a published-benchmark comparison, and the set was small enough that I'd treat it as directional. If I were continuing, the first thing I'd do is run against MIRAGE for a comparable number."

That answer is respected. A confident unfounded number that collapses under one follow-up is not.

---

### Q8.2 — "How do you evaluate the *ranking* quality? There's no ground truth for 'best emerging treatment.'"

Correct — and worth saying out loud, because recognizing that the ground truth is genuinely contested is a mark of seriousness.

Approaches:
- **Expert agreement** — clinician ranks the same candidate set; measure rank correlation. Also measure inter-clinician agreement, because if two oncologists disagree with each other, that's the realistic ceiling for the system.
- **Retrospective validation** — for historical questions, check whether treatments the system would have ranked highly subsequently gained approval or positive Phase III results. Time-travel evaluation using the KG's temporal versioning: restrict the graph to a past date and see what it would have said.
- **Justification quality** — separate from ranking accuracy: are the stated reasons factually correct and well-sourced? A right answer for wrong reasons is a latent failure.

The retrospective/time-travel evaluation is the most compelling and directly leverages the bitemporal design from Q2.7. If you did any of it, lead with it.

---

### Q8.3 — "What didn't work? What would you do differently?"

Have two or three real answers ready. Strong candidates for this project:

- **Canonical linking on the long tail** was harder than planned and consumed disproportionate effort. If restarting, I'd scope the ontology tighter from day one and accept flagged-unlinked entities earlier rather than chasing coverage.
- **Multi-agent latency and cost** exceeded initial estimates; routing simple queries away from the full pipeline was a necessary retrofit that should have been in the design from the start.
- **Evaluation was built too late.** Building the eval harness before the third architectural iteration would have made every subsequent decision faster and more defensible.

*[Replace with your actual ones — specificity is the whole value here. Generic "I'd add more tests" answers are worse than nothing.]*

---

## 9. Design Tradeoffs (Staff/Senior-Level Probes)

### Q9.1 — "What's the weakest part of this system?"

Answer this well and it's the strongest signal you can send. A candidate who can accurately name their system's weakness is trusted on everything else they said.

Defensible picks:
- **Extraction quality is the foundation, and it's LLM-based, which means it's imperfect and the errors are not random.** Everything downstream inherits that. A wrong edge doesn't just produce one wrong answer — it corrupts every traversal through it, and it arrives wearing full provenance metadata that makes it look trustworthy.
- **Evaluation rigor** — see section 8.
- **KG freshness vs. cost** — the graph lags the literature by the ingestion cycle, which in a fast-moving oncology subfield is a real gap.

---

### Q9.2 — "If you had to cut this system in half, what goes?"

The multi-agent layer, before the KG. The KG plus hybrid retrieval delivers most of the grounding, explainability, and provenance value — that's the differentiated core. The agentic layer adds depth on genuinely complex research questions, but a large fraction of real queries are answerable with good retrieval and single-pass synthesis.

Cutting the KG instead would leave a conventional RAG system with a research agent bolted on, which is a commodity. The KG is the moat.

---

### Q9.3 — "Where does this break in production with real clinicians?"

- **Query phrasing mismatch** — clinicians use shorthand, institutional abbreviations, and dictation artifacts that don't match literature phrasing. Query understanding is the fragile point.
- **Patient context is rich and unstructured** — real cases have comorbidities, prior lines, performance status, and organ function that all constrain treatment. The system needs structured intake or it under-constrains and recommends things the patient can't tolerate.
- **Latency vs. clinical workflow** — a clinician has minutes between patients. A three-minute research run doesn't fit at point of care; it fits tumor board prep. Knowing which workflow you're actually in changes the whole product.
- **Trust calibration** — clinicians correctly distrust AI output. The provenance UI is not a nice-to-have; it's the adoption mechanism.

---

### Q9.4 — "What would v2 look like?"

Two or three concrete, ambitious-but-grounded directions:
- **Structured patient intake** — FHIR ingestion so patient context is structured rather than free text, enabling far tighter eligibility matching.
- **Active learning from clinician feedback** — when a clinician rejects a ranked option, capture why and feed it back into ranking criteria weights and extraction review queues.
- **Continuous eval in production** — sampled expert review as a standing process with drift monitoring, rather than point-in-time benchmarking.

---

## 10. Safety, Ethics, Regulatory

### Q10.1 — "Is this a regulated medical device?"

Know the landscape and be precise. The relevant frame in the US is clinical decision support software: broadly, software that provides recommendations to a healthcare professional while displaying the basis for those recommendations — such that the professional can independently review it and doesn't rely primarily on the software — sits in a different regulatory posture than software that directs a clinical decision or that the clinician must simply trust.

**How that shaped design, which is the real point of the question:** the transparency layer isn't only for user trust, it's what keeps the system in the "reviewable recommendation" posture. Showing sources, reasoning traces, and evidence quality lets the clinician independently evaluate the basis rather than taking output on faith. A version that gave a confident recommendation without exposing its evidence would be a different product with a different regulatory profile.

Patient-facing output is a separate and more sensitive question, with clear scoping needed around what it will and won't say.

**Say clearly:** you're an engineer, not a regulatory expert, and shipping clinically would require formal regulatory review. Confidently over-claiming regulatory knowledge is a bad look; understanding that it constrains architecture is an excellent one.

---

### Q10.2 — "How do you handle PHI?"

Principles: minimize (structured clinical attributes — mutation, stage, prior therapy — rather than identifiers), keep sensitive inference in-infrastructure (a major reason for self-hosting), encrypt in transit and at rest, audit-log all access, and never let patient context leak into the KG, which is a shared knowledge artifact and must stay patient-agnostic.

That last one is a design invariant worth stating explicitly: the KG holds published knowledge, patient context lives only in the request scope. Mixing them would be both a privacy failure and a correctness failure.

---

### Q10.3 — "What if it recommends something harmful?"

Layered defense: safety and contraindications as a hard gate rather than a weighted criterion (Q5.4); mandatory display of evidence quality so weak support is visible; abstention when evidence is insufficient; explicit framing as decision support with clinician-in-the-loop; and clear scope boundaries — the system does not dose, does not diagnose, and does not tell patients to change treatment.

**Don't be defensive here.** The right posture is "here are the specific mechanisms, and here is what they don't cover." Claiming a safety-critical system is fully safe is the answer that fails.

---

### Q10.4 — "Health equity and bias in the underlying literature?"

Biomedical literature is not a neutral sample — clinical trials have historically underrepresented certain populations, so evidence strength for a treatment is often evidence strength *in the studied population*. A system that ranks purely on evidence strength will systematically favor treatments studied in well-represented groups.

Mitigations: surface trial population demographics as metadata so a clinician can see the mismatch, and treat "evidence exists but not in this population" as distinct from "evidence is weak." Fully solving this isn't possible downstream of the literature — but making the gap visible instead of laundering it into a confident ranking is both achievable and the right thing to do.

---

## 11. Rapid-Fire Technical

Short answers; these come as follow-ups.

**Cypher vs. Gremlin?** Cypher — declarative, readable, better ecosystem for the traversal patterns I needed.

**Which vector DB and why?** *[FILL IN.]* Be ready on filtered search performance — metadata filtering (date, study type, indication) was a hard requirement, and pre- vs post-filtering behavior differs meaningfully across engines.

**HNSW parameters?** Know your `M` and `ef_construction`/`ef_search`, and the recall-vs-latency tradeoff they control.

**Chunk size?** *[FILL IN]* — and defend it with the semantic-boundary rationale from Q2.4, not a round number.

**How many nodes/edges/documents?** *[FILL IN.]* You will be asked. Not knowing your own graph's scale is a bad moment.

**Airflow vs. Prefect/Dagster?** Airflow for maturity and operator ecosystem. Acknowledge Dagster's better data-asset model and testing story as a legitimate alternative — showing you know the tradeoff beats defending the choice absolutely.

**Celery vs. Airflow — why both?** Airflow orchestrates scheduled DAGs and dependencies; Celery handles high-fanout parallel task execution and interactive async work. Airflow's task slots are the wrong abstraction for thousands of short parallel LLM calls.

**Embedding dimensionality / index size?** *[FILL IN]* — and know your index memory footprint.

**How do you handle KG updates without downtime?** Soft updates with bitemporal edges (Q2.7), so writes don't block reads and no version is destructively overwritten.

---

## 12. Behavioral Questions (STAR format)

For each, prepare a **specific** story. Vague answers fail these more than wrong answers do.

### Q12.1 — "Tell me about a hard technical decision and how you made it."
Best candidate: KG + hybrid retrieval vs. pure vector RAG. Frame as: constraint (explainability requirement) → options considered → what you tested → decision → what it cost you (complexity, ingestion expense) → whether you'd redo it.

### Q12.2 — "Tell me about something that failed."
Use a real one. Section 8.3 candidates work. Structure: what you expected, what happened, what you learned, what you changed. The learning must be specific and the change must be real.

### Q12.3 — "How did you prioritize with limited time?"
Good frame: KG quality over agent sophistication. A sophisticated agent over a bad graph is a fast way to be confidently wrong. Foundation before capability.

### Q12.4 — "How did you validate you were building the right thing?"
*[FILL IN — did you talk to clinicians or researchers? If yes, this is one of your strongest available answers and you should have a specific anecdote about something a domain expert told you that changed the design. If no, say what you'd do differently.]*

### Q12.5 — "What are you most proud of?"
Recommend: the auditability design — that every claim resolves to a source, an extraction, a reasoning step, and a graph version. Not because it's the flashiest part, but because it's what makes the system usable in a domain where an unverifiable answer is worthless. It's also the part that made everything else debuggable.

### Q12.6 — "Explain this to a non-technical stakeholder."
Practice a 60-second version with no jargon: *"Medical AI tools today give you an answer but not a reason. Ours builds a map of what the medical literature actually says — which drugs affect which genes, which trials are recruiting, which studies contradict each other — and then researches your question against that map. Every sentence in the answer links back to the paper it came from, so a doctor can check the work instead of trusting it."*

---

## 13. Questions to Ask Them

Ask 3–4. Good ones signal seniority.

1. How do you evaluate model quality here, and who owns that — is there a dedicated eval function or does it sit with the engineers shipping features?
2. What's the relationship between engineering and clinical/domain experts? Are clinicians in the loop during development or only at review?
3. Where is the team on the build-vs-buy line for foundation models — self-hosting, fine-tuning, or API-first?
4. What's the biggest source of technical debt right now?
5. How do you handle the regulatory dimension — is there in-house regulatory expertise engineers can consult, or does that come later?
6. What does success look like for this role in six months?

---

## 14. Pre-Interview Checklist

Do these in order:

- [ ] **Fill in every `[FILL IN]` in this doc.** These are the exact spots where an interviewer will find the seam.
- [ ] **Get your numbers straight** — KG scale (nodes/edges/docs), latency, and above all your retrieval evaluation. Section 8 is the highest-risk area by a wide margin.
- [ ] **Add metrics to your project page.** "Significantly outperforms" with no number is currently working against you, not for you.
- [ ] **Rehearse Q1.1 out loud until it's 90 seconds and doesn't sound recited.**
- [ ] **Prepare three genuine "what went wrong" stories** with specific technical detail.
- [ ] **Draw the architecture from memory on a blank page.** You will be asked to whiteboard it.
- [ ] **Refresh Tx-Gemma, BioMni, ADK, and MCP specifics** — you listed them, so they're fair game, and surface-level familiarity with a tool you claimed shows immediately.
- [ ] **Decide your honest scope statement.** If parts were prototype rather than production, or if you worked with others, say so proactively. Overclaiming is the fastest way to lose a room; owning scope precisely reads as confidence.
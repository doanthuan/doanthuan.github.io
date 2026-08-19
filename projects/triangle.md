---
layout: page
title: Triangle Health — Agentic Graph-Retrieved Reasoning
permalink: /projects/triangle/
---

# Triangle Health

**Date:** 2025-2026

**URL**: [https://www.trianglehealth.com](https://www.trianglehealth.com)

**Project Overview:**
![Data-Pipelines](/assets/triangle/overview.png)
- Triangle Health transforms clinical corpora into a provenance-aware biomedical knowledge graph and layers hybrid GraphRAG retrieval with agentic research to deliver grounded, explainable answers for patients, clinicians, and researchers. Initial focus: oncology and neurology.

## System Overview

Clinicians and researchers asking questions like *"what emerging treatments exist for an EGFR-mutant NSCLC patient who progressed on osimertinib?"* get answers from standard LLMs that are fluent, plausible, and unverifiable — and in medicine, unverifiable is unusable.

I designed and built an **Agentic Graph-Retrieved Reasoning Framework** that answers such questions with evidence a clinician can independently check. It integrates three layers:

1. A **provenance-aware biomedical knowledge graph** where every node and edge traces back to a source document and extraction step
2. A **hybrid semantic–graph retrieval engine** that returns subgraphs with reasoning traces, not bags of text chunks
3. A **multi-agent reasoning layer** that decomposes clinical questions, researches them in parallel, and ranks treatment options with explicit justification

![System-Overview](/assets/triangle/system_overview.png)

The through-line is **auditability**: every claim in a final answer resolves to a source document, an extracted relation, an agent reasoning step, and a versioned KG entity. The system is decision *support*, not a decision maker — it surfaces evidence and ranked options with justification; a clinician decides. It serves three personas with different surfaces: researchers (literature and mechanism exploration), clinicians (trial matching and treatment options for a specific patient profile), and patients (grounded plain-language explanations with citations).

Oncology came first because mutation-driven treatment selection is inherently a graph problem (variant → gene → pathway → drug → trial) and the trial landscape moves faster than any human can track; neurology second as a stress test with sparser, noisier evidence. Narrow scope also made high-precision entity canonicalization tractable.

---

## 🌐 Key Features

### 1. Provenance-Aware Biomedical Knowledge Graph
![KnowledgeGraph](/assets/triangle/kg_construction.png)
The KG is built from ClinicalTrials.gov, PubMed, Semantic Scholar, OpenAlex, and PrimeKG via per-source **Airflow DAGs** scheduled to each source's update cadence, with **Celery** fanning out the embarrassingly-parallel per-document work. The pipeline:

- **Fetch, normalize, and dedupe** — cross-source dedup on DOI/PMID/NCT ID plus fuzzy title+author matching, so overlapping sources don't triple-count evidence and corrupt confidence weighting
- **Semantic chunking** — split on document structure first (abstract sections, IMRaD headings, trial eligibility blocks), then at low-similarity semantic boundaries; fixed-size chunking routinely severs a finding from its qualifier, which is actively dangerous in this domain
- **Contextual embedding** — each chunk is prefixed with a compact context header (title, section, study type, canonical entities) drawn from the KG, keeping chunks retrievable when they use pronouns or shorthand
- **LLM structured summarization** — papers into a population/intervention/comparator/outcome schema; trials into phase, status, eligibility, arms, endpoints
- **Layered entity extraction** — a gazetteer pass against UMLS, MeSH, DrugBank, and HGNC (high precision, near-zero cost) plus an LLM pass for novel compounds and unusual mutation notation, then reconciliation
- **Canonical linking** — surface forms mapped to UMLS CUIs, MeSH, OMIM, and DrugBank IDs, with context-based disambiguation for polysemous mentions (e.g. "ALS"), document-local abbreviation tables, and type constraints; unresolvable mentions are kept flagged `canonical: false` rather than silently dropped
- **Relation extraction** — typed edges written to **Neo4j**, chunk embeddings to the vector store, both keyed by the same canonical entity IDs so retrieval can join across them

**Every edge carries full provenance:** supporting document IDs, extraction method and model version, calibrated confidence, the evidence spans that produced the assertion, an assertion type (asserted / negated / hypothesized / contradicted), and bitemporal validity.

**Contradictions are represented, not resolved away** — two papers that disagree coexist as explicitly conflicting edges, weighted downstream by evidence hierarchy (meta-analysis and Phase III above Phase II above case reports above preclinical). Retracted papers are flagged against retraction databases and excluded.

**The KG is bitemporally versioned**: superseded edges are closed, not deleted, so any past answer can be reconstructed against the exact graph state that produced it, and snapshot tags keep evaluation results reproducible.

---

### 2. Hybrid Graph + Semantic Retrieval
![KnowledgeGraphRetrieval](/assets/triangle/kg_retrieval.png)
Vector RAG alone can't compose multi-hop relational queries ("which drugs target a pathway downstream of a gene mutated in this tumor?"), and a graph alone is bad at fuzzy, novel-phrasing queries — so the retrieval engine runs both and fuses:

1. **Query understanding** — entities in the question are extracted and canonicalized with the *same* linking pipeline as ingestion, so query entities and graph entities share one ID space; intent is classified (trial matching vs. mechanism vs. comparison)
2. **Dense retrieval** — BioBERT / PubMedBERT embeddings with ANN search, filtered by hard constraints (date, study type, indication); domain encoders separate near-identical strings like `EGFR T790M` vs. `EGFR C797S` that general encoders conflate
3. **Graph retrieval** — traversal anchored on canonicalized query entities, bounded by **typed path templates per intent**, 2–3 hop depth limits, hub penalties on super-connected nodes, beam search over paths, and edge-confidence pruning
4. **Fusion scoring** — semantic similarity, graph relevance (path length, relation types, number of independent connecting paths), extraction confidence, and node centrality, normalized to a comparable scale
5. **Cross-encoder reranking** over the top candidates — cheap and broad first, expensive and narrow last
6. **Assembly** — a subgraph, supporting chunks, provenance metadata, and the traversal path as a reasoning trace

The engine has an **explicit abstention path**: if fused scores fall below threshold, it reports insufficient evidence rather than returning the least-bad chunks. In clinical decision support, a confidently-presented weak match is a safety issue, so "I don't know" is a first-class output.

This module supports clinical trial matching, mutation–treatment alignment, and case-report similarity scoring.

---

### 3. Multi-Agent Scientific Research and Ranking
![AgentResearchRanking](/assets/triangle/agent_research_ranking.png)
The reasoning layer is built around a **Medical Deep Research orchestrator agent** that plans the investigation, delegates to specialized subagents, and synthesizes their findings. Multi-agent earned its complexity here for concrete reasons: heterogeneous tool sets per subagent, parallel execution of independent subtasks, clean per-agent context isolation, and independent verification — two agents reaching a conclusion via different evidence paths is stronger than one asserting it twice.

#### 🧭 Orchestrator: Medical Deep Research Agent
- Decomposes a clinical question into typed subtasks with declared dependencies, so independent branches run in parallel
- **Validates plans before execution** (schema checks, subtask bounds, entity-resolvability checks) and **replans** — bounded to prevent loops — when subagents return low-confidence or contradictory results
- Routes simple factual lookups to a fast retrieval-plus-synthesis path; only genuine research questions get the full pipeline
- The plan itself is part of the audit trail

#### 🔍 Deep Research Subagent
- ReAct agent with web and biomedical knowledge-base tools — reasoning and searching interleave because research is genuinely iterative: a paper naming an unknown mechanism changes what you search next
- Loop controls: max iterations, per-agent token budgets, and repetition detection to catch agents re-issuing near-identical searches

#### 🧠 Medical Subagent
- ReAct agent backed by a **self-hosted Tx-Gemma model** (served on **vLLM** for continuous-batching throughput) and **Therapeutic Data Commons (TDC) tools**, with **BioMni** providing the biomedical tool library
- Self-hosting was deliberate: clinical queries stay in-infrastructure, high-volume specialist calls stay economical, and pinned weights keep evals reproducible
- Clinical interpretation aligned with disease phenotype and mutation profile, with mechanistic evaluation and explicit citations

Agents share state through an orchestrator-managed scratchpad keyed by subtask — subagents see their task, relevant prior findings, and their tools, not each other's full context — and retrieved-document IDs are tracked so no agent re-processes a paper another already found. **LangChain** provides the agent scaffolding and **MCP** the tool interface layer, so the KG query interface, TDC tools, and web search all present the same shape to agents and new tools don't require touching agent code.

![MedicalResearchAgent](/assets/triangle/medical_research_agent.png)

#### 🏆 Tournament-Style Treatment Ranking
Candidate treatments are ranked through **pairwise matchups** rather than absolute 0–10 scoring — LLMs are substantially better at relative than absolute judgment, and pairwise comparison produces the artifact a clinician actually wants: an explicit recorded justification for why one option ranked above another.

Each matchup is judged on structured criteria:

- Evidence strength and biological targeting
- Safety and contraindications — a **hard gate**, not a weighted term: a contraindication for the specific patient profile disqualifies outright
- Guideline and regulatory support
- Patient-specific fit
- Mechanistic plausibility — deliberately weighted lowest, as the criterion most vulnerable to plausible-sounding confabulation

Judge reliability is engineered, not assumed: matchup order is randomized against position bias, high-stakes pairs are evaluated in both orders with disagreement recorded as low confidence, and candidate descriptions are length-normalized against verbosity bias. Intransitive cycles (A beats B beats C beats A) are surfaced as a tier of comparable options rather than smoothed into false precision.

**Outcome:** a ranked list of emerging and high-confidence treatments with transparent, per-comparison justification.

---

## 📈 Grounding & Evaluation

Rather than trusting the model to be truthful, the architecture makes claims **checkable**, through four mechanisms:

1. **Hard provenance constraints** — synthesis must cite from retrieved evidence; any claim without an attachable source ID is flagged in post-processing
2. **KG constraint checking** — relations asserted in an answer are programmatically verifiable against graph edges, which free-text RAG cannot offer
3. **Multi-agent cross-checking** — independent agents reaching conclusions via different evidence paths, with disagreement surfaced rather than hidden
4. **Abstention** — a system that can say "insufficient evidence" hallucinates less than one forced to always answer

Quality is measured with **claim-level groundedness** (answers decomposed into atomic claims, each checked against retrieved evidence via entailment, validated against human annotation) and **citation accuracy** (does the cited source actually support the claim — cited-but-unsupported being the more insidious failure). Golden-set retrieval regression against pinned KG snapshots and trace assertions on agent behavior ("did every claim carry a source," "did it stay within budget") gate changes.

None of this eliminates hallucination — but it reduces it and, critically, makes remaining errors **detectable by a reviewing clinician**, because every claim carries a checkable source. For clinical safety, detectability matters as much as raw rate.

---

## ⚙️ Engineering & Serving

- **FastAPI** async API layer; long research runs are submitted as jobs (enqueue to Celery, return a job ID) with results streamed back over SSE as agents complete — the plan shows immediately, then findings land progressively
- **Celery queues separated by workload class** (ingestion, extraction, agent execution) so batch backlogs never starve interactive queries
- **Cost control** via model tiering (small models for routing and extraction, frontier models only for planning and synthesis), self-hosted specialist inference, document-hash caching of extraction, KG-version-aware result caching, and per-request token ceilings that bound the blast radius of a pathological agent loop
- **Latency by query class** — simple lookups answer in seconds on the fast path; a clinician researching treatment options gets a thorough multi-minute research run with visible progress, which beats a five-second shallow answer

---

## 🔒 Safety & Privacy by Design

- **Reviewable-recommendation posture** — the transparency layer isn't only for trust; showing sources, reasoning traces, and evidence quality lets a clinician independently evaluate the basis rather than take output on faith, which is what keeps the system in decision-support scope
- **Safety as a gate** — contraindications disqualify rather than deduct points; the system does not dose, does not diagnose, and does not tell patients to change treatment
- **PHI invariant** — the KG holds published knowledge only and stays patient-agnostic; patient context lives exclusively in request scope, minimized to structured clinical attributes, with sensitive inference kept in-infrastructure via self-hosting
- **Evidence-population visibility** — trial demographics are surfaced as metadata, so "evidence exists but not in this population" is distinguishable from "evidence is weak" rather than laundered into a confident ranking

---

## 🛠 Technical Stack
- Python, FastAPI, Celery.
- Apache Airflow, Neo4j, knowledge graph.
- GraphRAG, hybrid search, cross-encoder reranking.
- Deep‑research agents, Tx‑Gemma, BioMni, vLLM.
- LangChain, MCP.

---

## 🎯 Impact
This project advances medical AI by introducing:

- **Symbolic–neural hybrid intelligence** — a knowledge graph and dense retrieval fused over a shared canonical entity space
- **Agentic scientific reasoning** — orchestrated, validated, budget-bounded research agents
- **Provenance-driven transparency** — every inference tied to source documents, extracted relations, agent reasoning steps, and versioned KG entities
- **Patient-specific evidence synthesis** — with safety gates and abstention built in

It enables trustworthy, auditable, and clinically-aligned decision support for oncology, precision medicine, and scientific discovery.

---

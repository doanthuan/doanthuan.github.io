# Triangle Health

**Date:** 2025-2026

**URL**: [https://www.trianglehealth.com](https://www.trianglehealth.com)

**Project Overview:**
![Data-Pipelines](/assets/triangle/overview.png)
- Triangle Health transforms clinical corpora into a biomedical knowledge graph and layers GraphRAG with agentic research to deliver grounded, explainable answers for patients, clinicians, and researchers. Initial focus: oncology and neurology.

## System Overview
I designed and built an **Agentic Graph-Retrieved Reasoning Framework**, a next-generation AI system that integrates **biomedical knowledge graphs**, **hybrid semantic–graph retrieval**, and **multi-agent reasoning** to support transparent, evidence-grounded medical decision-making.

![System-Overview](/assets/triangle/system_overview.png)

This project addresses a core challenge in biomedical AI: combining structured scientific knowledge with deep reasoning in a way that is **auditable, explainable, and reliable**. The system empowers researchers and clinicians to explore **clinical trials, emerging treatments, mechanistic evidence**, and patient-specific reasoning paths.

---

## 🌐 Key Features

### 1. Biomedical Knowledge Graph Construction
![KnowledgeGraph](/assets/triangle/kg_construction.png)
Using sources such as ClinicalTrials.gov, PubMed, Semantic Scholar, OpenAlex, and PrimeKG, I built a **provenance-aware biomedical Knowledge Graph (KG)**.  
Key components include:

- LLM-driven summarization of scientific papers and clinical trials  
- Semantic chunking and contextual embedding  
- Entity extraction (diseases, drugs, genes, mutations, trials)  
- Relation extraction with canonical linking (UMLS, MeSH, OMIM, DrugBank)  
- Hybrid storage combining a **Graph Database** and **Vector Database**

**Outcome:** a unified and versioned biomedical KG enriched with high-quality insights.

---

### 2. Hybrid Graph + Semantic Retrieval
![KnowledgeGraphRetrieval](/assets/triangle/kg_retrieval.png)
To retrieve patient-specific evidence, I implemented a **dual retrieval engine**:

- **Semantic dense retrieval** using BioBERT / PubMedBERT embeddings  
- **Graph traversal–based symbolic retrieval**  
- **Fusion scoring** combining similarity, graph relevance, confidence, and centrality

This module supports:

- Clinical trial matching  
- Mutation–treatment alignment  
- Case-report similarity scoring  

**Outcome:** Precise, explainable retrieval with KG subgraphs, metadata, and reasoning traces.

---

### 3. Multi-Agent Scientific Research and Ranking
![AgentResearchRanking](/assets/triangle/agent_research_ranking.png)
This framework integrates a full **agentic reasoning pipeline** built around a **Medical Deep Research orchestrator agent** that plans the investigation, delegates to specialized subagents, and synthesizes their findings:

#### 🧭 Orchestrator: Medical Deep Research Agent
- Decomposes a clinical question into research subtasks  
- Routes subtasks to the appropriate subagent and coordinates multi-hop reasoning  
- Aggregates, reconciles, and synthesizes subagent outputs into grounded conclusions

#### 🔍 Deep Research Subagent
- ReAct agent with tools to access the **web** and the biomedical **knowledge base**  
- Iterative, multi-hop biomedical literature exploration  
- Evidence filtering, validation, and consistency checks

#### 🧠 Medical Subagent
- ReAct agent backed by a **self-hosted Tx-Gemma model** and **Therapeutic Data Commons (TDC) tools**  
- Clinical interpretation aligned with disease phenotype and mutation profile  
- Mechanistic evaluation of treatments with explicit citations

![MedicalResearchAgent](/assets/triangle/medical_research_agent.png)

#### 🏆 Tournament-Style Treatment Ranking
Each treatment is evaluated through pairwise matchups using criteria such as:

- Evidence strength and biological targeting  
- Safety and contraindications  
- Guideline and regulatory support  
- Patient-specific fit  
- Mechanistic plausibility  

**Outcome:** A ranked list of emerging and high-confidence treatments with transparent justification.

---

## 📈 Results & Contributions

### ✔ Improved Retrieval Accuracy  
Hybrid retrieval significantly outperforms traditional biomedical RAG systems.

### ✔ Enhanced Interpretability  
Provides:
- KG subgraphs  
- Provenance metadata  
- Agent reasoning logs  
- Citation-backed justifications  

### ✔ Reduced Hallucination  
Multi-agent debate + KG grounding lowers the rate of unverifiable claims.

### ✔ Modular and Auditable  
Every inference is tied to:
- Source documents  
- Extracted relations  
- Agent reasoning steps  
- Versioned entities in the KG  

---
## 🛠 Technical Stack
- Python, FastAPI, Celery.
- Apache Airflow, Neo4j, knowledge graph.
- GraphRAG, hybrid search, reranking.
- Deep‑research agents, Tx‑Gemma, BioMni, vLLM.
- Google ADK, MCP.


---

## 🎯 Impact
This project advances medical AI by introducing:

- **Symbolic–Neural hybrid intelligence**  
- **Agentic scientific reasoning**  
- **Provenance-driven transparency**  
- **Patient-specific evidence synthesis**

It enables trustworthy, auditable, and clinically-aligned decision support for oncology, precision medicine, and scientific discovery.

---
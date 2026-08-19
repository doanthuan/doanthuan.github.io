
## Triangle Health

**Date:** 2025

**URL**: [https://www.trianglehealth.com](https://www.trianglehealth.com)

**Overview:**
![Data-Pipelines](/assets/triangle/overview.png)
- Triangle Health transforms clinical corpora into a biomedical knowledge graph and layers GraphRAG with agentic research to deliver grounded, explainable answers for patients, clinicians, and researchers. Initial focus: oncology and neurology.
- Sources and data:
    - ClinicalTrials.gov, PubMed, FDA drug/label data, PrimeKG.
    - Optional patient EHRs and clinical notes (with consent).
- Features:
    - Q&A assistant for personal health conditions.
    - Comprehensive reports: promising therapies, standard‑of‑care summaries, off‑label options, and drug safety.
    - Biomedical deep‑research tasks: literature review, hypothesis exploration, fact‑checking, drug discovery and toxicity, genomics and molecular biology.

### Data pipelines
![Data-Pipelines](/assets/triangle/datapipeline.png)
- ETL pipelines orchestrated with Apache Airflow to ingest and normalize clinical trials, publications, and FDA drug/label data.
- Biomedical entity and relationship extraction (drugs, indications, outcomes, biomarkers) to construct a Neo4j knowledge graph.
- Parsing and transformation of patient EHRs and clinical notes into personal insights using smol‑docling.

### Knowledge Graph & Graph RAG
![Knowledge-Graph](/assets/triangle/kg.png)
- Graph indexing and embeddings to enable path‑aware querying, evidence tracing, and graph‑native retrieval.
- GraphRAG over Neo4j to retrieve context along clinically relevant paths (e.g., drug → indication → outcomes).
- Reranking to improve faithfulness and specificity for clinician‑facing answers.

### Deep Research Agents
![Tx-gemma](/assets/triangle/tx-gemma.png)
- Deep research agents orchestrated with Google ADK and MCP for multi‑step plans: query formulation, evidence gathering, synthesis, and citation.
- Use cases:
    - Fact‑checker that verifies claims in emerging reports.
    - Leverage Tx‑Gemma and BioMni for biomedical deep‑research tasks: drug discovery and toxicity; genomics and molecular biology; cell/gene therapy; biomanufacturing.
![Fact-Check](/assets/triangle/fact-check.png)

### Deployments & Observability
- AWS for infrastructure.
- FastAPI for the backend, SQS for queues, Celery for workers, Redis for caching.
- Self‑hosted GPUs with vLLM for Tx‑Gemma and BioMni inference.
- Tracing and observability with Langfuse and Datadog.

### Technologies
- Python, FastAPI, Celery.
- Apache Airflow, Neo4j, knowledge graph.
- GraphRAG, hybrid search, reranking.
- Deep‑research agents, Tx‑Gemma, BioMni, vLLM.
- Google ADK, MCP.

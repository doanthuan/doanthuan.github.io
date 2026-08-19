---
layout: page
title: Ensayo — Multi-Agent RAG and LLM-Powered Test Automation
permalink: /projects/ensayo/
---

<h3>
Project Information
</h3>

**Date:** 2023–2024

**Description:**
Ensayo is an AI platform that combines a multi-agent Retrieval-Augmented Generation (RAG) system with LLM-powered test automation for API-driven products. In customer-facing scenarios (e.g., banking), Ensayo delivers accurate, compliant, and context-aware assistance using advanced RAG techniques. For engineering workflows, it analyzes Swagger/OpenAPI specs to automatically generate structured test plans and BDD (Gherkin) scripts, reducing manual effort while improving coverage and quality.

![Multi-Agent](/assets/multiagent-2024-10-15.png)

### Key Capabilities
- **Multi-agent RAG for banking and enterprise**: specialized agents for retrieval, compliance, and internal API orchestration.
- **Advanced retrieval quality**: hybrid search, re-ranking, parent-document retrieval, semantic/structured chunking, and self-reflective (agentic) RAG.
- **Privacy and compliance by design**: document-level authorization, PII detection and anonymization.
- **LLM-based test automation**: from OpenAPI to test plan to BDD (Gherkin), with rule- and model-based validation.
- **Observability and evaluation**: DeepEval for RAG and model components, Langfuse for tracing.

---

### LLM-Powered Test Automation

![Test Generation](/assets/usecase-testgen.png)

#### Model Selection
- Criteria: benchmark performance, features (languages, context length, knowledge cutoff), license, privacy, and size (7B/13B).
- Selected open-source models: Llama2, Mistral, Llama3.

#### Evaluation Strategy
![Evaluation App](/assets/eval-app.png)
1. Compare outputs against GPT‑4 as a reference.
2. Rule-based scoring: template conformance, keyword checks, and Gherkin validation.

#### Data Pipeline
<img src="/assets/data-processing.png" width="500"/>
- Generate test plans and BDD from Swagger/OpenAPI with GPT‑4 and OSS LLMs.
- Validate using GPT‑4 and rule-based checks; human verification loop.
- Deploy pipeline using AWS Lambda for scalable automation.

#### Model Finetuning and Optimization
1. **LoRA finetuning** with PEFT + Accelerate on Hugging Face.
2. **AWS SageMaker** training and hosting.
   <img src="/assets/finetune.png" width="500"/>
3. **Extended Context Length** for long specifications.
   ![Extend Context](/assets/extend-context-length.png)
4. **RLHF / Preference Optimization** via DPOTrainer with a small human-preference dataset.
   ![Full Cycle LLM](/assets/full_cycle_llm.jpg)

---



#### RAG-Based Chatbot
- **RAG-Based Chatbot**: LLM augmented by retrieval over domain corpora (policies, procedures, documents, financial data) for trustworthy, grounded answers.
- **Multi-Agent Framework**: agents for document retrieval, compliance checking, and internal API integration collaborate to solve complex workflows.
- **Security & Compliance**: retrieval constrained to authorized sources; GDPR/AML-aware workflows.

<img src="/assets/basic_rag.webp" width="600"/>


#### RAG Techniques and Data Layer

![Advanced RAG](/assets/advanced_rag.jpeg)

##### Data Management
![Embedding Process](/assets/embedding-process.png)
- Document parsing with UnstructuredIO for PDFs and enterprise content.
- Continuous sync via LangChain Indexing API ([link](https://python.langchain.com/docs/how_to/indexing/)).
- Authorization using `Document` metadata for access-controlled retrieval.
- Data privacy with Microsoft Presidio for PII detection and anonymization.

##### Chunking Strategies
- Evaluate multiple fixed chunk sizes.
- Semantic chunking ([reference](https://pub.towardsai.net/advanced-rag-05-exploring-semantic-chunking-97c12af20a4d)).
- Document-specific splitting ([5 levels of text splitting](https://github.com/FullStackRetrieval-com/RetrievalTutorials/blob/main/tutorials/LevelsOfTextSplitting/5_Levels_Of_Text_Splitting.ipynb)).

#### Parent Document Retriever
<img src="/assets/parent_doc.png" width="400"/>
- LangChain ParentDocumentRetriever ([docs](https://python.langchain.com/docs/modules/data_connection/retrievers/parent_document_retriever/))
- RAGStack example ([docs](https://docs.datastax.com/en/ragstack/docs/examples/advanced-rag.html#create-a-parentdocumentretriever-rag-chain))
- Overview article ([link](https://medium.com/ai-insights-cobet/rag-and-parent-document-retrievers-making-sense-of-complex-contexts-with-code-5bd5c3474a8a))

#### Hybrid Search
<img src="/assets/hybrid_search.png" width="300"/>
- Combine BM25 and semantic search in a single query for higher relevance.

#### Re-ranking
![Reranker](/assets/reranker.png)
- Apply Cohere reranker via LangChain; support for both reranker models and LLM-based re-ranking.

#### Self-Reflective RAG with LangGraph
![Self-RAG](/assets/self-rag.png)
- Agentic RAG refinement ([article](https://blog.langchain.dev/agentic-rag-with-langgraph/)).

#### Multi-Modal Embedding
![Multimodal](/assets/multimodal.png)
- Extend retrieval to tables and images.
- Google Gemini multimodal embeddings; Multi-Vector Retriever in LangChain ([guide](https://blog.langchain.dev/semi-structured-multi-modal-rag/)).

#### Context Understanding with RAPTOR
![RAPTOR](/assets/raptor.webp)
- Tree-based retrieval with hierarchical summarization for global understanding.
- Overview ([link](https://ai.gopubby.com/advanced-rag-12-enhancing-global-understanding-b13dc9a8db39)).


### Evaluation
- Use DeepEval to assess retrieval, generation, and end-to-end RAG performance.
- Retrievers and embedding model tested with task-specific retrieval evaluation.

### Technologies
- Llama2, Llama3, Mistral; Transformers
- LangChain, LangGraph; FastAPI
- AWS SageMaker; Docker, Kubernetes, Helm
- pgvector; Cohere Reranker
- Semantic caching; Advanced RAG; Multi-Agent design
- Deepeval; Langfuse; Streaming
- Ollama, vLLM

### Deployment
- AWS SageMaker for model serving
- FastAPI backend services
- Containerized with Docker; orchestrated via Kubernetes and Helm
- Langfuse for observability and tracing

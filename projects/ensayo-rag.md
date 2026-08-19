---
layout: page
title: 
permalink: /projects/ensayo-rag/
---

<h3>
Project Information
</h3>

**Date:** 2024

**Description:**
The project focuses on developing a sophisticated, multi-agent Retrieval-Augmented Generation (RAG)-based chatbot system designed for the banking domain. This solution leverages Large Language Models (LLMs) and retrieval mechanisms to deliver highly accurate, contextually aware, and domain-specific interactions for banking customers and internal employees.

![Image](/assets/multiagent-2024-10-15.png)

#### Key Components:

1. **RAG-Based Chatbot:**
The chatbot uses a Retrieval-Augmented Generation (RAG) model, where an LLM is enhanced by retrieving banking-specific information like documents, policies, and financial data. This ensures responses are fact-based and trustworthy, improving the chatbot’s reliability.
2. **Multi-Agent Framework:**
The chatbot operates in a multi-agent architecture, where specialized agents handle tasks such as document retrieval, compliance checking, and internal APIs. These agents collaborate to manage complex banking workflows efficiently.
3. **Enhanced Security and Compliance:**
Security and compliance are prioritized, and ensuring adherence to regulations like GDPR and AML. Retrieval systems are restricted to authorized information, safeguarding data and maintaining regulatory compliance.

**Technologies:**
- Llama3
- LangChain, LangGraph
- Transformers
- AWS SageMaker
- pgvector
- Semanic Caching
- Advanced RAG techniques
- Multi-Agent Design
- FastAPI
- Deepeval
- Langfuse
- Streaming
- Ollama, vllm

In this project, the main component is RAG pipeline. We have applied serveral techniques for RAG improvement.

### Basic RAG Flow

<img src="/assets/basic_rag.webp" width="600"/>



### RAG Techniques
![Image](/assets/advanced_rag.jpeg)


#### Data Management
![Image](/assets/embedding-process.png)

Document Parsing:

- User uploaded documents and confluence pages (in exported PDF format) would be uploaded into system. 
- Use **UnstructuredIO** package to parse and extraction information.


Keep in sync documents:
 - Use LangChain indexing API to keep in sync documents
 - [https://python.langchain.com/docs/how_to/indexing/](https://python.langchain.com/docs/how_to/indexing/)

Authorization:
 - Use metadata in Document object to filter retrival results by document access leval

Data Privacy:
 - Using [Microsoft Presidio](https://microsoft.github.io/presidio/) to detect and anonymization Personal Identifiable Information

#### Chunking Strategies
- Test with multiple fixed chunk size
- Applied [semantic chunking](https://pub.towardsai.net/advanced-rag-05-exploring-semantic-chunking-97c12af20a4d)
- Applied document specific splitting (ref: [5 levels of text splitting](https://github.com/FullStackRetrieval-com/RetrievalTutorials/blob/main/tutorials/LevelsOfTextSplitting/5_Levels_Of_Text_Splitting.ipynb) )

#### Parent Document Retriever
<img src="/assets/parent_doc.png" width="400"/>

https://python.langchain.com/docs/modules/data_connection/retrievers/parent_document_retriever/

https://docs.datastax.com/en/ragstack/docs/examples/advanced-rag.html#create-a-parentdocumentretriever-rag-chain

https://medium.com/ai-insights-cobet/rag-and-parent-document-retrievers-making-sense-of-complex-contexts-with-code-5bd5c3474a8a


#### Hybrid Search
<img src="/assets/hybrid_search.png" width="300"/>
- Hybrid search combines semantic and keyword search in one query for more relevant results.
- Combining BM25 and Semantic Search for better results with Langchain


#### Re-ranking
![Image](/assets/reranker.png)

The re-ranking methods described in this article can be mainly divided into the following two types:

- Re-ranking models
- LLM

We applied Cohere reranker from Langchain

#### Self-Reflective RAG with LangGraph
![Image](/assets/self-rag.png)
https://blog.langchain.dev/agentic-rag-with-langgraph/



#### Multi Modal Embedding
![Image](/assets/multimodal.png)

- We want to apply RAG to not only text, but also tables, images
- Google Gemini for multimodal embedding
- Multi-Vector Retriever with LangChain
- [https://blog.langchain.dev/semi-structured-multi-modal-rag/](https://blog.langchain.dev/semi-structured-multi-modal-rag/)


#### Context Understanding with RAPTOR
![Image](/assets/raptor.webp)
- Many important real-world tasks, including scientific literature review, legal case briefing, and medical diagnosis, require knowledge understanding across chunks or documents.
- RAPTOR is a novel tree-based retrieval system designed for recursively embedding, clustering, and summarizing text segments. It constructs a tree from the bottom up, offering varying levels of summarization.
- [More Details](https://ai.gopubby.com/advanced-rag-12-enhancing-global-understanding-b13dc9a8db39)

### Evaluation
- Use DeepEval to evaluate each components and end-to-end RAG pipeline evaluation.
- Retrieval evaluation for embedding model

### Deployment
- AWS SageMaker for LLM
- FastAPI for backend
- Docker, Kubernetes, Helm
- Langfuse for tracing


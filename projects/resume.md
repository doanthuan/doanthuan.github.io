# Thuan Doan

(+84)902727231 | doanvuthuan@gmail.com | [LinkedIn](https://www.linkedin.com/in/doan-thuan-16ab1a61/) | [Profile](https://doanthuan.github.io)

---

## Summary
AI Engineer with 10+ years of software engineering experience, including 5 years in AI, building machine learning, LLM, retrieval systems, and agentic workflows for production. Currently interested in physical AI and world models.

---

## Experience

### Spartan — Senior AI Engineer
*Nov 2024 – Present | Ho Chi Minh, Vietnam*

- Supported solving AI problems across projects within the company.
- Conducted technical sharing, mentored junior engineers, and supported AI hiring and interviews.

**TriangleHealth** | *Apache Airflow, TxGemma, GraphRAG, Neo4j, Google ADK, MCP, vLLM*
- Built scalable Airflow pipelines processing 10M+ biomedical entities to construct a Neo4j knowledge graph powering medical research workflows.
- Designed graph enhanced retrieval (hybrid) + reranking (TourRank) improving answer accuracy by +15-20% on internal medical retrieval benchmarks.
- Developed multi-agent medical research system (Google ADK + TxGemma/BioMNI) to minimize hallucinations and deliver evidence-grounded, verifiable outputs. 
- Optimized model deployment using vLLM. Implemented tracing and observability with Langfuse and Datadog.

**LayerProof** | *Kotlin, Micronaut, PostgreSQL, SQS, Gemini, OpenAI, Claude, Python, PyTorch, AWS ECS Fargate*
- Designed an event-sourced durable workflow engine with a strict determinism contract and pluggable queue backends, powering fully-replayable multi-agent pipelines that generate social-media campaigns, slide decks, and mindmaps.
- Built a Python image-processing worker on ECS Fargate running custom GPU models (CyberAgent LayerD for slide layer decomposition, BiRefNet matting, U²-Net background removal) alongside OCR inpainting, raster→SVG, and headless-Chromium rendering, all co-scheduled on a weighted-semaphore capacity pool.

**QAMobileAgent** | *GUI-Owl-7B, Appium, uiautomator2, Hugging Face Transformers, FastAPI, LoRA*
- Designed a hybrid GUI testing agent for banking apps combining a multimodal vision-language model (GUI-Owl-7B) with deterministic Appium execution and rule-based safety fallbacks.
- Built structured UI state representation using screenshots, accessibility trees, and OCR. Designed fine-tuning roadmap with SFT, preference optimization for domain adaptation.

### HCLTech — AI Engineer/Researcher
*Jun 2023 – Oct 2024 | Ho Chi Minh, Vietnam*

**FinGPT** | *Azure OpenAI, LangChain, LangGraph, FastAPI, Langfuse, Promptfoo, DeepEval*
- Built a multi-agent conversational AI system for retail banking, integrating planner, retrieval, banking API, and compliance agents, supporting 1K+ daily user queries.
- Designed and implemented a production-grade RAG pipeline (hybrid search + re-ranking), improving response accuracy by ~20–25% on internal evaluation benchmarks.
- Developed compliance and safety layer (PII detection/redaction, audit logging, role-based retrieval), reducing risk of sensitive data exposure and ensuring 100% audit traceability.

**EnsayoAI** | *Llama3, LangGraph, RAG, Langfuse, Promptfoo, DeepEval, pgvector, FastAPI, AWS SageMaker*
- Developed a generative AI testing platform leveraging Llama 3 and Mistral, automating test case generation and reducing manual QA effort by ~40–60%.
- Applied domain-adaptive fine-tuning (LoRA, RLHF), improving relevance and correctness of generated test cases by ~15–20% based on evaluation metrics.
- Designed and deployed LLM evaluation pipelines (DeepEval, Promptfoo) to benchmark model performance and ensure reliability across test scenarios.

### Infinigru — Machine Learning Engineer
*Aug 2018 – Apr 2023 | Korea/Vietnam*

**GruBot** | *Python, wav2vec, Kaldi, RASA, BERT, LSTM, Kubernetes, MLflow, Prometheus, Grafana*
- Developed and fine-tuned Korean and Vietnamese speech recognition models using Kaldi and wav2vec.
- Built RASA-based conversational AI chatbot with NLU pipeline and dialogue management, handling multi-turn conversations with seamless human agent hand-off.
- Optimized BERT-based models for Named Entity Recognition (NER) and Sentiment Analysis.

**ViettelFDS** | *Python, Java, Spring Boot, Apache Kafka, Apache Flink, PySpark, XGBoost, LightGBM, SHAP, VertexAI*
- Developed high-performance real-time fraud detection system combining rule-based systems with ML models (Random Forest, XGBoost, LightGBM) to reduce false positives and detect evolving fraud patterns.
- Built real-time data pipeline using Apache Kafka and Flink for processing payment transactions with threshold-based decision-making.

**AirPatrol** | *U-Net, Mask R-CNN, Xception, TensorFlow Lite, ONNX, TensorRT, Jetson Nano*
- Developed and deployed deep learning models (U-Net, Mask R-CNN, Xception) for crack detection and wildfire classification, optimized with TensorFlow Lite for real-time inference on NVIDIA Jetson Nano.

### Earlier Experience
*Global Outsource Solution (Technical Lead), Global CyberSoft / Forix (Software Engineer) | 2008–2018*

- Led development of scalable web and enterprise systems across e-commerce, CMS, and data platforms.
- Built backend services and system architectures; established CI/CD pipelines and engineering best practices.

---

## Education

**University of Information Technology** — Master's Degree in Computer Science
*2020 – 2023 | Ho Chi Minh, Vietnam*
- Research assistant at MMLab. Topics: NLP, LLMs, VLM, SLAM, 3D vision, deep generative models.

**Japan Advanced Institute of Science and Technology (JAIST)** — Research Assistant
*Jan–Mar 2020 | Ishikawa, Japan*
- Supervision of Prof. Kazuhiro Ogata. Research Topic: Theorem proving and state machine with CafeOBJ.

**Kookmin University** — Research Assistant
*2011 – 2012 | Seoul, Korea*
- Supervision of Prof. Young Kim. Research Topic: Security-Enhanced Linux (SELinux) for web servers.

**University of Science** — Bachelor's Degree in Computer Science
*2004 – 2008 | Ho Chi Minh, Vietnam*

---

### 📦 **Technical Skills**

- **Programming:** Claude Code, Python, JavaScript, Java, PHP
- **AI Frameworks & Tools:** TensorFlow, PyTorch, Scikit-learn, Hugging Face Transformers, Pandas, Numpy, Matplotlib, OpenCV, NLTK, MLflow, Datadog, Sentry, Prometheus, Grafana, SageMaker, VertexAI
- **LLM Tools:** LangChain, Google ADK, LlamaIndex, Langfuse, LiteLLM, vLLM, Ollama, Unsloth, DeepEval
- **App Frameworks:** FastAPI, Flask, Spring, Laravel, React, NodeJS, VueJS
- **Data & Cloud:** MySQL, PostgreSQL, MongoDB, Redis, Neo4j, Airflow, Kafka, Flink, PySpark, ChromaDB, AWS, GCP

---

## Hobby Projects

**ChefBuddy** | *NVIDIA GR00T N1.5, LeRobot, SO-100/SO-101, LoRA, TensorRT*
- Built a kitchen robot using NVIDIA's GR00T N1.5 vision-language-action model with SO-100 arm for language-conditioned pick-and-place tasks such as sandwich assembly.
- Implemented full pipeline from teleoperation data recording to LoRA fine-tuning and real-time deployment.

**HomeExplorer** | *ROS, GMapping, Hector SLAM, YOLOv5, TensorRT, Jetson Nano*
- Built an edge-based SLAM and semantic mapping system combining 2D LiDAR localization with camera-based object detection on NVIDIA Jetson Nano.

---

## Publication

- Efficient Finetuning Large Language Models For Vietnamese Chatbot (first author) — MAPR 2023

---


## Additional

- TensorFlow Developer Certificate
- Coursera Specializations (ML, DL, NLP, Data Science)
- IELTS 6.5






---
layout: page
title: Backbase — Conversational Banking with Multi‑Agent LLM Chatbot
permalink: /projects/backbase/
---

<h3>
Project Information
</h3>

**Date:** 2024

**Keywords:** Conversational Banking, LLM, Chatbot, Multi‑Agents

**Overview:**
Backbase is a conversational banking experience powered by a multi‑agent LLM system. It enables secure, compliant, and context‑aware customer interactions across retail banking journeys (balances, transfers, card services, onboarding) while integrating with core and third‑party APIs. The system uses retrieval‑augmented workflows, tool‑use, and orchestration to deliver accurate, auditable answers and automate routine tasks.

**Features:**
![Features](/assets/backbase/plan.png)

**Architecture:**
![Architecture](/assets/backbase/overal.png)

### Capabilities
- **Banking dialog orchestration:** intent detection, slot filling, disambiguation, and follow‑ups across multi‑turn flows.
- **Tool‑augmented agents:** connectors for account info, payments, and card mgmt; safe function‑calling with guardrails.
- **RAG for policy grounding:** retrieve product terms, fees, eligibility, and procedures for trustworthy responses.
- **Compliance & privacy:** PII detection/redaction, audit logs, and role‑based retrieval over authorized sources.
- **Observability:** tracing, evaluation, and analytics for model and agent behaviors.

### Architecture
- **Multi‑Agent Controller:** planner, retrieval agent, banking‑API agent, and compliance agent collaborating via messages.
- **RAG Layer:** hybrid search + re‑ranking + parent‑document retrieval for high‑precision grounding.
- **Integration Layer:** FastAPI services exposing tools to agents; adapters to core banking and payment rails.
- **Evaluation & Tracing:** DeepEval for quality checks; Langfuse for traces and outcomes.

### Technologies
- Azure OpenAI API
- LangChain, LangGraph, FastAPI
- Langfuse, promptfoo, Deepeval
- Nemo Guardrails, Sentry

### My Contributions
- Designed the multi‑agent conversation architecture and tool interfaces for banking tasks.
- Implemented RAG pipelines with hybrid retrieval and re‑ranking for policy grounding.
- Built FastAPI adapters to core banking APIs with secure function‑calling.
- Established observability (tracing/eval) and privacy guardrails for production readiness.

### Outcomes
- Faster self‑service resolution and reduced contact‑center load.
- Consistent, policy‑grounded answers with auditability.
- Modular agent framework enabling rapid expansion to new banking use cases.

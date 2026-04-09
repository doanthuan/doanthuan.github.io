# 🚀 LayerProof

### Multi-Agent AI System for Generating Professional Presentations

**Date:** 2026

**URL**: [https://layerproof.app/](https://layerproof.app/)

---

## ✨ Overview

**LayerProff** is a multi-agent AI system that transforms documents, prompts, or notes into **professional presentation slides (PPTX)**.

Unlike traditional tools, it uses **specialized AI agents** for planning, writing, designing, and refining slides — mimicking how humans actually create presentations.

---


## 🧩 Problem

Creating high-quality slides is still:

* ⏱️ Time-consuming
* 🧠 Cognitively heavy (summarization + storytelling + design)
* 🎨 Design-intensive

Existing AI tools:

* produce verbose content
* lack narrative flow
* ignore layout & visuals
* cannot adapt to domain-specific style

👉 Slide generation is **not just summarization** — it requires:

* structured reasoning
* multimodal generation
* iterative refinement

---

## 💡 Solution

We decompose slide generation into a **multi-agent system**:

* 🧠 Planner → defines structure
* ✍️ Writer → generates concise slides
* 🔍 Retriever → grounds content (RAG)
* 🎨 Designer → adds visuals
* 📐 Layout → organizes slides
* 🧪 Critic → improves quality

---

## 🏗️ Architecture

### 🔷 System Flow

```mermaid
flowchart TD
    A[Input: PDF / Prompt / Doc]
    B[Preprocessing + RAG]
    C[Planner Agent]
    D[Slide Writer]
    E[Visual Designer]
    F[Layout Agent]
    G[Critic Agent]
    H[Renderer: PPTX]

    A --> B --> C --> D --> E --> F --> G --> H
```

---

### 🤖 Multi-Agent Design

| Agent        | Responsibility                      |
| ------------ | ----------------------------------- |
| 🧠 Planner   | Defines slide structure & narrative |
| 🔍 Retriever | Extracts relevant content (RAG)     |
| ✍️ Writer    | Generates titles & bullet points    |
| 🎨 Designer  | Chooses charts / images / diagrams  |
| 📐 Layout    | Selects slide templates             |
| 🧪 Critic    | Improves clarity & coherence        |

---

## 🎨 Visual Generation

* Text reasoning: **OpenAI API**
* Image generation: **Google Imagen**

### Example Generated Visuals

![Image](/assets/layerproff/sample1.webp)
![Image](/assets/layerproff/sample2.webp)
![Image](/assets/layerproff/sample3.webp)


---

## 🛠️ Tech Stack

### Core AI

* LLM: OpenAI API
* Image: Google Imagen

### Retrieval (RAG)

* FAISS / Qdrant
* BGE / E5 embeddings
* Cross-encoder reranker

### Orchestration

* LangGraph (multi-agent workflow)
* Custom agent loop

### Rendering

* python-pptx
* Mermaid / Graphviz
* matplotlib / plotly

### Backend

* FastAPI
* Async pipeline

---

## ⚙️ Implementation

### 🔹 Phase 1 — Multi-Agent System

* Multi-agent orchestration
* RAG grounding
* Visual generation (Imagen)
* Iterative refinement loop
* PPTX export

---


### 🔹 Phase 2 — User-Preference Image Optimization (In Progress)

**Goal:** Improve slide visuals by optimizing the image generation process to align with **user preferences** — producing images that match the intended style, tone, and domain.

---

#### 🧠 Approach

* **Preference-driven prompt engineering** — dynamically construct image prompts based on user-specified style, color palette, and domain context
* **Iterative feedback loop** — the Critic agent evaluates generated images against user preferences and triggers re-generation with refined prompts
* **Style templates** — curated prompt templates for common domains (tech, business, education, healthcare) that users can select or customize
* **Image selection pipeline** — generate multiple candidates per slide and rank them using CLIP-based similarity scoring against user-provided reference images or style descriptions

---

#### 🎯 What Gets Optimized

* prompt construction aligned to user style preferences
* consistency across slides via shared style context
* domain-specific visual tone (AI, fintech, education, etc.)
* iterative refinement based on user feedback

---


### 🔬 Research

Inspired by recent work :

* AutoPresent (programmatic slides)
* PPTAgent (edit-based generation)
* SlideAudit (design critique)

---

## 🧪 Evaluation

* Content relevance
* Slide conciseness
* Narrative coherence
* Design quality
* Human preference ranking

---


## ⭐ Key Highlights

* 🧠 Multi-agent system
* 📄 Document → Slides
* 🎨 Multimodal generation
* 🔁 Self-refinement loop
* 🧪 User-preference image optimization


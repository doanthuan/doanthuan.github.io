---
layout: page
title: 
permalink: /projects/ensayo-test/
---

<h3>
Project Information
</h3>
**Date:** 2023

**Description:**
- This project focuses on leveraging open-source Large Language Models (LLMs) to enhance automated software testing workflows.
- The system utilizes advanced capabilities of LLMs to analyze Swagger OpenAPI specs and automatically generate a structured test plan, ensuring that key testing scenarios are identified and covered. From this generated test plan, the system further automates the creation of BDD scripts in Gherkin format, aligning with the principles of test-driven development and making it easier for teams to follow human-readable, scenario-based testing methodologies.
- The project aims to streamline the testing process, reduce manual efforts, and improve the overall quality of software testing in development pipelines. This approach enhances testing automation, making it adaptable to various API-driven systems


![Image](/assets/usecase-testgen.png)

**Technologies:**
- Llama3, Mistral
- LangChain
- Transformers
- AWS SageMaker
- LoRA
- RLHF
- FastAPI
- Langfuse
- Streaming
- Ollama, vllm

In this project, the main component is RAG pipeline. We have applied serveral techniques for RAG improvement.

### Model Selection
Some criteria to select models:
- Model performance on many benchmarks
- Features: language, context length, knowledge cutoff date
- Model license,
- Data privacy.
- Model size: 7B and 13B.

We selected top open source LLMs to use in our projects: LLama2, Mistral, Llama3.


### Model Evaluation

![Image](/assets/eval-app.png)

1. Using OpenAI GPT4
    - Compare result with output of GPT4
2. Rule Score
    - Implement some rule score with some criteria:
        - Follow test plan template stuctures 
        - Keyword matching
        - Gherkin validation tool

### Data Pipeline

<img src="/assets/data-processing.png" width="500"/>

- Use OpenAI GPT4 to generate test plan and BDD from swagger specs
- Validate with GPT4, rule-based checking
- Verified by human
- Deploy data pipeline using AWS Lambda function

### Model Finetuning
1. Use LoRA techniques to finetune

    In our case, we are going to leverage Hugging Face Transformers, Accelerate, and PEFT.

2. AWS SageMaker

    <img src="/assets/finetune.png" width="500"/>


3. Extend Context Length
![Image](/assets/extend-context-length.png)

4. RLHF

We build simple dataset with human preferences and use DPOTrainer from HuggingFace for the finetuning

![Image](/assets/full_cycle_llm.jpg)


### Deployment
- AWS SageMaker for LLM
- FastAPI for backend
- Docker, Kubernetes, Helm
- Langfuse for tracing


# MobileTestAgent

## Date: 2025

## Overview

I designed a **GUI-based mobile testing agent** for banking applications that combines a **multimodal GUI model**, a **structured mobile UI state**, and a **deterministic execution layer** to automate end-to-end test flows on Android apps.

The project is organized into **two phases**:

* **Phase 1** focuses on building a practical, self-hosted testing agent using a **GUI-specialized vision-language model** and a hybrid execution architecture.
* **Phase 2** focuses on improving reliability and domain fit through **fine-tuning**, preference optimization, and later reinforcement learning.

The system is inspired by recent progress in GUI agents:

* **DroidBot-GPT** showed that GUI automation can be framed as a decision-making loop over app state and action history. 
* **UI-R1** showed that reinforcement learning can improve GUI action prediction and efficiency. 
* **Mobile-Agent-v3 / GUI-Owl** demonstrated that GUI-specialized multimodal agents can achieve strong performance across Android and desktop benchmarks. 


![System-Overview](/assets/uiagent/overview.png)

---

## Why this project matters

Banking applications are among the most challenging mobile apps to test because they involve:

* multi-step transactional workflows
* OTP and biometric verification
* session timeout and security prompts
* high-risk negative scenarios
* strict requirements for reproducibility and auditability

Traditional automation frameworks are strong for scripted regression, but they are often brittle when:

* UI layouts change
* test flows branch dynamically
* error handling and recovery are required
* exploratory or goal-driven automation is needed

This project explores a more adaptive approach:
using a **GUI agent** that can understand screens, reason about the next best action, and execute test steps under controlled constraints.

---

# Phase 1 — Build the GUI Testing Agent

## Goal

Build a **practical mobile GUI agent** for banking app testing that is:

* easy to implement
* self-hostable
* easy to customize
* compatible with existing mobile test infrastructure
* traceable and debuggable for QA use

---

## Core idea

Instead of relying on a pure end-to-end “black box” agent, I designed a **hybrid architecture**:

![overview](/assets/uiagent/uiagent.png)

### 1. Deterministic execution layer

Handles:

* tap
* type
* swipe
* back
* wait
* screen assertions
* evidence capture

Built on top of:

* **Appium** or **uiautomator2**
* Android emulator or real device farm
* screenshot capture
* accessibility tree / UI XML dump

### 2. Multimodal GUI policy

Uses a **GUI-specialized VLM** to decide the next best action based on:

* screenshot
* structured UI state
* recent action history
* current test goal
* candidate actions

### 3. Rule-based fallback layer

Handles fragile but common banking/system flows:

* permission popups
* biometric prompts
* session timeout dialogs
* OS-level alerts
* OTP verification interruptions

This hybrid design was chosen because early systems like **DroidBot-GPT** showed the power of LLM-guided GUI automation, but also highlighted the limits of relying only on prompt-based decision-making without stronger state representation and control. 

---

## Model choice

For the first phase, I selected **GUI-Owl-7B** as the main policy model.

### Why GUI-Owl-7B

* it is already specialized for **GUI automation**
* it is small enough to be realistically **self-hosted and fine-tuned**
* it provides a strong balance between **performance and implementation cost**
* the Mobile-Agent-v3 report positions GUI-Owl as a strong open-source foundation for mobile and desktop GUI tasks. 

### Why not start with a larger model

A much larger GUI model may improve benchmark numbers, but it also increases:

* hosting cost
* inference latency
* fine-tuning complexity
* iteration time

For an engineering-focused testing system, I prioritized:

* fast iteration
* controllable behavior
* lower operational cost
* easier domain adaptation

---

## Architecture

### Input to the agent

At each step, the system constructs a compact state containing:

* current screenshot
* current screen type
* visible UI elements
* recent action history
* current testing objective
* candidate action space

### Output from the agent

The agent returns a structured action, for example:

```json
{
  "thought": "The amount field must be filled before continuing.",
  "action": "type",
  "target": {
    "element_id": 3,
    "value": "100000000"
  },
  "expected_outcome": "Amount field is populated."
}
```

This makes the system easier to:

* debug
* validate
* log
* replay
* improve later with supervised fine-tuning

---

## Screen ontology

To make the agent more stable for banking use cases, I introduced a lightweight **screen ontology**, such as:

* login
* dashboard
* transfer_form
* beneficiary_select
* otp
* transfer_success
* transfer_error
* session_expired
* biometric_prompt

This improves:

* prompting quality
* action selection
* assertion design
* recovery policies

---

## Testing scenarios covered

The initial project scope focuses on core banking flows such as:

* valid login
* invalid password
* invalid OTP
* session timeout and re-login
* balance inquiry
* transfer to saved beneficiary
* insufficient balance validation
* add beneficiary
* freeze / unfreeze card
* transaction history search
* statement download
* biometric prompt handling
* permission popup handling

---

## Key engineering decisions

### Structured state over screenshot-only

I did not rely on image-only reasoning in Phase 1.
Instead, I used:

* screenshot
* accessibility tree
* OCR if needed
* normalized visible elements
* recent history

This makes the agent easier to inspect and later fine-tune.

### Fixed action schema

The agent is constrained to a controlled action space rather than arbitrary free-form outputs.

### Agent as decision-maker, not raw executor

The model selects the action, but the actual interaction is executed by a deterministic automation layer.

This separation improves:

* reproducibility
* safety
* observability
* enterprise readiness

---

## Tech stack

* **GUI model**: GUI-Owl-7B
* **Automation layer**: Appium / uiautomator2
* **Runtime**: Python
* **Serving**: Hugging Face Transformers / FastAPI
* **Storage**: screenshots, UI dumps, action traces
* **Optional infra**: Postgres, object storage, tracing dashboard

---

## Phase 1 outcome

Phase 1 delivers a **working banking app GUI test agent** that can:

* navigate mobile banking workflows
* perform structured multi-step actions
* handle some dynamic UI changes better than rigid test scripts
* produce detailed action traces for analysis
* serve as a foundation for data collection and later model adaptation

---

# Phase 2 — Enhance with Fine-Tuning

## Goal

Improve the agent’s reliability on a specific banking app through **domain adaptation**, starting with **supervised fine-tuning** and then optionally extending to **preference tuning** and **reinforcement learning**.

---

## Why fine-tuning is needed

A general GUI model may still struggle with banking-specific issues such as:

* custom UI components
* icon-heavy controls
* security flows
* proprietary navigation patterns
* domain-specific error messages
* test-specific assertions

Recent GUI-agent research suggests that general models become much more useful when adapted to task-specific trajectories and action patterns. In particular, **UI-R1** shows that GUI agent behavior can be improved through learning-based optimization beyond pure prompting. 

---

## Fine-tuning strategy

### Stage 1 — Supervised Fine-Tuning (SFT)

I would first collect step-level trajectories from:

* manual QA runs
* existing Appium scripts
* human demonstrations
* failed agent runs corrected by testers

Each step becomes a training sample:

```text
(task, state, history, visible elements) -> best next action
```

This is the highest-ROI training stage.

### Stage 2 — Recovery fine-tuning

Add examples specifically for:

* wrong screen recovery
* popup handling
* session expiry
* OTP retries
* keyboard obstruction
* security interruptions

This improves practical robustness.

### Stage 3 — Preference tuning

For the same screen state, compare:

* good action
* plausible but wrong action
* inefficient action

Then train the model to prefer better decisions.

### Stage 4 — Reinforcement learning (optional)

Once the policy is already stable, use RL to improve:

* efficiency
* recovery quality
* loop avoidance
* robustness under dynamic UI variations

This follows the general idea explored in **UI-R1**, where RL is used to improve action prediction and behavior efficiency after a stronger base policy already exists. 

---

## Training data design

I would maintain three datasets:

### 1. SFT dataset

For next-action prediction:

* current task
* screen type
* screenshot
* UI tree / OCR text
* recent history
* correct next action

### 2. Recovery dataset

For failure correction:

* failed state
* best recovery action
* optional explanation

### 3. Preference dataset

For action ranking:

* current state
* preferred action
* rejected action

This creates a practical improvement loop without needing full RL from the start.

---

## Fine-tuning method

The preferred approach is **LoRA fine-tuning** because it is:

* cheaper than full fine-tuning
* easier to iterate
* easier to rollback
* practical for app-specific specialization

The model can be adapted without retraining the entire visual-language stack.

---

## Evaluation plan

To measure improvements after fine-tuning, I would evaluate at four levels:

### Step-level

* exact action match
* correct element selection
* valid JSON action format

### Screen-level

* screen type recognition
* popup classification
* assertion accuracy

### Episode-level

* task success rate
* average steps to complete
* timeout rate
* recovery success rate

### Banking-risk level

* unsafe action rate
* missed error popup rate
* missed evidence capture
* wrong terminal state behavior

---

## Phase 2 outcome

Phase 2 transforms the system from a general GUI agent into a **banking-specialized testing agent** that is better at:

* domain-specific navigation
* security and recovery flows
* negative test scenarios
* assertion-driven testing
* efficient execution on the target app

---

# My Role

* Designed the overall **agent architecture**
* Selected the **GUI-specialized VLM strategy**
* Defined the **hybrid policy + deterministic execution pattern**
* Designed the **fine-tuning roadmap**
* Structured the data strategy for:

  * SFT
  * recovery learning
  * preference optimization
  * future RL
* Positioned the solution for a **real enterprise testing context**

---

# Key Takeaways

This project reflects a practical belief:

> The best GUI testing agent for enterprise mobile apps is not a pure black-box model.
> It is a **hybrid system**: multimodal policy for decision-making, deterministic execution for control, and iterative fine-tuning for domain specialization.

Phase 1 establishes the foundation.
Phase 2 makes it production-worthy.

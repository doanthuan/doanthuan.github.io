
# 🏦 GUI Mobile Agent for Banking App

## Overview

Modern mobile banking applications are **highly dynamic, security-critical, and expensive to test**.  
Traditional automation frameworks (e.g., Appium) rely heavily on **hard-coded scripts**, making them brittle and difficult to scale.

This project explores a new paradigm:

> **AI-native GUI testing agent** that understands screens, reasons about actions, and executes workflows safely.

---

## 🚧 Challenge

Banking apps introduce challenges:

- Complex multi-step workflows (login → OTP → transfer)
- Security layers (biometric, OTP, session timeout)
- Dynamic UI (popups, async loading)
- High reliability requirements

👉 Result: fragile, expensive automation systems.

---

## 💡 Solution

A **hybrid GUI agent system** combining:

- 🧠 **Multimodal AI (GUI-Owl-7B)** for decision-making  
- ⚙️ **Deterministic execution (Appium/uiautomator2)**  
- 🧩 **Structured UI state (image + UI tree + history)**  
- 🛡️ **Rule-based safety layer**

> The model decides **what to do**, the system controls **how to execute safely**.

---

## 🏗️ Architecture

### Agent Loop

```
State → Model → Action → Execution → Assertion → Next State
```

### Components

- Device Layer (Android + Appium)
- State Builder (UI + OCR + history)
- VLM Policy (GUI-Owl-7B)
- Rule Engine (OTP, popup handling)
- Assertion Engine (validation)

---

## Phase 1 — Build Agent

### Goal
Build a self-hosted, controllable GUI testing agent.

### Key Decisions
- Hybrid system (AI + deterministic execution)
- Structured state (not screenshot-only)
- Fixed action schema

### Coverage
- login, OTP, transfer, errors, session timeout
- beneficiary management
- transaction history

---

## Phase 2 — Fine-Tuning

### Approach

1. **Supervised Fine-Tuning (SFT)**
   - (state → action)

2. **Recovery Learning**
   - handle failures, popups, edge cases

3. **Preference Optimization**
   - choose better actions

4. **Reinforcement Learning (optional)**
   - optimize efficiency and robustness

---

## 📈 Impact

- Reduced brittle scripts
- Enabled goal-driven testing
- Improved adaptability
- Built foundation for self-improving QA

---

## 👨‍💻 My Role

- Designed system architecture
- Built hybrid agent framework
- Integrated GUI-Owl-7B
- Designed training pipeline (SFT → RL)
- Defined evaluation metrics

---

## 🔑 Key Takeaways

> AI + Systems + Data = Future of Testing

---

## 📌 Summary

Built a hybrid GUI testing agent using **GUI-Owl-7B**, structured UI state, and deterministic execution.  
Enhanced with fine-tuning and learning loop for robust banking app automation.

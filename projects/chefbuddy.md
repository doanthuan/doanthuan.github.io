---
layout: page
title: ChefBuddy — Kitchen Robot
permalink: /projects/chefbuddy/
---

# ChefBuddy: Kitchen Robot Using VLA & Diffusion Policy

**Date:** 2025

**URL**: [https://github.com/doanthuan/chefbuddy](https://github.com/doanthuan/chefbuddy)

**Project Overview:**
![ChefBuddy-Overview](/assets/chefbuddy/overview.jpeg)

- A personal hobby project exploring robotic manipulation in the kitchen. ChefBuddy uses NVIDIA's **GR00T N1.5** vision-language-action (VLA) model paired with an **SO-100/SO-101** robotic arm to perform pick-and-place tasks like dual-ingredient sandwich assembly. Built as a hands-on learning experience, the project covers the full pipeline: dataset recording via teleoperation, LoRA fine-tuning under memory constraints, systematic debugging, simulation with data augmentation, and real-time deployment.

## System Overview

The system combines a 6-DOF robotic arm with a 3B-parameter VLA model for language-conditioned manipulation:

- **Robot**: SO-100/SO-101 robotic arm (5 arm joints + 1 gripper)
- **Vision**: Dual-camera setup — wrist-mounted and scene overview — at 640x480, 30fps
- **Compute**: RTX 4080 Super (16GB VRAM)
- **Model**: NVIDIA GR00T N1.5 (3B parameters) with Isaac-GR00T framework
- **Data Format**: LeRobot v3.0 with HuggingFace Hub integration

---

## 🤖 Key Features

### 1. Teleoperation & Data Collection

Demonstrations are recorded through teleoperation of a leader arm, with dual cameras capturing synchronized observations. Datasets are stored in **LeRobot v3.0 format** and shared via **HuggingFace Hub**.

- Dual-ingredient sandwich assembly datasets (cheese: 50 episodes / 14,212 frames; bread: 50 episodes / 13,483 frames)
- Persistent camera device mapping via **udev rules** to ensure consistent `/dev/wrist` and `/dev/leader` symlinks across reboots
- OpenCV-based capture with sustained 30+ FPS on both cameras after hardware debugging

---

### 2. Memory-Optimized LoRA Fine-Tuning

Standard fine-tuning of GR00T N1.5 requires ~200M trainable parameters, exceeding 16GB VRAM. **LoRA** reduces this to ~10M parameters (20x reduction):

- `lora-rank 32`, `lora-alpha 64`, `lora-dropout 0.1`
- Batch size 16, gradient accumulation 8 steps, learning rate 0.0001
- Training up to 5,000+ steps for pick-and-place convergence
- Auto-detection of robot-specific joint keys (`single_arm`, `gripper`) replacing hardcoded humanoid defaults
- Corrected camera mapping: `observation.images.main` (wrist) and `observation.images.secondary_0` (scene)

---

### 3. Language-Conditioned Multitask Training

A critical discovery: training with `--no-tune_diffusion_model` froze the Eagle VLM backbone, causing the model to ignore language instructions entirely — behavior was 100% visual, 0% language-conditioned.

**Solution:** Unfreezing the diffusion model component enabled proper language grounding, allowing the robot to differentiate tasks (e.g., "pick up cheese" vs. "pick up bread") based on the instruction.

---

### 4. Simulation & Data Augmentation

**Isaac Sim Environment:**

- Custom USD scene for multi-ingredient sandwich assembly (2 bread slices, cheese, patty)
- Physics-enabled workspace (1.2m x 0.8m x 0.85m table) with dual-camera system
- Optimized scene from 37.9 MB to 5-10 MB by removing unnecessary kitchen fixtures

**MimicGen Data Augmentation Pipeline:**

1. Convert joint-space actions (6D) to end-effector actions (8D)
2. Automatically detect subtask boundaries
3. Generate variations by recombining subtask segments
4. Convert back to joint-space

**Result:** 1 original demonstration expanded to 10 augmented variations with **71.4% generation success rate**. Height threshold calibrated from default 0.20m to 0.05m to match actual cube dimensions (0.015m).

---

### 5. Real-to-Sim Digital Twin

Implemented real-to-sim synchronization between the physical SO-101 and Isaac Sim via **ROS2**:

- **Teleoperation node** reads leader arm positions
- **Joint state bridge** publishes follower positions
- **Isaac Sim subscriber** mirrors joint commands in simulation

Key fixes: resolved GLIBCXX library conflicts by using Isaac Sim's internal ROS2 libraries, isolated network topics with `ROS_DOMAIN_ID=42`, and corrected joint name mismatches in ArticulationController.

---

### 6. Client-Server Deployment

Local inference architecture using **ZMQ** protocol:

- **Server**: Loads GR00T model on GPU, runs inference on incoming observations, returns action predictions
- **Client**: Captures camera images, sends observations to server, executes returned actions on the robot
- Bypasses browser-based control limitations (joint count mismatches, device mismatch errors)

---

## 🔧 Key Challenges & Solutions

| Challenge                               | Root Cause                           | Solution                                  |
| --------------------------------------- | ------------------------------------ | ----------------------------------------- |
| Robot ignores language instructions     | VLM backbone frozen during training  | Unfreeze diffusion model component        |
| Gripper unresponsive, tiny oscillations | Model undertrained at 2,000 steps    | Extended training to 5,000-15,000+ steps  |
| MimicGen generation failures            | Height threshold too strict (0.20m)  | Calibrated to 0.05m for 1.5cm cube        |
| Camera devices swap on reboot           | No persistent device mapping         | udev rules with vendor/product/serial IDs |
| ROS2 topic interference                 | Shared ROS_DOMAIN_ID across machines | Unique domain ID (42) for isolation       |
| VRAM exceeded during fine-tuning        | ~200M trainable parameters           | LoRA reducing to ~10M parameters          |

---

## 📈 Results & Contributions

- **Language-conditioned multitask manipulation** — robot correctly differentiates tasks based on natural language instructions
- **71.4% MimicGen augmentation success** — automated expansion of single demonstrations into multiple training variations
- **Real-to-sim digital twin** — bidirectional synchronization between physical robot and Isaac Sim
- **Comprehensive debugging infrastructure** — enhanced logging with action statistics, joint testing tools, real-time state monitoring, and dataset visualization
- **Memory-efficient training** — full GR00T N1.5 fine-tuning on a single 16GB consumer GPU via LoRA

---

## 🛠 Technical Stack

- NVIDIA GR00T N1.5, Isaac-GR00T, Isaac Sim
- LeRobot v3.0, HuggingFace Hub
- LoRA fine-tuning, MimicGen
- ROS2, ZMQ, OpenCV
- Python, udev, CycloneDDS

---

# 🌍 Home Explorer

## Date: 2023

## 🚀 Overview

**Home Explorer** is a real-time **SLAM + Semantic Mapping system** powered by **2D LiDAR** and running entirely on edge hardware using **NVIDIA Jetson Nano**.

It combines:

* 📍 **2D LiDAR SLAM (robust localization & mapping)**
* 👁️ **Camera-based object detection**
* 🧠 **Semantic map fusion**

👉 Result: a system that builds a **precise map** and understands **what exists in that space**

---


## ✨ Features

* ✅ Real-time **2D LiDAR SLAM**
* ✅ Robust mapping (lighting-independent)
* ✅ On-device **object detection (TensorRT optimized)**
* ✅ **Semantic map (objects anchored in space)**
* ✅ Fully **edge-based (Jetson Nano)**

---

## 🏗️ System Architecture

```id="z6c3y2"
2D LiDAR Scan
     ↓
SLAM (GMapping / Hector SLAM / Cartographer)
     ↓
Occupancy Grid Map + Robot Pose
     ↓
Camera Input → Object Detection (YOLO TensorRT)
     ↓
Coordinate Projection (Camera → LiDAR frame)
     ↓
Semantic Fusion
     ↓
Semantic Occupancy Map
```

![Flow](/assets/slam/flow.png)

---

## ⚙️ Tech Stack

### 📍 SLAM (LiDAR-based)

* GMapping
* Hector SLAM
* Cartographer

📌 Why LiDAR SLAM?

* More **stable than vision SLAM**
* Works in **low light / textureless environments**
* Widely used in robotics

---

### 👁️ Object Detection (Edge AI)

* YOLOv5n / YOLOv8n (TensorRT)
* MobileNet-SSD (lighter)

---

### 🔧 Platform

* Jetson Nano (CUDA + TensorRT)
* ROS (Robot Operating System)

---

## 🧰 Hardware Setup

### 🔧 Core Components

* 🟢 NVIDIA Jetson Nano
* 📡 2D LiDAR:

  * RPLIDAR A1 / A2 (budget-friendly)
* 📷 USB Camera

### ⚙️ Optional

* IMU (for better localization)
* Robot base (for mobility)

---

## 📊 Example Output

### 🗺️ Occupancy Grid Map

* 2D map of environment
* Walls, obstacles, free space

### 🧠 Semantic Layer

* Objects placed on map:

  * 🪑 Chair → mapped position
  * 🚪 Door → detected region
  * 🖥️ Monitor → workspace

---
![Overview](/assets/slam/slam.jpg)

## 🧠 How It Works

### 1. LiDAR SLAM

* Builds occupancy grid map
* Estimates robot pose

### 2. Object Detection

* Detects objects from camera stream
* Runs fully on Jetson Nano (TensorRT)

### 3. Coordinate Fusion

* Transform camera detections → LiDAR frame
* Use calibration (extrinsics)

### 4. Semantic Mapping

* Attach labels to map positions
* Build **semantic occupancy grid**


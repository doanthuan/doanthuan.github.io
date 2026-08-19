---
layout: page
title: 
permalink: /research/knowledge-engineering/
---
<h3></h3>

## Solve geometry math problems using symbolic programming and ontology

**Date:** 2021\
**Github**: [https://github.com/doanthuan/intelligent-system-solving](https://github.com/doanthuan/intelligent-system-solving)

### Overview
- Solve geometry math problems using symbolic programming and ontology


<img src="/assets/ke/overal.png" width="500"/>

**Technologies:**
- Python, sympy, flask


### Problem Statement
- **Input**: Given calculation objects are points, line segments, angles or triangles. The hypothesis will be given some known factors, the conclusion will be some unknown factors or prove a certain relationship or equality. The program allows entering questions and problems in natural language.

- **Output**: present the solution in the form of specific steps.
Requirement: model the problem (model and method of representing knowledge related to triangles - some), design a deductive algorithm based on that representation method.

### Method
For basic plane geometry, specifically knowledge about triangles and some related parts, knowledge base will be designed as code in Python files, using data types, data structures in Python as well as the SymPy library that supports it.

#### Symbolic programming with SymPy

<img src="/assets/ke/pic5.png" width="500"/>

<img src="/assets/ke/pic6.png" width="500"/>

<img src="/assets/ke/pic7.png" width="500"/>

<img src="/assets/ke/pic8.png" width="500"/>

<img src="/assets/ke/pic9.png" width="500"/>

<img src="/assets/ke/pic10.png" width="500"/>



#### Implementation
- **Objects**: Each calculation object is represented as a class in Python.

<img src="/assets/ke/pic1.png" width="500"/>


- **Rules**: inference rules are expressed as if then functions:

<img src="/assets/ke/pic2.png" width="500"/>

### Deductive Algorithms
For the topic of solving some basic plane geometry problems, the problem model also has three main components as follows:

O={O_1,O_2,…,O_n } are abstract concepts and calculation objects, for example: triangle, angle, edge, perpendicular line,…

F={F_1,F_2,…,F_n } are known events, given by the problem, for example: a=5, A ̂=π/2,…

Goal={g_1,g_2,…,g_n } is the target set, which can be determining values, or finding relationships…

The main algorithms applied in the program are the progressive deductive algorithm and the propagation algorithm. In addition, many other algorithms are applied to support many different types of problems.

### Result
- The system allows solving some types of 7th grade geometry problems, including problems about points, angles, lines, triangles...
- Allows entering questions and problems in natural language

**Demo:** [https://youtu.be/h87Mec6faqo](https://youtu.be/h87Mec6faqo )

<img src="/assets/ke/pic3.png" width="500"/>

<img src="/assets/ke/pic4.png" width="500"/>

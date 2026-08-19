---
layout: page
title: 
permalink: /research/agent-code/
---
<h3></h3>

## AI Agents for Software Engineering

### Introduction
- The recent advance in Large Language Models (LLMs) has shaped a new paradigm of AI agents, i.e., LLM-based agents.
Compared to standalone LLMs, LLM-based agents substantially extend the versatility and expertise of LLMs by enhancing LLMs with
the capabilities of perceiving and utilizing external resources and tools. To date, LLM-based agents have been applied and shown
remarkable effectiveness in Software Engineering (SE). The synergy between multiple agents and human interaction brings further
promise in tackling complex real-world SE problems

<img src="/assets/agent-code/overal.png" width="600"/>

### Perspectives
- **SE perspective:** how LLM-based agents are applied across different software development
and improvement activities, including individual tasks (e.g., **requirements engineering, code generation, static code checking, testing, and debugging**)
 as well as the endto-end procedure of software development and improvement. From this perspective, we provide a comprehensive
overview of how SE tasks are tackled by LLM-based agents

- **Agent perspective:** the design of components in LLM-based agents for SE. Specifically,
we analyze key components, including **planning, memory, perception, and action**, in these agents. Beyond basic agent
construction, we also analyze multi-agent systems, including their agent roles, collaboration mechanisms, and humanagent collaboration. From this perspective, we summarize
the characteristics of different components of LLM-based agents when applied to the SE domain

![Image](/assets/agent-code/perspective.png)

### Background

#### Basic Framework of LLM-based Agents

LLM-based agents are typically composed of four key components: *planning, memory, perception, and action *. 
The planning and memory serve as the key components of the LLM-controlled brain, which interacts with the environment
through the perception and action components to achieve specific goals. Figure 2 illustrates the basic framework of
LLM-based agents.

**Planning.** The planning component decomposes complex tasks into multiple sub-tasks and schedules the subtasks to achieve final goals. In particular, agents can (i)
generate a plan without adjustment by different reasoning
strategies, or (ii) adjust a generated plan with the external
feedback (e.g., environmental feedback or human feedback).

**Memory.** The memory component records the historical
thoughts, actions, and environmental observations generated during the agent execution. Based on
the accumulated memory, agents can revisit and utilize
the previous records and experience, so as to tackle the
complex tasks more effectively. The memory management
(i.e., how to represent the memory) and utilization (i.e., how
to read/write or retrieve the memory) are essential, which
directly impact the efficiency and effectiveness of the agent
system.

**Perception.** The perception component receives the information from the environment, which can facilitate better
planning. In particular, agents can perceive multi-modal inputs, e.g., textual inputs, visual inputs, and auditory inputs.
Action. Based on the planning and decisions made by

**Action.** Based on the planning and decisions made by
the brain, the action component conducts concrete actions
to interact with and impact the environment. One essential
mechanism in action is to control and utilize external tools,
which can extend the inherent capabilities of LLMs by
accessing more external resources and extending the action
space beyond textual-alone interaction

<img src="/assets/agent-code/framework.png" width="600"/>

#### Advanced LLM-based Agent Systems

**Multi-agent Systems.** While a single-agent system can
be specialized to solve one certain task, enabling the collaboration between multiple agents (i.e., multi-agent systems) can
further solve more complex tasks associated with diverse
knowledge domains. In particular, in a multi-agent system,
each agent is assigned a distinct role and relevant expertise, making it specifically responsible for different tasks;
in addition, the agents can communicate with each other
and share the progress/information as the task proceeds.
Typically, agents can work collaboratively (i.e., by working
on different sub-tasks to achieve a final goal) or competitively (i.e., by working on the same task while debating
adversarially).

**Human-Agent Coordination.** Agent systems can further
incorporate the instructions from humans and then proceed
with tasks under human guidance. This human-agent coordination paradigm facilitates better alignment with human
preference and uses human expertise. In particular, during
human-agent interaction, humans can not only provide
agents with task requirements and feedback on the current
task status, but also cooperate with agents to achieve goals
together.


### Software Engineering Perspectives
<img src="/assets/agent-code/pic1.png"/>

#### Requirements Engineering
Requirements Engineering (RE) is a crucial phase for initializing the software development procedure. Generally, it
can cover the following phases .
- Elicitation: New requirements are elicited and collected.
- Modeling: Abstract yet interpretable models are used to
describe requirements, e.g., Unified Modeling Language
(UML)and Entity-Relationship-Attribute (ERA)
model.
- Negotiation: Negotiation plays a crucial role in facilitating
communication of different stakeholders and ensuring
consistency, especially in conflicting requirements.
- Specification: Requirements are determined and documented in a formal format.
- Verification: Requirements and models are validated to
ensure they fully and unambiguously reflect the intent of
stakeholders.
- Evolution: Requirements evolution refers to the ongoing
process of refining and adapting requirements in response
to changing needs and conditions

<img src="/assets/agent-code/pic2.png" width="600"/>


#### Code Generation

Code generation has been extensively explored with the
development of AI technology. Due to being pre-trained
on massive textual data (especially large code corpus), LLMs
demonstrate promising effectiveness in generating code for
given code contexts or natural language descriptions. Nevertheless, the code generated by LLMs can sometimes be
unsatisfactory due to issues such as the notorious hallucination. Therefore, beyond simply leveraging standalone
LLMs for code generation, researchers also build LLMbased agents that can enhance the capabilities of LLMs via
planning and iterative refinement

<img src="/assets/agent-code/pic3.png" width="600"/>

- Existing LLM-based Agents for Code Generation

<img src="/assets/agent-code/pic5.png" />



#### Static Code Checking
Static code checking refers to examining the quality of code
without executing the code. In particular, static code checking has been essential in the modern continuous integration pipeline, as it is efficient to identify diverse categories
of code quality issues (e.g., different bugs, vulnerabilities,
or code smells) before extensively executing the tests. In
practice, it is common to adopt static analysis techniques
to automatically detect bugs/vulnerabilities (i.e., static bug
detection) or involve peer reviews to check the quality of
code (i.e., code review).
<img src="/assets/agent-code/pic6.png" width="600"/>


#### Testing
Software testing is essential for software quality assurance.
LLMs have demonstrated promising proficiency in test generation, including generating test code, test inputs, and test
oracles. However, generating high-quality tests in practice
can be challenging, as the generated tests should not only
be syntactically and semantically correct (i.e., both the inputs
and oracles should satisfy the specification of the software
under test) but also be sufficient (i.e., the tests should cover
as many states of the software under test as possible). As
shown by previous work, the tests generated by standalone LLMs still exhibit correctness issues (i.e., compilation
errors, run-time errors, and oracle issues) and unsatisfactory
coverage. Therefore, researchers build LLM-based agents to
extend the capabilities of standalone LLMs in test generation. We organize these works based on the test levels (i.e.,
unit testing and system testing)
<img src="/assets/agent-code/pic9.png" width="600"/>

- **Unit Testing:**
Unit testing checks the isolated and small unit (e.g., method
or class) in the software under test, which helps quickly
identify and localize the bugs, especially for complicated
software systems. Yuan et al. [122] perform a study showing
the potentials of LLMs (e.g., ChatGPT) in generating unit
tests with decent readability and usability. However, the
unit tests generated by standalone LLMs still exhibit compilation/execution errors and limited coverage. Therefore,
recent works have built LLM-based agents that primarily extend standalone LLMs by iteratively refining the generated
unit tests towards better correctness, coverage, and fault
detection capabilities
<img src="/assets/agent-code/pic10.png"/>


- **System Testing:**
System testing is a comprehensive process that assesses
an integrated software system/component to guarantee
that it fulfills its specification and operates as intended
across diverse settings. For example, fuzzing testing and
GUI (Graphical User Interface) testing are common testing
paradigms at the system level. Leveraging LLMs for system
testing can be challenging, as generating valid and effective
system-level test cases should satisfy the constraints that are
contained implicitly and explicitly in the specifications or
domain knowledge of the software system under test. LLMbased agents are designed to better incorporate the domain
knowledge of the software system under test compared to generating system-level tests via standalone LLMs. We then
organize these works according to the software systems under test

<img src="/assets/agent-code/pic11.png"/>


#### End-to-end Software Development

Given the high autonomy and the flexibility from multiagent synergy, LLM-based agent systems can further tackle
the end-to-end procedure of software development (e.g.,
developing a Snake Game application from scratch) beyond
an individual phase of software development. In particular,
alike the real-wold software development team, these agent
systems can cover the entire software development life cycle
(i.e., requirements engineering, architecture design, code
generation, and software quality assurance) by incorporating the synergy between multiple agents that are specialized
with different roles and relevant expertise. Table 10 summarizes the existing LLM-based agents for end-to-end software
development.

<img src="/assets/agent-code/pic12.png"/>

- **Benchmarks for End-to-end Software Development**:
<img src="/assets/agent-code/pic13.png"/>

#### End-to-end Software Maintenance

Software systems undergo maintenance as requirements
continuously change (i.e., adding, deleting, or modifying features) or unexpected software behaviors arise. In
practice, users report unsatisfactory behaviors that they
encounter; developers then diagnose the reported issues
and modify the software to fix them. Such an end-to-end
software maintenance process can be time-consuming and
labor-intensive in practice, as it involves multiple phases
including understanding user-reported issues, localizing
code for maintenance, and precisely editing code to address
issues. Recently, there has been an increasing number of
multi-agent systems aiming at automatically solving issues
of real-world software projects.

<img src="/assets/agent-code/pic14.png"/>


### Agent Perspectives

Based on the common framework of LLM-based
agents, this section summarizes the common
paradigms of the **planning, perception, memory, and action**
components in existing LLM-based agents for SE

#### Planning

In SE, intricate tasks such as development and maintenance activities necessitate the orchestrated efforts of various agents through multiple iterative cycles. Therefore,
planning is an essential component for agent systems by
meticulously delineating task sequences and strategically
scheduling agents to ensure the seamless progression of the
SE process

<img src="/assets/agent-code/pic15.png"/>

#### Memory
The memory component is a pivotal mechanism responsible
for storing the trajectories of historical thoughts, actions,
and environmental observations, enabling agents to sustain
coherent reasoning and address intricate tasks. In SE, complex development and maintenance tasks generally necessitate agents conducting iterative revisions, wherein historical
intermediate information, e.g., generated code and testing
reports, significantly impacts integrity and continuity. We
then detail the implementation of memory mechanisms in
SE from four perspectives: memory duration, ownership,
format, and operation.

<img src="/assets/agent-code/pic16.png"/>

#### Perception
Existing LLM-based agents for SE primarily adopt two
perception paradigms: textual input perception and visual
input perception

#### Action
The action component of existing LLM-based agents for SE
primarily involves using external tools to extend their capabilities beyond the interactive dialogue typical of standalone
LLMs. 
<img src="/assets/agent-code/pic17.png"/>

### Multi-Agent System
Based on our statistics, 52.8% of existing agents for SE
are multi-agent systems. These systems benefit from the
division of specialized roles and coordination among agents,which effectively addresses the complexity of SE tasks, particularly for end-to-end activities spanning multiple phases.
This section provides an overview of existing multi-agent
systems for SE, with a focus on their agent roles and
coordination mechanisms

<img src="/assets/agent-code/pic18.png" width="600"/>

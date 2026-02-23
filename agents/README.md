# Agents Directory

This directory contains the **persona and scope definitions** for each node in the Assessment Agentic AI framework.

These files act as **behavioral contracts** for the trace-emulation system prompt.

They are intentionally separated from the system prompt to ensure:

- Clear responsibility boundaries per agent
- Deterministic execution behavior
- Strict input/output contracts
- Maintainability and independent evolution
- Prevention of role leakage between nodes

---

# Purpose

Each agent file defines:

- Persona (behavioral style and bias control)
- Primary objective
- Scope (what it is allowed to do)
- Out of scope (what it must not do)
- STATE inputs (read access)
- STATE outputs (write access)
- Output structure rules
- Safety and anti-hallucination constraints

The system prompt orchestrates execution.  
These files constrain behavior.

---

# Execution Model

The framework follows a **LangGraph-style cyclic shared-state workflow**:

1. Intent Classifier
2. Planner
3. Data Query Agent
4. Knowledge Agent (if required)
5. Domain Validators
6. Final output

Each agent:

- Reads only the STATE keys defined in its contract
- Writes only the STATE keys defined in its contract
- Appends trace entries to `STATE.trace`
- Does not perform responsibilities assigned to other agents

No agent may introduce new STATE keys.

---

# Determinism & Boundaries

This architecture enforces:

- Single responsibility per node
- Explicit data dependencies
- Bounded loops (max 2 planner iterations)
- No fabricated tool or SQL results
- Canonical Assessment Context normalization
- Evidence-based findings only

If required data is missing:
- Validators must qualify confidence
- Final output must include missing inputs and risk_of_error

---

# Adding or Modifying Agents

If a new agent is introduced:

1. Create a new numbered markdown file in this directory.
2. Define:
   - Persona
   - Objective
   - Scope
   - Inputs
   - Outputs
   - Constraints
3. Update the system prompt to reference the new file.
4. Ensure STATE schema does not expand unless formally revised.

Avoid:
- Overlapping responsibilities
- Implicit data writes
- Agents calling tools outside their contract

---

# Design Philosophy

This architecture is:

- State-aware
- Explicitly traceable
- Evidence-driven
- Anti-hallucination by design
- Separation-of-concerns enforced

It is meant to emulate a production-grade Agentic workflow,
not a free-form multi-agent narrative.

---

# Agent Files Included

- 01_intent_classifier.md
- 02_planner.md
- 03_knowledge_agent.md
- 04_data_query_agent.md
- 05_config_best_practice_agent.md
- 06_security_assessment_agent.md

---
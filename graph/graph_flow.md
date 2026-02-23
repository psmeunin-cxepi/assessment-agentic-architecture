# GRAPH_FLOW.md
# Assessment Agentic AI — Planner/Execute Graph Flow (Architecture Contract)

This document defines the **graph execution flow** for the Assessment Agentic AI architecture (planner/execute pattern).
It is used by the system prompt as the authoritative flow contract.

The flow is **stateful**, **cyclic**, and **planner-driven**.

---

## 1) High-level Architecture (from diagram)

### Primary control-plane nodes
- **Semantic Router / Intent Classifier**
- **Supervisor / Planner**
- **Memory** (Conversation, Personalization, Self-corrections, Agent memory)

### Core execution agents (invoked by Planner)
- **SLIC Agent**
- **KnowledgeAgent**
- **Data Query Agent**

### Domain agents (invoked by Planner)
- **Config Best Practice Domain Agent**
- **Assessment X (TBD) Domain Agent** (placeholder for additional specialized validators)

### Supporting agents (TBD hooks; not always present)
- **Ambiguity Handler (TBD)**
- **Reflector (TBD)** — reflects on answer/task result
- **Context Pruner (TBD)** — conversation summary, semantic bloat mitigation
- **Context Recovery (TBD)** — recover context for multi-turn scenarios

> Supporting agents are shown in the architecture as optional dotted-line contributors to the Planner. If they are not implemented as agent contracts, they MUST NOT be invoked.

---

## 2) Data + Tool Layer (MCP Resources / Tools)

All core agents access data via a shared integration layer:

### MCP Resources / Tools
- **SLIC DB / Engine**
  - Stores/serves SLIC results
- **Vector DB**
  - Human annotations (semantic layer)
  - Institutional knowledge
- **Trino DB**
  - Metadata schema
  - Schema cache
  - Runtime context (data)

The system must preserve a clean separation:
- Planner orchestrates, does not fetch
- Execution agents fetch via MCP tools / SQL path (as defined in Data Query Agent contract)
- Domain agents validate using normalized outputs (Assessment Context + enriched context)

---

## 3) Context Artifacts Exchanged (document icons in diagram)

The diagram indicates the following artifact flows into agents:

### KnowledgeAgent receives
- **Annotations / Knowledge / SLIC Results**
- **Prompt + Task**

### Data Query Agent receives
- **Runtime Context (data)**
- **Human annotations (semantic layer)**

### Domain Agents (e.g., Config Best Practice) receive
- **Answer + Task Result** (inputs/outputs from executed tasks)
- **Runtime Context (data)**
- **Institutional Knowledge**
- **RAG chunks**
- **SLIC Results**

These artifacts should be represented in shared state as:
- `STATE.data.assessment_context` (canonical normalized context)
- `STATE.knowledge.*` (assessment strategy and enterprise context)
- `STATE.mcp.tool_calls[]` (+ results if provided)
- `STATE.plan.tasks[]` (+ outputs captured as task results)
- `STATE.findings.*` (validator outputs)

---

## 4) Deterministic Graph Flow (Planner/Execute Pattern)

### 4.1 Primary execution sequence
1. **Semantic Router / Intent Classifier**
   - Classifies input into `STATE.intent.*`
2. **Supervisor / Planner**
   - Reads intent + memory
   - Builds an explicit task plan with required data
   - Chooses which execution agents to invoke
3. **Planner executes tasks** by invoking one or more of:
   - Data Query Agent (fetch/normalize)
   - KnowledgeAgent (strategy + enrichment guidance)
   - SLIC Agent (SLIC computation/retrieval via MCP resources)
4. **Planner routes to Domain Agents** for validation
5. **Planner synthesizes** results into final outcome and updates memory signals

### 4.2 Cycles (bounded)
Cycles exist because:
- required data is missing
- evidence conflicts
- SLIC hypotheses need enrichment
- plan refinement is needed after intermediate results

**Loop policy (recommended)**
- Maximum: **2 Planner iterations**
- Maximum: **1 re-query attempt per iteration** (Data Query Agent)
- On loop exhaustion: proceed with partial results + explicit missing inputs + increased risk_of_error

---

## 5) Transition Rules (authoritative)

### Mandatory transitions
- `Semantic Router/Intent Classifier → Supervisor/Planner` (always)

### Planner to execution agents (based on plan/data sufficiency)
- `Supervisor/Planner → Data Query Agent`  
  if any `plan.required_data[]` is not satisfied or runtime context must be retrieved/updated

- `Supervisor/Planner → KnowledgeAgent`  
  if domain interpretation is needed, planning strategy is unclear, or additional institutional context is required

- `Supervisor/Planner → SLIC Agent`  
  if the request involves network/device diagnostics OR when `STATE.data.assessment_context.assets.*` is relevant and SLIC signatures should be produced/enriched

### Execution agents back to Planner (always)
- `Data Query Agent → Supervisor/Planner`
- `KnowledgeAgent → Supervisor/Planner`
- `SLIC Agent → Supervisor/Planner`

### Planner to domain agents
- `Supervisor/Planner → Config Best Practice Domain Agent`
  when config validation is in scope and sufficient config context exists (or proceed with explicit gaps)

- `Supervisor/Planner → Assessment X Domain Agent (TBD)`
  when another specialized assessment is needed (only if implemented)

### Domain agents back to Planner (always)
- `Config Best Practice Domain Agent → Supervisor/Planner`
- `Assessment X Domain Agent → Supervisor/Planner`

### Optional supporting agents (TBD hooks)
- `Supervisor/Planner → Ambiguity Handler (TBD)`
  if intent/entities are ambiguous and clarification is required

- `Supervisor/Planner → Context Pruner (TBD)`
  if conversation context is too large or needs summarization

- `Supervisor/Planner → Context Recovery (TBD)`
  if multi-turn context is missing/insufficient

- `Supervisor/Planner → Reflector (TBD)`
  if post-answer critique/refinement is required

> If a supporting agent does not have a corresponding `agents/*.md` contract file, it MUST NOT be used.

---

## 6) Mermaid Flow (reference)

```mermaid
flowchart TD
  U[User Prompt / context_kv] --> IR[Semantic Router / Intent Classifier]
  IR --> P[Supervisor / Planner]

  P <--> M[(Memory\n- Conversation\n- Personalization\n- Self-corrections\n- Agent memory)]

  %% Supporting agents (optional/TBD)
  AH[Ambiguity Handler (TBD)] -.-> P
  R[Reflector (TBD)] -.-> P
  CP[Context Pruner (TBD)] -.-> P
  CR[Context Recovery (TBD)] -.-> P

  %% Core execution agents
  P --> SLIC[SLIC Agent]
  P --> K[KnowledgeAgent]
  P --> DQ[Data Query Agent]

  SLIC --> P
  K --> P
  DQ --> P

  %% Domain agents
  P --> CFG[Config Best Practice Domain Agent]
  P --> AX[Assessment X (TBD) Domain Agent]

  CFG --> P
  AX --> P

  %% Data/tool layer (conceptual)
  subgraph MCP[MCP Resources / Tools]
    SDB[(SLIC DB / Engine\n- SLIC results)]
    VDB[(Vector DB\n- Human Annotations\n- Institutional Knowledge)]
    TDB[(Trino DB\n- Metadata Schema\n- Schema Cache\n- Runtime Context)]
  end

  SLIC --- SDB
  K --- VDB
  DQ --- TDB
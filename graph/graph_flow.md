# graph_flow.md
# Assessment Agentic AI — Planner/Execute Graph Flow (Architecture Contract)

This document defines the **graph execution flow** for the Assessment Agentic AI architecture (planner/execute pattern).
It is used by the system prompt as the authoritative flow contract.

The flow is **stateful**, **planner-driven**, and **single-pass** (cyclic replanning is a future enhancement — see Section 4.2).

---

## 1) High-level Architecture (from diagram)

### Primary control-plane nodes
- **Semantic Router / Intent Classifier**
- **Supervisor / Planner**
- **Memory** (Conversation, Personalization, Self-corrections, Agent memory)

### Core execution agents (invoked conditionally by Planner)
- **Knowledge Agent** — enterprise knowledge retrieval (RAG)
- **Data Query Agent** — assessment data retrieval (MCP tools or SQL queries)

### Domain agents (invoked by Planner)
- **Config Best Practice Domain Agent** — configuration standards validation (skills: `cbp_assessment`, `cbp_expert_insights`, `cbp_generic`)
- **Security Assessment Agent** — security posture assessment (skill: `security_assessment`)
- **Assessment X (TBD) Domain Agent** (placeholder for additional specialized validators)

### Supporting agents (TBD hooks; not always present)
- ~~**Ambiguity Handler (TBD)**~~ — realized by Intent Classifier clarification gate (see Section 5)
- **Reflector (TBD)** — reflects on answer/task result
- **Context Pruner (TBD)** — conversation summary, semantic bloat mitigation
- **Context Recovery (TBD)** — recover context for multi-turn scenarios

> Supporting agents are shown in the architecture as optional dotted-line contributors to the Planner. If they are not implemented as agent contracts, they MUST NOT be invoked.

---

## 2) Data + Tool Layer (MCP Resources / Tools)

All core agents access data via a shared integration layer:

### MCP Resources / Tools
- **MCP Tools** (registered in `mcp_server/server.py`; full contracts in [`tools/mcp/`](../tools/mcp/))
  - `assessment_analysis_tool` — current assessment analysis (summary, filtering, assets, technology, exceptions)
  - `assessment_comparison_tool` — compare current vs previous assessment (trend, delta, asset changes)
  - `issue_tracking_tool` — unresolved issues and persistently failing rules
  - Internal service functions (`get_assessment_summary`, `compare_assessments`, `get_unresolved_issues`, `get_assessment_details`) are implementation details not directly callable by agents
- **Vector DB**
  - Enterprise knowledge (RAG)
  - Policies, standards, approved exceptions, organizational context
- **Trino DB (SQL Path)**
  - Raw configuration data
  - Assessment metadata
  - Requires schema + ontology for query generation

The system must preserve a clean separation:
- Planner orchestrates, does not fetch
- Execution agents fetch via MCP tools / SQL path (as defined in Data Query Agent contract)
- Domain agents validate using normalized outputs (Assessment Context + enriched context)

---

## 3) Context Artifacts Exchanged (document icons in diagram)

The diagram indicates the following artifact flows into agents:

### Knowledge Agent receives
- **User intent** (from Intent Classifier)
- **Task definition** (from Planner)

### Data Query Agent receives
- **Intent entities** (semantic parameters from Intent Classifier)
- **Enterprise context** (optional, from Knowledge Agent task outputs)
- **Schema + Ontology** (for SQL Path)

### Domain Agents (e.g., Config Best Practice) receive
- **Assessment context** (from Data Query Agent task outputs)
- **Enterprise context** (from Knowledge Agent task outputs, if available)
- **Intent entities** (for scope filtering)

These artifacts are represented in shared state as:
- `STATE.intent.*` (intent_class, entities[])
- `STATE.plan.tasks[]` (task definitions with embedded required_data and outputs)
  - Each task contains `outputs: {}` where agent writes results
  - Knowledge Agent writes: `outputs.enterprise_context` (RAG chunks)
  - Data Query Agent writes: `outputs.assessment_context` (MCP or SQL data)
  - Domain Agents write: `outputs.findings[]`, `outputs.summary`, `outputs.prioritized_risks[]`, `outputs.asset_trend[]` (trend mode), `outputs.chart_hints[]` (optional)
- `STATE.schema` (database structure for SQL Path)
- `STATE.ontology` (data semantics for SQL Path)
- `STATE.trace.*` (execution provenance)

---

## 4) Deterministic Graph Flow (Planner/Execute Pattern)

### 4.1 Primary execution sequence
1. **Semantic Router / Intent Classifier**
   - Classifies input into `STATE.intent.intent_class`
   - Extracts semantic entities into `STATE.intent.entities[]` (site, device, timeframe, severity, etc.)
2. **Supervisor / Planner**
   - Reads intent + entities
   - Builds conditional task plan:
     - **Knowledge Agent task** (if enterprise context needed for query interpretation or policy reference)
     - **Data Query Agent task** (if domain agent has data dependencies)
     - **Domain Agent task(s)** (always - performs assessment/analysis)
   - Each task includes embedded `required_data[]` for domain agents
   - Tasks have `depends_on[]` for execution ordering
3. **Graph executes tasks sequentially** (respecting dependencies):
   - Knowledge Agent (conditional) → writes `task.outputs.enterprise_context` (RAG chunks)
   - Data Query Agent (conditional) → writes `task.outputs.assessment_context` (MCP or SQL data)
   - Domain Agent(s) → reads upstream task outputs, writes `task.outputs.findings[]` or `task.outputs.summary`
4. **Graph returns final STATE** with all task outputs embedded

### 4.2 Single-pass execution (no replanning)

**Current implementation**: Plan/execute pattern with single iteration
- Planner creates complete task plan upfront
- Tasks execute once in dependency order
- No cycles or replanning logic

**Data insufficiency handling**:
- Domain agents qualify findings with `data_gaps[]` and `assumptions[]`
- Confidence scores reflect evidence quality
- Agents proceed with available data rather than blocking

**Future enhancement** (Phase 2+):
- Replanning may be reintroduced for complex multi-turn scenarios
- Would require stop_conditions and iteration limits

---

## 5) Transition Rules (authoritative)

### Clarification gate (pre-Planner)
- `Semantic Router/Intent Classifier → User` (clarification)
  **Condition**: `intent_class == "unknown_or_needs_clarification"`
  - Graph short-circuits: `STATE.intent.clarification_question` is returned to the user
  - Planner does NOT run; no tasks are created
  - User's reply re-enters the graph as a new `user_prompt`
  - This is a pre-planning gate — not a replan cycle; the single-pass constraint remains intact

### Mandatory transitions
- `Semantic Router/Intent Classifier → Supervisor/Planner` (when `intent_class != "unknown_or_needs_clarification"`)

### Planner to execution agents (conditional, based on requirements)
- `Supervisor/Planner → Knowledge Agent`  
  **Condition**: Determined by intent_class routing rules:
  - `cbp_expert_insights` → **Always** (enterprise knowledge required to enrich SLIC findings)
  - `cbp_generic` → **Always** (enterprise knowledge grounds the answer)
  - `cbp_assessment` → **Conditional** (invoke when enterprise enrichment improves answer quality)
  - `security_assessment` → **Conditional** (if enterprise context needed)
  - Follow-up questions needing policy/standard references
  - Queries about organizational practices or approved exceptions
  **Not invoked**: For `cbp_assessment` requests with no enterprise context needs

- `Supervisor/Planner → Data Query Agent`  
  **Condition**: Domain agent declares data dependencies (via `task.required_data[]`):
  - Configuration assessment needs assessment data (`cbp_assessment`, `cbp_expert_insights`)
  - Security assessment needs config/inventory data
  - Any domain agent with non-empty data dependencies
  **Not invoked**: For queries that can be answered from conversation history or general knowledge

### Execution agents to next task (via task dependency chain)
- Knowledge Agent completes → marks task status 'completed', writes outputs
- Data Query Agent completes → marks task status 'completed', writes outputs
- Domain agents read upstream task outputs via `tasks.find(t => t.depends_on.includes(task_id))`
- No explicit agent-to-Planner transitions (graph orchestrates via task execution order)

### Planner to domain agents
- `Supervisor/Planner → Config Best Practice Domain Agent`
  **Condition**: `intent_class` is one of `cbp_assessment`, `cbp_expert_insights`, or `cbp_generic`
  - `cbp_assessment` — assessment results, findings, stats, trends, risk, remediation
  - `cbp_expert_insights` — SLIC findings requiring enterprise knowledge enrichment
  - `cbp_generic` — general configuration / best-practice question (enterprise knowledge only, no assessment data)

- `Supervisor/Planner → Security Assessment Agent`
  **Condition**: `intent_class` is `security_assessment`

- `Supervisor/Planner → Assessment X Domain Agent (TBD)`
  when another specialized assessment is needed (only if implemented)

### Domain agents complete execution (final task in chain)
- Domain agents write final outputs to their task objects
- Mark task status as 'completed'
- Graph execution completes, returns final STATE to user interface

### Optional supporting agents (TBD hooks)
- ~~`Supervisor/Planner → Ambiguity Handler (TBD)`~~ — **Realized**: ambiguity is handled by the Intent Classifier's clarification gate (Section 5, Clarification gate). No separate Ambiguity Handler agent is needed.

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
  IR -->|intent_class known| P[Supervisor / Planner]
  IR -->|needs clarification| CQ[Return clarification_question to User]
  CQ -.->|user replies| U

  P <--> M[(Memory - Conversation, Personalization, Self-corrections, Agent memory)]
  REFL[Reflector TBD] -.-> P
  CP[Context Pruner TBD] -.-> P
  CR[Context Recovery TBD] -.-> P

  %% Core execution agents (conditional)
  P -->|if enterprise context needed| K[Knowledge Agent]
  P -->|if data dependencies exist| DQ[Data Query Agent]

  K -->|writes task.outputs| P
  DQ -->|writes task.outputs| P

  P --> CFG[Config Best Practice Domain Agent]
  P --> SEC[Security Assessment Agent]
  P --> AX[Assessment X TBD Domain Agent]

  CFG --> P
  SEC --> P
  AX --> P

  subgraph MCP[MCP Resources / Tools]
    MCPT[(MCP Tools - assessment_analysis_tool, assessment_comparison_tool, issue_tracking_tool)]
    VDB[(Vector DB - Enterprise Knowledge RAG, Policies and Standards)]
    TDB[(Trino DB - Raw config data, Schema and Ontology)]
  end

  K --- VDB
  DQ --- MCPT
  DQ --- TDB
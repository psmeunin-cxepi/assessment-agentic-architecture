# Deployment Options: Multi-Agent Services vs. Per-Domain Graph

**Purpose:** Compare two implementation approaches for the Assessment Agentic Architecture.
**Date:** 2026-02-27

---

## Overview

Both options share the same functional goal — route a user prompt through intent classification, planning, core data/knowledge retrieval, and domain-specific reasoning — but differ fundamentally in **how agents are deployed and interact at runtime**.

|                        | **Option 1: Multi-Agent Services**            | **Option 2: Per-Domain Graph**                                                                                  |
| ---------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Architecture style** | Distributed — agents are independent services | Per-domain — each domain gets its own LangGraph instance containing IC, Planner, KA, DQA, and Domain Task nodes |
| **Communication**      | Inter-service (A2A-ready, JSON-RPC, HTTP)     | Intra-graph (shared state object, function calls)                                                               |
| **Runtime boundary**   | Multiple processes / containers               | One process per domain graph                                                                                    |
| **Scaling unit**       | Per-agent                                     | Per-domain graph instance                                                                                       |

---

## Option 1: Multi-Agent Plan & Execute

### Description

Each agent runs as an **independent service** with its own runtime. A central Supervisor/Planner orchestrates by delegating tasks to Core Agents and Domain Agents via service calls. Each agent can have its own LLM, tools, and scaling policy.

### Architecture Diagram Summary

```
                    ┌───────────────────────┐
                    │ Semantic Router /      │
                    │ Intent Classifier      │
                    │ Agent                  │
                    └──────────┬────────────┘
                               ▼
                    ┌───────────────────────┐       ┌─────────┐
                    │ Supervisor / Planner  │       │         │
                    │ Agent                 │◄─────▶│ Memory  │
                    └──┬─────┬─────┬───────┘       └─────────┘
          ┌────────────┤     │     ├────────────────────┐
          ▼            ▼     ▼     ▼                    ▼
   ┌────────────┐ ┌────────┐ ┌──────────┐   ┌──────────────────┐
   │ SLIC Agent │ │  KA    │ │   DQA    │   │  CBP Domain      │
   │            │ │ Agent  │ │  Agent   │   │  Agent           │
   └──────┬─────┘ └───┬────┘ └────┬─────┘   └────────┬─────────┘
          ▼            ▼           ▼                   │
   ┌──────────┐ ┌──────────┐ ┌──────────┐             │
   │ SLIC DB  │ │Vector DB │ │ Trino DB │             │
   └──────────┘ └──────────┘ └──────────┘             │
                                                       ▼
                                            ┌──────────────────────┐
                                            │ Security Assessment  │
                                            │ Agent                │
                                            └──────────────────────┘

   ┌─────────────────────────────────────────────────────┐
   │ Supporting Agents (TBD):                            │
   │ Ambiguity Handler, Reflector, Context Pruner,       │
   │ Context Recovery                                    │
   └─────────────────────────────────────────────────────┘
```

### Components

**Orchestration Agents** (independent agents, same as Core/Domain Agents):
- **Semantic Router / Intent Classifier Agent** — self-contained agent that classifies user intent and routes to the Planner
- **Supervisor / Planner Agent** — self-contained agent that generates a task plan, delegates to Core/Domain Agents, and aggregates results
- **Memory** — Conversation history, Personalization, Self-Corrections, Agent memory

**Core Agents** (independent agents, each with MCP tool access):
| Agent                | Data Source                                               | Outputs                                   |
| -------------------- | --------------------------------------------------------- | ----------------------------------------- |
| **SLIC Agent**       | SLIC DB / Engine                                          | SLIC results, annotations                 |
| **Knowledge Agent**  | Vector DB (Human Annotations, Institutional Knowledge)    | RAG chunks, knowledge context             |
| **Data Query Agent** | Trino DB (Metadata Schema, Schema Cache, Runtime Context) — queries DB directly or via MCP tools with predefined queries | Structured query results, runtime context |

**Domain Agents** (independent agents, consume Core Agent outputs):
| Agent                                   | Scope                                                        |
| --------------------------------------- | ------------------------------------------------------------ |
| **Config Best Practice Agent**          | Assessment analysis, risk prioritization, corrective actions |
| **Security Assessment Agent**           | Security posture analysis, vulnerability assessment          |

**Supporting Agents** (all TBD, connected to Supervisor/Planner):
| Agent                 | Purpose                                         |
| --------------------- | ----------------------------------------------- |
| **Ambiguity Handler** | Clarification flows for ambiguous prompts       |
| **Reflector**         | Reflects on answer/task results for quality     |
| **Context Pruner**    | Conversation summary, semantic bloat mitigation |
| **Context Recovery**  | Recovers context for multi-turn scenarios       |

### Data Flow (Option 1)

In this model, the Planner **assembles state from individual agent responses** across service boundaries. There is no shared in-memory state object — each agent receives a request payload and returns a response payload. The Planner composes the full picture.

```
User prompt
    │
    ▼
┌──────────────────────────────────┐
│ Intent Classifier Agent          │  ← receives: user_prompt, context_kv
│ (service call)                   │  → returns: intent_class, entities[], confidence
└──────────────────┬───────────────┘
                   │ Planner receives intent response
                   ▼
┌──────────────────────────────────┐
│ Planner Agent                    │  ← reads: intent response + Memory
│ (builds task plan)               │  → emits: task plan with agent assignments
└──────────────────┬───────────────┘
                   │ Delegates tasks via service calls
          ┌────────┼────────┐
          ▼        ▼        ▼
    ┌──────────┐ ┌──────┐ ┌──────┐
    │ SLIC     │ │  KA  │ │ DQA  │   Each agent receives a task request
    │ Agent    │ │ Agent│ │Agent │   and returns task outputs
    └────┬─────┘ └──┬───┘ └──┬───┘
         │          │        │
         ▼          ▼        ▼        Planner collects responses
┌──────────────────────────────────┐
│ Planner composes context from    │  ← assembles: SLIC results + enterprise_context
│ Core Agent responses             │    + assessment_context into domain task request
└──────────────────┬───────────────┘
                   │ Delegates to Domain Agent
                   ▼
         ┌──────────────────┐
         │ Domain Agent     │  ← receives: composed context (assessment_context,
         │ (service call)   │    enterprise_context, SLIC results, intent)
         └────────┬─────────┘  → returns: findings[], summary, prioritized_risks[]
                  │
                  ▼
          Final response to user
```

**Key mechanism:** The Planner controls what data each agent sees. It selects which upstream outputs to include in each service call payload — providing natural context management (an agent only receives what the Planner explicitly sends).

**Data contract per service call:**

| Service Call | Request Payload | Response Payload |
|---|---|---|
| Intent Classifier | `user_prompt`, `context_kv` | `intent_class`, `meta_intent`, `entities[]`, `confidence`, `clarification_question` |
| SLIC Agent | Task definition, intent entities | `slic_results`, `annotations` |
| Knowledge Agent | Task definition, intent, conversation history | `enterprise_context`, `retrieval_query` |
| Data Query Agent | Task definition, intent entities, schema/ontology | `assessment_context` |
| Domain Agent | Task definition, composed upstream outputs (assessment_context, enterprise_context, SLIC results) | `findings[]`, `summary`, `prioritized_risks[]`, `chart_hints[]` |

### Key Characteristics

- **Every agent is a separate service/runtime** — this includes the Semantic Router/IC and Supervisor/Planner, not just Core and Domain Agents
- All agents can be independently deployed, scaled, and versioned
- Planner-to-agent communication crosses **process/network boundaries**
- A2A-compatible: all agents (including orchestration agents) can expose Agent Cards and communicate via JSON-RPC
- **Multiple Domain Agents**, each specialized for a vertical
- **SLIC is a separate Core Agent** with its own DB connection
- Domain Agents receive enriched context (RAG chunks, SLIC results, runtime data) composed by the Planner from Core Agent outputs
- **Context is Planner-controlled** — the Planner decides what enters each agent's request payload (selective composition)

---

## Option 2: Per-Domain Graph with LLM Nodes

### Description

Each domain is deployed as its **own LangGraph instance**. Within each graph, the Intent Classifier, Planner, Knowledge Agent, Data Query Agent, and Domain Task Agent exist as LLM-powered nodes sharing a single state object. There are no separate agent services — the graph's edges control routing, and each node makes its own LLM call with its own prompt/persona.

The key insight: rather than one platform-wide graph or one platform-wide multi-agent system, **each domain (CBP, SA) gets its own self-contained graph** with its own copy of the core nodes. The Intent Classifier is scoped to that domain.

### Architecture Diagram Summary

```
  ┌───────────────────────────────────────────────────────────────┐
  │  CBP Domain Graph Instance                                   │
  │                                                               │
  │  ┌──────────────────────────┐                                 │
  │  │ CBP Intent_Classifier    │                                 │
  │  │ Node                     │                                 │
  │  └────────────┬─────────────┘                                 │
  │               ▼                                               │
  │  ┌──────────────────────────┐       ┌─────────┐              │
  │  │ Supervisor / Planner Node│◄─────▶│ Memory  │              │
  │  └──┬──────────┬────────┬───┘       └─────────┘              │
  │     │          │        │                                     │
  │     ▼          ▼        ▼                                     │
  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────────────┐        │
  │  │ SLIC   │ │ KA     │ │ DQA    │ │ CBP Domain Task  │        │
  │  │ Node   │ │ Node   │ │ Node   │ │ Agent Node       │        │
  │  └───┬────┘ └───┬────┘ └───┬────┘ └──────────────────┘        │
  │      │          │          │                                   │
  │      ▼      ┌───┘          ▼      All nodes share one         │
  │  ┌────────┐ ▼        ┌────────┐   GraphState. No network     │
  │  │SLIC DB │ VectorDB │Trino DB│   boundaries.                 │
  │  └────────┘          └────────┘                                │
  └───────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────┐
  │  SA Domain Graph Instance (same structure, SA-scoped IC)     │
  └───────────────────────────────────────────────────────────────┘

  Supporting Nodes (TBD) included per graph:
  Reflector, Ambiguity Handler, Context Pruner, Context Recovery
```

### Components

Each domain graph instance contains the same node structure:

**Orchestration LLM Nodes** (same logical role as Option 1's orchestration agents, but implemented as graph nodes):
- **Domain-scoped Intent_Classifier Node** — LLM node that classifies intent within the scope of this domain (e.g., CBP-related intents only)
- **Supervisor / Planner Node** — LLM node that plans tasks and routes to other nodes via graph edges
- **Memory** — same store as Option 1 (Conversation, Personalization, Self-Corrections, Agent memory)

**Core Agent Nodes** (LLM nodes with MCP tool access, duplicated per graph):
| Node                      | Data Source                                               | Outputs                                   |
| ------------------------- | --------------------------------------------------------- | ----------------------------------------- |
| **SLIC Agent Node**       | SLIC DB / Engine                                          | SLIC results, annotations                 |
| **Knowledge Agent Node**  | Vector DB (Human Annotations, Institutional Knowledge)    | RAG chunks, institutional knowledge       |
| **Data Query Agent Node** | Trino DB (Metadata Schema, Schema Cache, Runtime Context) — queries DB directly or via MCP tools with predefined queries | Structured query results, runtime context |

**Domain Task Agent Node** (one per graph, domain-specific):
| Graph Instance     | Domain Task Node Scope                                                                        |
| ------------------ | --------------------------------------------------------------------------------------------- |
| CBP Graph          | Config Best Practice reasoning — assessment analysis, risk prioritization, corrective actions |
| SA Graph           | Security Assessment reasoning                                                                 |

**Supporting Nodes** (TBD, included per graph):
- Reflector, Ambiguity Handler, Context Pruner, Context Recovery

### Data Flow (Option 2)

In this model, all nodes within a domain graph **share a single GraphState object** in memory. Each node writes to its designated section(s) of the state directly — there are no network calls or serialization between nodes within a graph.

```
GraphState (shared in-memory object within one domain graph)
├── input
│   ├── user_prompt              ← External input
│   └── context_kv               ← Optional key-value context
├── intent
│   ├── intent_class             ← Intent Classifier Node writes
│   ├── meta_intent              ← Intent Classifier Node writes
│   ├── domain_details           ← Intent Classifier Node writes
│   ├── entities[]               ← Intent Classifier Node writes
│   ├── confidence               ← Intent Classifier Node writes
│   └── clarification_question   ← Intent Classifier Node writes (only when ambiguous)
├── plan
│   ├── tasks[]                  ← Planner Node writes
│   │   └── [each task]
│   │       ├── id, description, owner, depends_on[], status
│   │       ├── required_data[]      ← Planner Node declares
│   │       └── outputs              ← Executing node writes
│   │           ├── slic_results         (SLIC Agent Node)
│   │           ├── enterprise_context   (Knowledge Agent Node)
│   │           ├── retrieval_query      (Knowledge Agent Node)
│   │           ├── assessment_context   (Data Query Agent Node)
│   │           ├── findings[]           (Domain Task Agent Node)
│   │           ├── summary              (Domain Task Agent Node)
│   │           ├── prioritized_risks[]  (Domain Task Agent Node)
│   │           ├── asset_trend[]        (Domain Task Agent Node, trend mode)
│   │           └── chart_hints[]        (Domain Task Agent Node, optional)
│   └── routing[]                ← Planner Node writes
├── schema                       ← Pre-loaded (for Data Query Agent SQL path)
├── ontology                     ← Pre-loaded (for Data Query Agent SQL path)
├── conversation
│   └── history[]                ← Multi-turn context (future)
└── trace
    ├── node_run_order[]         ← All nodes append
    └── state_deltas[]           ← All nodes append
```

**Execution flow within a domain graph:**

```
User prompt enters graph
    │
    ▼
Intent Classifier Node ──▶ writes to GraphState.intent.*
    │
    ▼
Planner Node ──▶ reads GraphState.intent.*, writes GraphState.plan.*
    │
    ├──▶ SLIC Agent Node ──▶ reads GraphState.plan/intent, writes task.outputs.slic_results
    ├──▶ Knowledge Agent Node ──▶ reads GraphState.plan/intent, writes task.outputs.enterprise_context
    ├──▶ Data Query Agent Node ──▶ reads GraphState.plan/intent/schema/ontology, writes task.outputs.assessment_context
    │
    ▼
Domain Task Agent Node ──▶ reads all upstream task.outputs from GraphState
                        ──▶ writes task.outputs.findings[], summary, prioritized_risks[]
    │
    ▼
Graph returns final GraphState
```

**Key mechanism:** All nodes see the full GraphState — there is no Planner-mediated context selection. Each node reads what it needs directly from the shared object and writes to its designated section(s). Context pruning, if needed, must be done explicitly (e.g., by a Context Pruner node).

### Key Characteristics

- **One graph instance per domain** — each domain (CBP, SA) is deployed as its own self-contained LangGraph
- Within each graph, nodes share a single `GraphState` object — no network boundaries
- **SLIC is a separate LLM node** within each domain graph (same as Option 1, but as a node rather than a service)
- **Core nodes (SLIC, KA, DQA) are duplicated per graph** — each domain graph has its own SLIC, KA, and DQA nodes
- Each node makes its own **LLM call** with its own persona/prompt (these are LLM nodes, not simple functions)
- The Intent Classifier is **domain-scoped** — each graph has a classifier tuned to its domain's intent space
- The Domain Task Agent Node is **specialized per graph** — the CBP graph has a CBP Task Node, the SA graph has an SA Task Node, etc.
- **Context is fully shared** — all nodes see the entire GraphState; no Planner-mediated context selection

---

## Comparative Analysis

| Dimension                | **Option 1: Multi-Agent Services**                                            | **Option 2: Per-Domain Graph**                                                                              |
| ------------------------ | ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Deployment**           | Multiple services/containers — deploy, scale, version independently           | One graph deployment per domain — add a domain = deploy a new graph instance                                |
| **Communication**        | Cross-process (HTTP/JSON-RPC, A2A-ready)                                      | Intra-graph (shared state, function calls); cross-domain requires an external router                        |
| **Latency**              | Higher — network hops between Planner and each agent                          | Lower within a domain graph — no serialization, no network overhead                                         |
| **State Management**     | Planner assembles state from multiple agent responses; serialization required | Native shared `GraphState` per graph — all nodes read/write designated sections                              |
| **Model Flexibility**    | Each agent can use a different LLM (cheap for DQA, expensive for CBP)         | Each node can still make separate LLM calls, but typically shares the same model/provider within a graph    |
| **Scaling**              | Per-agent scaling (DQA gets more instances if Trino is slow)                  | Per-domain-graph scaling; can't scale KA independently of DQA within a graph                                |
| **Domain Extensibility** | Add a new Domain Agent as a new service — no changes to existing agents       | Add a new domain = deploy a new graph instance (isolated, no changes to existing graphs)                    |
| **A2A Compatibility**    | Native — each agent can expose an Agent Card                                  | Each graph instance could expose an Agent Card at the domain level, but internal nodes have no A2A identity |
| **Observability**        | Each agent call is a discrete span in traces                                  | Node execution is part of one graph trace per domain; LangGraph-level instrumentation                       |
| **Context Management**   | Controllable — Planner decides what enters context per agent                  | Shared within a graph — all nodes see the full state; context pruning must be explicit                      |
| **Failure Isolation**    | Agent failure doesn't crash others; Planner can retry or skip                 | Node failure can fail the domain graph; other domain graphs are unaffected                                  |
| **Team Ownership**       | Different teams can own different agents independently                        | One team per domain graph; domain teams operate independently of each other                                 |
| **SLIC Handling**        | Separate SLIC Agent with dedicated DB connection                              | Separate SLIC Node per graph with dedicated DB connection (duplicated per graph)                            |
| **Domain Agents**        | Shared Core Agents serve multiple Domain Agents                               | Core nodes (SLIC, KA, DQA) are duplicated in each domain graph — no sharing across domains                  |
| **Security Boundaries**  | Network-level isolation between agents; natural for multi-tenant              | Process-level isolation between domain graphs; nodes within a graph share a process                         |

---

## Key Architectural Differences

### 1. Agent Identity

**Option 1:** All agents — including the Semantic Router/IC and Supervisor/Planner — are **independent, self-contained services** with their own runtime, identity, and potentially their own Agent Card. They can be discovered, versioned, and replaced independently. Core Agents are **shared** — one KA serves all Domain Agents.

**Option 2:** All agents — including the Intent Classifier and Planner — are **LLM-powered nodes** within a domain-scoped graph. Each domain graph is an independent deployable unit, but the nodes inside it have no external identity. Core Agent nodes (KA, DQA) are **duplicated** per domain graph.

### 2. Domain Agent Strategy

**Option 1:** Multiple specialized Domain Agents (CBP, SA) as separate agents sharing a common set of Core Agents. Adding a new domain = deploying a new agent. Core Agents (KA, DQA) are shared across all domains.

**Option 2:** Each domain gets its **own complete graph instance** with a domain-specific Domain Task Agent Node and its own copies of KA/DQA. Adding a new domain = deploying a new graph instance. Core Agent logic is duplicated per domain graph.

### 2a. Core Agent Sharing — A Critical Difference

|                               | Option 1                      | Option 2                      |
| ----------------------------- | ----------------------------- | ----------------------------- |
| **KA instances**              | 1 (shared service)            | N (one per domain graph)      |
| **DQA instances**             | 1 (shared service)            | N (one per domain graph)      |
| **SLIC instances**            | 1 (shared service)            | N (one per domain graph)      |
| **Prompt/config consistency** | One source of truth           | Must keep N copies in sync    |
| **Resource efficiency**       | Core Agents serve all domains | Duplicated compute per domain |

### 3. SLIC Agent

**Option 1:** SLIC is a **dedicated Core Agent service** with its own connection to the SLIC DB/Engine.

**Option 2:** SLIC is a **dedicated LLM node** within each domain graph, with its own connection to the SLIC DB. Same responsibility as Option 1, but duplicated per graph rather than shared as a central service.

### 4. Supporting Agents

Both options include the same set of supporting capabilities (Ambiguity Handler, Reflector, Context Pruner, Context Recovery) — all TBD. In Option 1, these are shared services. In Option 2, they are nodes duplicated per domain graph.

### 5. Cross-Domain Routing

**Option 1:** The Semantic Router / Intent Classifier is a **single agent service** that acts as the entry point across all domains. A user asking a CBP question and an SA question in the same session stays within one orchestration layer.

**Option 2:** Each graph has its own domain-scoped Intent Classifier **node**. This means an **external routing layer is required** to direct the user's prompt to the correct domain graph in the first place. This external router is not shown in the diagram but is architecturally necessary — it sits above the per-domain graphs and must decide: "Is this a CBP question or an SA question?" before dispatching to the right graph instance.

---

## When to Choose

| Choose **Option 1** when...                                                | Choose **Option 2** when...                                             |
| -------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Core Agent logic must be shared across domains (single KA, single DQA)     | Each domain should be fully self-contained and independently deployable |
| You need per-agent scaling (e.g., DQA under heavy Trino load)              | Per-domain scaling is sufficient (scale the whole CBP graph)            |
| Cross-tenant or security-boundary isolation at the agent level is required | Domain-level isolation is sufficient (each graph is its own process)    |
| You want A2A interoperability with external agents at the agent level      | A2A is needed only at the domain-graph boundary (one card per domain)   |
| Core Agent prompts/config must stay in sync from a single source           | Duplicated core nodes per graph are acceptable                          |
| Each agent may need a different LLM or model version                       | A single model serves all reasoning tasks within a domain graph         |
| You want to independently deploy/rollback agents (KA separate from DQA)    | You prefer deploying/rolling back an entire domain as one unit          |
| Latency tolerance is higher (user accepts "thinking" time)                 | Latency within a domain is critical — every network hop counts          |

---

## Hybrid Consideration

The two options are not mutually exclusive. A pragmatic path:

1. **Start with Option 2** for the first domain (CBP) — single self-contained graph, fast iteration, simple deployment.
2. **Extract shared Core Agents into services (Option 1)** when a second domain graph ships and duplicating SLIC/KA/DQA becomes a maintenance burden.
3. **Domain graphs become thin** — they keep their domain-scoped IC, Planner, and Domain Task Node, but call shared KA/DQA services instead of hosting their own copies.

This produces a convergence: **Option 2 for domain isolation + Option 1 for core agent sharing** — each domain deploys its own graph, but Core Agents are centralized services consumed by all domain graphs.

This aligns with the **Migration Policy** in [Architectural Strategy v4](Architectural%20Strategy%20v4.md) §5.3: start contained, extract services when evidence (duplication cost, scaling needs, consistency requirements) justifies it.

### Open Questions

- **External domain router:** Option 2 requires something above the domain graphs to route incoming prompts to the right graph. Is this a simple classifier, a platform-level semantic router, or a gateway? This component is not shown in the diagram but is architecturally required.
- **Cross-domain queries:** What happens when a user prompt spans two domains (e.g., "How do my config best practice results affect my security posture?")? Option 1 handles this in the Planner; Option 2 would need cross-graph coordination.
- **Core node drift:** If each domain graph has its own SLIC/KA/DQA nodes, how do you keep their prompts, tool definitions, and configurations in sync across N graphs?

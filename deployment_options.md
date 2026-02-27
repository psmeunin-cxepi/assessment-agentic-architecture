# Deployment Options: Multi-Agent Services vs. Graph-Embedded Agents

**Purpose:** Compare two implementation approaches for the Assessment Agentic Architecture.
**Date:** 2026-02-27

---

## Overview

Both options share the same functional goal — route a user prompt through intent classification, planning, core data/knowledge retrieval, and domain-specific reasoning — but differ fundamentally in **how agents are deployed and interact at runtime**.

For the full list of agents, their roles, data sources, outputs, and skills, see [Logical Architecture](logical_architecture.md). This document focuses on deployment differences only.

|                        | **Option 1: Multi-Agent Services**            | **Option 2: Graph-Embedded Agents**                                                                             |
| ---------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Architecture style** | Distributed — agents are independent services | Per-domain — each domain gets its own LangGraph instance containing Intent Classifier, Planner, Knowledge Agent, Data Query Agent, and Domain Task nodes |
| **Communication**      | Inter-service (A2A, RemoteGraph)              | Intra-graph (shared state object, function calls)                                                               |
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
   ┌────────────┐ ┌─────────────────┐ ┌──────────────────┐   ┌──────────────────────────────┐
   │ SLIC Agent │ │ Knowledge       │ │ Data Query       │   │ Config Best Practice         │
   │            │ │ Agent           │ │ Agent            │   │ Domain Agent                 │
   └──────┬─────┘ └────────┬────────┘ └────────┬─────────┘   └──────────────┬───────────────┘
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
    ┌──────────┐ ┌─────────────────┐ ┌──────────────────┐
    │ SLIC     │ │ Knowledge       │ │ Data Query       │   Each agent receives a task request
    │ Agent    │ │ Agent           │ │ Agent            │   and returns task outputs
    └────┬─────┘ └────────┬────────┘ └────────┬─────────┘
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

- **Every agent is a separate service/runtime** — orchestration agents (Intent Classifier, Planner), core agents (SLIC, Knowledge Agent, Data Query Agent), and domain agents (Config Best Practice Agent, Security Assessment Agent) all run as independent services
- All agents can be independently deployed, scaled, and versioned
- **Agent-to-agent communication** is facilitated either through A2A or RemoteGraph. Both approaches require deployed API endpoints — A2A endpoints (JSON-RPC) or LangGraph Platform API endpoints, respectively
- **Core Agents are shared services** — a single Knowledge Agent, single Data Query Agent, and single SLIC Agent serve all Domain Agents, avoiding duplication
- **Context crosses service boundaries via explicit payload composition** — the Planner assembles each request payload with the relevant upstream outputs; an agent only sees what the Planner sends it

---

## Option 2: Graph-Embedded Agents

### Description

Each domain is deployed as its **own LangGraph instance**. Within each graph, the Intent Classifier, Planner, Knowledge Agent, Data Query Agent, and Domain Task Agent exist as LLM-powered nodes sharing a single state object. There are no separate agent services — the graph's edges control routing, and each node makes its own LLM call with its own prompt/persona.

The key insight: rather than one platform-wide graph or one platform-wide multi-agent system, **each domain (Config Best Practice, Security Assessment) gets its own self-contained graph** with its own copy of the core nodes. The Intent Classifier is scoped to that domain.

### Architecture Diagram Summary

```
  ┌──────────────────────────────────────────────────────────────────────────────────────────┐
  │  Config Best Practice Domain Graph Instance                                             │
  │                                                                                          │
  │  ┌──────────────────────────────────────────┐                                            │
  │  │ Config Best Practice Intent_Classifier   │                                            │
  │  │ Node                                     │                                            │
  │  └─────────────────────┬────────────────────┘                                            │
  │                        ▼                                                                  │
  │  ┌──────────────────────────────────────────┐       ┌─────────┐                          │
  │  │ Supervisor / Planner Node                │◄─────▶│ Memory  │                          │
  │  └──┬──────────────────┬────────────────┬───┘       └─────────┘                          │
  │     │                  │                │                                                 │
  │     ▼                  ▼                ▼                                                 │
  │  ┌────────┐ ┌─────────────────┐ ┌──────────────────┐ ┌──────────────────────────────┐    │
  │  │ SLIC   │ │ Knowledge       │ │ Data Query       │ │ Config Best Practice         │    │
  │  │ Node   │ │ Agent Node      │ │ Agent Node       │ │ Domain Task Agent Node       │    │
  │  └───┬────┘ └────────┬────────┘ └────────┬─────────┘ └──────────────────────────────┘    │
  │      │               │                   │                                                │
  │      ▼           ┌───┘                   ▼           All nodes share one                  │
  │  ┌────────┐      ▼              ┌────────┐           GraphState. No network               │
  │  │SLIC DB │   VectorDB          │Trino DB│           boundaries.                          │
  │  └────────┘                     └────────┘                                                │
  └──────────────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────────────────┐
  │  Security Assessment Domain Graph Instance (same structure, Security Assessment-scoped   │
  │  Intent Classifier)                                                                      │
  └──────────────────────────────────────────────────────────────────────────────────────────┘

  Supporting Nodes (TBD) included per graph:
  Reflector, Ambiguity Handler, Context Pruner, Context Recovery
```

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

- **One graph instance per domain** — each domain (Config Best Practice, Security Assessment) is deployed as its own self-contained LangGraph
- Within each graph, nodes share a single `GraphState` object — no network boundaries
- **Core nodes (SLIC, Knowledge Agent, Data Query Agent) are duplicated per graph** — each domain graph has its own SLIC, Knowledge Agent, and Data Query Agent nodes
- The Intent Classifier is **domain-scoped** — each graph has a classifier tuned to its domain's intent space
- **Context is fully shared** — all nodes see the entire GraphState; no Planner-mediated context selection
- **Scaling unit is the entire domain graph** — individual nodes (e.g., Knowledge Agent, Data Query Agent) cannot be scaled independently within a graph
- **An external routing layer is required** — since each domain has its own graph with a domain-scoped Intent Classifier, an external component must route incoming prompts to the correct domain graph
- **Sub-graph variation:** Core Agents and the Domain Task Agent can each be built as sub-graphs rather than flat nodes, giving each agent its own internal graph structure while remaining invocable as a single node from the Planner's perspective
- Supporting Nodes (Reflector, Ambiguity Handler, Context Pruner, Context Recovery) are included per graph — all TBD

---

## Comparative Analysis

| Dimension                  | **Option 1: Multi-Agent Services**                                            | **Option 2: Graph-Embedded Agents**                                                                         |
| -------------------------- | ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Deployment**             | Multiple services/containers — deploy, scale, version independently           | One graph deployment per domain — add a domain = deploy a new graph instance                                |
| **Communication**          | Cross-process (A2A, RemoteGraph)                                              | Intra-graph (shared state, function calls); cross-domain requires an external router                        |
| **Latency**                | Higher — network hops between Planner and each agent                          | Lower within a domain graph — no serialization, no network overhead                                         |
| **State Management**       | Each agent manages its own internal state in isolation; the Planner assembles cross-agent context by composing upstream outputs into request payloads | Native shared `GraphState` per graph — all nodes read/write designated sections                              |
| **Context Management**     | Each agent manages its own context window independently; cross-agent context is composed explicitly by the Planner into request payloads | All nodes see the full GraphState; context window is managed at the graph and node level                     |
| **Scaling**                | Per-agent scaling                                                             | Per-domain-graph scaling                                                                                    |
| **Core Agent Sharing**     | Shared Core Agents serve all Domain Agents — single Knowledge Agent, single Data Query Agent, single SLIC | Core nodes (SLIC, Knowledge Agent, Data Query Agent) are duplicated in each domain graph |
| **Domain Extensibility**   | Add a new Domain Agent as a new service                                       | Add a new domain graph instance                                                                             |
| **External Routing**       | Single Intent Classifier service handles all domains                          | An external routing layer is required above the domain graphs to dispatch prompts to the correct graph      |
| **A2A Compatibility**      | Native — each agent can expose an Agent Card                                  | Each graph instance could expose an Agent Card at the domain level, but internal nodes have no A2A identity |
| **Observability**          | Each agent call is a discrete span in traces                                  | Node execution is part of one graph trace per domain; LangGraph-level instrumentation                       |
| **Failure Isolation**      | Agent failure doesn't crash others; Planner can retry or skip                 | Node failure can fail the domain graph; other domain graphs are unaffected                                  |
| **Deployment Complexity**  | Multiple services to deploy, monitor, and maintain with API endpoints         | One deployable unit per domain, but core agent logic is duplicated and must be kept in sync across graphs    |
| **Team Ownership**         | Different teams can own different agents independently                        | One team per domain graph; domain teams operate independently of each other                                 |
| **Tenant Isolation**       | Tenant identity validated at every service boundary; more enforcement points to audit | Tenant isolation at the graph-instance level; must ensure GraphState and checkpoints never cross tenant boundaries |

---

## Proposed Migration Path

This path phases the deployment from a single self-contained graph to a service-backed architecture as domain count grows.

1. **Start with Option 2** for the first domain (Config Best Practice) — single self-contained graph, fast iteration, simple deployment.
2. **Extract Core Agents into shared services** when a second domain graph ships and duplicating Knowledge Agent, Data Query Agent, and SLIC becomes a maintenance burden (prompt drift, duplicated compute, configuration inconsistency across graphs).
3. **Domain graphs become thin** — they retain their Planner and Domain Agent nodes but delegate core capabilities to the shared services. The Intent Classifier migrates to a single shared service that routes across all domains.

This produces a convergence: **Option 2 for domain isolation + Option 1 for core agent sharing** — each domain deploys its own graph, but Core Agents are centralized services consumed by all domain graphs.

This aligns with the **Migration Policy** in [Architectural Strategy v4](Architectural%20Strategy%20v4.md) §5.3: start contained, extract services when evidence (duplication cost, scaling needs, consistency requirements) justifies it.

### Open Questions

- **External domain router:** Option 2 requires something above the domain graphs to route incoming prompts to the right graph. Is this a simple classifier, a platform-level semantic router, or a gateway? This component is not shown in the diagram but is architecturally required.
- **Cross-domain queries:** What happens when a user prompt spans two domains (e.g., "How do my config best practice results affect my security posture?")? Option 1 handles this in the Planner; Option 2 would need cross-graph coordination.

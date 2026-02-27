# Architectural Strategy: Agentic Ecosystem Design

**Subject:** Service-Oriented (SOA) vs. Library-Oriented (LOA) Architectures
**Goal:** Defining a scalable, extensible framework for Core and Domain Agents.
**Version:** 4.0
**Last Updated:** 2026-02-26

---

## 1. Executive Summary

To build an agentic system that scales, we must decide how **Domain Agents** (the specialists, e.g., Config Best Practice, Security Assessment) interact with **Core Agents** (the shared utilities, e.g., Knowledge, Data Query). We categorize these interactions into two primary patterns:

| Pattern | Model | One-liner |
| --- | --- | --- |
| **Service-Oriented (SOA)** | "The Delegator" | Domain Agent sends a request to a Core Agent over a network protocol; receives the result. |
| **Library-Oriented (LOA)** | "The Expert" | Domain Agent imports a Core Agent's skills into its own runtime; executes locally. |

Neither pattern is universally superior. This document provides the criteria for choosing between them and defines how both coexist in a hybrid architecture. The default posture is **service-first (SOA)**, with LOA available as an optimization path once specific criteria are met (see §5.3).

**Scope note:** This document initially focuses on how Domain Agents interact with Core Agents — the most immediate architectural decision. However, the SOA/LOA choice has broader implications that will need to be addressed as the system evolves:

- **Domain ↔ Domain:** Can one Domain Agent inherit another's skills? (e.g., CBP importing Security Assessment reasoning, or vice versa.)
- **Orchestrator role:** Does the Planner remain a central coordinator, or could a Domain Agent that inherits enough skills subsume that role?
- **Composition depth:** If Domain Agent A inherits Domain Agent B's skills, and B itself inherits Core Agent C's skills, how deep does the inheritance chain go before context budget or coherence degrades?

These are open questions for future phases. Phase 1 treats Domain Agents as independent consumers of Core services, with the Planner as the orchestration layer.

---

## 2. Defining the Paradigms

### A. Service-Oriented Architecture (SOA): The "Delegator"

Capabilities are treated as independent, live services. A Domain Agent does not possess core logic; instead, it sends a request to a specialist via a network protocol (e.g., [A2A](https://github.com/a2aproject/A2A)).

* **Mechanism:** Agent-to-Agent communication (JSON-RPC over HTTP, SSE streaming, or message queues).
* **Analogy:** A General Contractor (Domain Agent) hiring a specialized Electrician (Core Agent) to handle the wiring. The Contractor doesn't need to know *how* to wire; they just need to know *who* to call.
* **Best For:** Compute-heavy tasks, cross-tenant operations, tasks requiring different LLM models, or when output size is large/unpredictable.

### B. Library-Oriented Architecture (LOA): The "Expert"

Capabilities are treated as modular Skill Packages using the [Agent Skills open format](https://agentskills.io/) (`SKILL.md`). A Domain Agent **imports** the expertise, instructions, and tools of the Core Agent directly into its own runtime.

* **Mechanism:** Dynamic prompt injection and local tool execution.
* **Analogy:** An Expert Artisan who owns a library of technical manuals. When they need to perform "wiring," they pull the specific manual from the shelf and use their own hands to execute.
* **Best For:** Procedural logic, context acquisition within a reasoning chain, and low-latency operations within a single trust zone.

> **Note on Agent Skills:** The `SKILL.md` format was originally developed by Anthropic and released as an open format. It has been adopted by a growing ecosystem of agent products (Cursor, Claude Code, GitHub, Roo Code, Amp, OpenHands, and others). It is not yet a formal standard body specification (e.g., IETF/W3C) but has broad industry traction. See [agentskills.io](https://agentskills.io/) for the specification and [GitHub](https://github.com/agentskills/agentskills) for the reference library.

---

## 3. Comparative Analysis

| Feature | **Service-Oriented (SOA)** | **Library-Oriented (LOA)** |
| --- | --- | --- |
| **Performance** | **Higher latency:** Subject to network round-trips and serialization overhead. | **Lower latency:** Zero-network execution; logic runs in the same process. |
| **Context Management** | **Controllable:** The caller *can* limit context to the final result only, but streaming events, multi-turn clarifications, and inlined artifacts may still accumulate depending on implementation choices. The advantage is having *control surfaces* that LOA lacks (poll vs. stream, artifact-by-reference vs. inline, response summarization). | **Inherent risk:** The imported skill's full instructions, tool calls, and intermediate results all execute inside the host agent's context window. No escape hatch exists to shed intermediate context. |
| **State Consistency** | **Challenging:** Requires passing state across a network boundary. Both modes must produce equivalent outputs into the same planner/task state schema — see §6 Compatibility Contract. | **Native:** Uses a single, unified GraphState throughout the task. No serialization boundary. |
| **Model Flexibility** | **Superior:** Each service can use a purpose-fit model (e.g., a cheap model for data retrieval, an expensive model for reasoning). | **Limited:** Typically tied to the host agent's primary model. The imported skill runs under the host's LLM. |
| **Maintenance** | **Decoupled:** Update the Knowledge Service without touching Domain Agents. Version contracts independently. | **Coupled:** Skills are portable folders, but updates may require re-validating every agent that imports them. |
| **Security** | **Granular:** Strict API-level permissions, data masking, and tenant-boundary enforcement at the network layer. | **Broad:** Skills typically inherit full access to the host agent's memory, tools, and credentials. Tenant isolation requires explicit guardrails. |
| **Observability** | **Structured:** Each service call is a discrete span in distributed traces. Easy to attribute latency, cost, and errors. | **Blended:** Skill execution is interleaved with the host agent's reasoning. Harder to isolate for debugging or cost attribution. |

---

## 4. Implementation Layers

Agent Skills and A2A operate at **different, complementary layers**:

| Layer | Technology | Role |
| --- | --- | --- |
| **Skill packaging** | [Agent Skills](https://agentskills.io/) (`SKILL.md`) | Defines *what* a skill contains: instructions, scripts, resources, tool definitions. |
| **Inter-agent communication** | [A2A Protocol](https://github.com/a2aproject/A2A) | Defines *how* agents discover each other and exchange messages/tasks over a network. |

An agent can use Agent Skills **without** A2A (pure LOA — local file-system skill loading), and can use A2A **without** Agent Skills (pure SOA — remote service delegation). The most flexible systems combine both.

### 4.1 Contextual Inheritance (The LOA Mechanism)

"Contextual Inheritance" allows a Domain Agent to adopt the persona and tools of a Core Agent dynamically, without leaving its reasoning loop.

1. **Identification:** The Domain Agent (e.g., Config Best Practice Agent) identifies a need for a core capability (e.g., knowledge retrieval).
2. **Skill Discovery:** It locates and reads the relevant `SKILL.md` manifest.
3. **Instructional Inheritance:** The skill's instructions are injected into the agent's system prompt.
4. **Tool Adoption:** The agent registers the tools defined in the skill package (e.g., vector search, document retrieval).
5. **Local Execution:** The agent performs the capability itself. Context is acquired and used immediately without a secondary agent hand-off.

**Example Skill Folder Structure:**

```text
/skills
  /core-knowledge/             # Core Skill
    ├── SKILL.md               # Retrieval instructions & procedures
    └── tools/                 # VectorDB search, document fetch
  /core-data-query/            # Core Skill
    ├── SKILL.md               # MCP tool invocation instructions
    └── tools/                 # assessment_analysis, comparison, issue_tracking
  /config-best-practice/       # Domain Skill (LOA variant)
    ├── SKILL.md               # CBP reasoning procedures & chart_hints logic
    └── tools/                 # Domain-specific analysis tools
```

**Multi-Skill Inheritance Example:**

When a Domain Agent needs to answer a complex query that requires both knowledge retrieval and data analysis, it doesn't call two separate services — it inherits two skills:

1. **Inherit Core:** Loads `core-knowledge/SKILL.md` to gain retrieval capabilities.
2. **Inherit Domain Data:** Loads `core-data-query/SKILL.md` to gain MCP tool access.
3. **Execute:** The agent now has the combined "brain" of both skills, executing the task in a single, unified state.

> **Context budget warning:** Each imported skill adds instructions and tool definitions to the host agent's context window. Importing multiple skills simultaneously requires careful budgeting — see §5.1 Context Budget criterion.

### 4.2 The Role of A2A (The Connectivity Tissue)

A2A provides the protocol layer for **discovery and delegation**:

* **Discovery:** Agents publish Agent Cards (via well-known URIs or registries) that advertise their capabilities and skills.
* **Delegation:** For tasks that require separate runtimes — different models, compute isolation, or security boundaries — the Domain Agent delegates via A2A's `tasks/send` or `tasks/sendSubscribe` methods (JSON-RPC).
* **Fallback escalation:** An agent that initially loads a skill locally (LOA) can fall back to SOA delegation if the task exceeds local resource limits or requires cross-tenant isolation.

> **Clarification on "Skill Transfer via A2A":** The A2A protocol does not define a standardized mechanism for transferring skill packages between agents. A2A covers message/task exchange and agent discovery — not package distribution. Skill acquisition would need to be implemented out-of-band (e.g., a repository pull, package registry, or file-system mount) or potentially as a future [A2A profile extension](https://google.github.io/A2A/#/documentation?id=extensions).

---

## 5. Decision Framework

### 5.1 Decision Matrix

Use the table below to determine if a new capability should be a **Skill (LOA)** or a **Service (SOA)**.

| Criteria | **Choose Skill (LOA)** | **Choose Service (SOA)** |
| --- | --- | --- |
| **Primary Goal** | Providing data, context, or procedures to enrich reasoning. | Compute-heavy processing (ETL, large-scale data queries, scraping). |
| **Latency Tolerance** | Low — needs to be near-instant within the reasoning loop. | Higher — user accepts a "thinking" phase; async is acceptable. |
| **Model Requirements** | Can run on the Domain Agent's model. | Requires a specialized, cheaper, or different LLM. |
| **Workflow Integration** | Part of a larger reasoning chain — intermediate results feed the next thought. | A standalone, isolated task with a well-defined output contract. |
| **Security Boundary** | Same tenant, same trust zone, same credentials. | Crosses tenant boundaries, requires data masking, or enforces API-level access control. |
| **Context Budget** | Skill's instructions + tool outputs fit within the host agent's context window. | Output is large or unpredictable; caller needs to control what enters context. |

### 5.2 Security Decision Gate

Before choosing LOA for any capability, apply this hard gate:

> **Can this capability cross a tenant or security boundary?**
> - **Yes →** SOA is mandatory. Network-level isolation, API permissions, and data masking are required.
> - **No →** LOA is permitted. Proceed with the remaining criteria in §5.1.

This gate is non-negotiable. No amount of latency or context-window benefit justifies importing a skill that handles cross-tenant data into a single agent's runtime.

### 5.3 Migration Policy

The default posture is **service-first (SOA)**. A capability may be promoted to LOA when all four conditions are met:

1. **Latency evidence:** Measured round-trip latency via SOA demonstrably degrades user experience below SLO thresholds.
2. **Context-fit evidence:** The skill's instructions and typical tool outputs fit within the host agent's context budget without exceeding 30% of the available window.
3. **Security review:** The capability has been verified to operate within a single trust zone with no cross-tenant data exposure.
4. **Compatibility verification:** The LOA version produces output structurally equivalent to the SOA version (see §6).

Until all four criteria are met, the capability remains a service.

---

## 6. Compatibility Contract

Both SOA and LOA modes must be **output-equivalent**. Regardless of which path executes a capability, the result written into the planner's task state must conform to the same schema.

```
┌─────────────────────────────┐
│   Domain Agent (caller)     │
│                             │
│   ┌──── LOA Path ────┐     │
│   │ Load SKILL.md    │     │
│   │ Execute locally  │──┐  │
│   └──────────────────┘  │  │
│                         ▼  │
│              ┌─────────────┐│
│              │ task.outputs ││  ← same schema
│              └─────────────┘│
│   ┌──── SOA Path ────┐  ▲  │
│   │ A2A SendMessage  │  │  │
│   │ Receive result   │──┘  │
│   └──────────────────┘     │
└─────────────────────────────┘
```

**Why this matters:** The Planner and any downstream consumers must not need to know *how* a capability was executed. They read from `STATE.plan.tasks[id].outputs` regardless. If a capability is migrated from SOA to LOA (or vice versa), nothing upstream or downstream should change.

---

## 7. Project Recommendation

For the Assessment Agentic Architecture project specifically:

| Capability | Recommended Mode | Rationale |
| --- | --- | --- |
| **Knowledge Agent** | SOA (default) | May serve multiple Domain Agents; independent scaling; RAG pipeline benefits from dedicated model tuning. Candidate for LOA promotion if latency becomes critical. |
| **Data Query Agent** | SOA (default) | Executes MCP tool calls against backend APIs; output size is variable and often large; benefits from context control (summary vs. full payload). |
| **Config Best Practice Agent** | Domain Agent (consumes SOA services) | Orchestrates KA and DQA results into domain-specific reasoning. |
| **Security Assessment Agent** | Domain Agent (consumes SOA services) | Same pattern as CBP — orchestrates core services for security domain. |

This is the Phase 1 posture. LOA optimizations can be evaluated per §5.3 once latency and context-fit data is available from production traces.

---

## Appendix A — LOA Evolution Path

While Phase 1 is service-first, the architecture is designed to **evolve toward selective LOA** where evidence supports it. The following illustrates what a mature LOA deployment could look like:

### From "Core" to "Universal" Skills

As the system matures and context windows grow, capabilities that start as SOA services may graduate to importable skills:

```text
Phase 1 (SOA-first)                    Phase N (Selective LOA)
┌──────────┐   A2A    ┌──────────┐     ┌──────────────────────────┐
│ CBP Agent│ ──────── │ KA Svc   │     │ CBP Agent                │
│          │   A2A    ├──────────┤     │  ├─ core-knowledge (LOA) │
│          │ ──────── │ DQA Svc  │     │  └─ core-data-query (SOA)│◄── large outputs stay SOA
└──────────┘          └──────────┘     └──────────────────────────┘
```

In Phase N, Core Knowledge has been promoted to LOA (latency-critical, small output, single-tenant), while Data Query remains SOA (variable output size, benefits from context control).

### Multi-Skill Inheritance at Scale

In a fully evolved system, a Domain Agent could inherit multiple skills for complex multi-step tasks:

1. **Inherit `core-knowledge`** — gain retrieval capabilities
2. **Inherit `assessment-rules`** — gain best-practice rule evaluation logic
3. **Delegate to `data-query` (SOA)** — large data fetch stays remote
4. **Execute reasoning** — combined skill context produces the final answer in unified state

This hybrid pattern — LOA for lightweight context, SOA for heavy data — is the target architecture once migration criteria (§5.3) are validated.

---

## Appendix B — Sources

- A2A Protocol: [github.com/a2aproject/A2A](https://github.com/a2aproject/A2A) — RC v1.0
- Agent Skills format: [agentskills.io](https://agentskills.io/) — open format by Anthropic
- Agent Skills GitHub: [github.com/agentskills/agentskills](https://github.com/agentskills/agentskills)
- LangGraph: [langchain-ai.github.io/langgraph](https://langchain-ai.github.io/langgraph/) — StateGraph orchestration

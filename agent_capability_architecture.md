# Agent Capability Architecture: From Static Composition to Dynamic Skills

**Purpose:** Define how agents in the Assessment Agentic Architecture acquire and compose their capabilities — from today's static, hand-wired approach to a future where capabilities are packaged as portable Agent Skills and loaded dynamically from a registry.

**Date:** 2026-02-27

**Relationship to other documents:**
- [logical_architecture.md](logical_architecture.md) — defines *what* agents exist, their skills, relationships, and data flows
- [deployment_options.md](deployment_options.md) — defines *how* agents are deployed and communicate at runtime
- This document defines *how agents acquire their capabilities* — orthogonal to both logical identity and deployment model

---

## 1. Executive Summary

Every agent in the architecture — whether it runs as an independent service (Option 1) or a graph node (Option 2) — must acquire capabilities: prompts, tools, context sources, reasoning patterns, and output contracts. Today these are **statically composed**: hand-wired into each agent at design time.

The **Agent Skills** open format offers a path to **dynamic composition**: capabilities are packaged as portable skill packages — containing instructions, scripts, references, and assets — that agents can discover and load from a registry at runtime. An agent becomes a **runtime that consumes skills** rather than a monolith with hardcoded capabilities.

| Approach | Model | One-liner |
| --- | --- | --- |
| **Static Composition** | "Hand-wired" | Agent capabilities are defined at design time — prompts, tools, and patterns baked into code or configuration. |
| **Dynamic Skills** | "Registry-loaded" | Agent capabilities are packaged as portable skill packages and loaded from a registry at runtime. |

The default posture is **static composition first**, with dynamic skills available as an evolution path once the skill catalog, registry infrastructure, and runtime validation criteria are in place (see §5).

---

## 2. Static Composition (Current State)

Today, each agent is built by assembling four building blocks at design time:

| Building Block | What It Is | Example |
| --- | --- | --- |
| **System Prompt** | The agent's persona, rules, constraints, and reasoning instructions | Config Best Practice Agent's assessment procedure, output format rules |
| **Tools** | Functions the agent can invoke to interact with external systems | MCP tools (`assessment_analysis`, `comparison`), vector search, SQL query |
| **Context Sources** | Data the agent receives as input to reason over | `assessment_context` from Data Query Agent, `enterprise_context` from Knowledge Agent |
| **Design Patterns** | Agentic reasoning structures: chain-of-thought, reflection, planning | ReAct loops, structured output generation, chart_hints composition |

### How Static Composition Works

```
┌─────────────────────────────────────────────────────────┐
│                 Agent (at design time)                   │
│                                                         │
│   System Prompt ──────── hardcoded in agent contract    │
│   Tools ─────────────── registered in agent code        │
│   Context Sources ───── wired by Planner routing        │
│   Design Patterns ───── embedded in prompt + code       │
│                                                         │
│   Result: a fixed-capability agent                      │
└─────────────────────────────────────────────────────────┘
```

### What This Looks Like in Practice

**Graph-Embedded Agents (Option 2) are the clearest example of static composition.** A LangGraph graph definition is entirely hand-wired at build time: nodes are defined with their prompts and tools, edges and conditional routing are coded explicitly, and the shared `GraphState` schema is fixed in advance. Every capability the graph possesses — which agents run, in what order, with what tools — is determined by the graph's source code, not discovered at runtime.

### Strengths

- **Simple to reason about** — everything an agent does is visible in its contract and code
- **No runtime discovery overhead** — all capabilities are known at deploy time
- **Easy to test** — fixed inputs produce deterministic behavior
- **Sufficient for Phase 1** — the initial agent count (6 agents, ~12 skills) is manageable

### Limitations

- **Adding a capability requires code changes** — new tools, prompts, or patterns mean updating agent code and redeploying
- **No sharing mechanism** — if two agents need the same reasoning pattern, it must be duplicated (or extracted into a shared library manually)
- **Scaling the catalog is manual** — as the number of agents and skills grows, keeping prompts, tool definitions, and patterns in sync across agents becomes a maintenance burden
- **No runtime adaptation** — an agent cannot acquire a new capability without being redeployed

---

## 3. Dynamic Skills (Evolution Path)

The **Agent Skills** open format provides a standard way to package agent capabilities as portable, discoverable units that can be loaded at runtime.

> **Note on Agent Skills:** The `SKILL.md` format was originally developed by Anthropic and released as an open format. It has been adopted by a growing ecosystem of agent products (Cursor, Claude Code, GitHub, Roo Code, Amp, OpenHands, and others). It is not yet a formal standards-body specification (e.g., IETF/W3C) but has broad industry traction. The format is **actively evolving** — features like `allowed-tools` are still experimental, and the spec does not yet cover areas like MCP tool bundling or inter-skill dependencies. Expect the format to expand as the ecosystem matures. See [agentskills.io](https://agentskills.io/) for the current specification and [GitHub](https://github.com/agentskills/agentskills) for the reference library.

### What a Skill Package Contains

A skill is a **folder** containing a required `SKILL.md` file plus optional directories. The following is an example — the actual contents can vary and are not limited to what is shown:

```text
my-skill/
├── SKILL.md          # Required: YAML frontmatter (name, description) + markdown instructions
├── scripts/          # Optional: executable code (Python, Bash, JavaScript)
├── references/       # Optional: additional documentation, loaded on demand
└── assets/           # Optional: templates, schemas, lookup tables, static resources
```

The spec uses **progressive disclosure** to manage context efficiently:
1. **Discovery** (~100 tokens) — only `name` and `description` are loaded at startup for all skills
2. **Activation** (<5000 tokens recommended) — the full `SKILL.md` body is loaded when a task matches
3. **Execution** (as needed) — `scripts/`, `references/`, and `assets/` are loaded only when required

The skill package maps to the same building blocks agents use today — but packaged portably:

| Building Block | Static Composition | Skill Package |
| --- | --- | --- |
| **System Prompt** | Hardcoded in agent contract | `SKILL.md` body — markdown instructions injected into the host agent's prompt at runtime |
| **Tools** | Registered in agent code | `scripts/` directory + `allowed-tools` frontmatter field (experimental) — the host agent adopts the skill's executable code and pre-approved tool list |
| **Context Sources** | Wired by Planner routing | Declared in `SKILL.md` instructions and `references/` — the host agent or registry resolves them |
| **Design Patterns** | Embedded in prompt + code | Encoded in the skill's instructions and script orchestration logic |

> **Spec vs. project extensions:** The current Agent Skills spec defines the folder structure, frontmatter fields, and progressive disclosure model described above. It does **not** yet define a standard way to bundle MCP tool definitions, declare inter-skill dependencies, or specify structured input/output schemas within a skill. Our architecture may extend the format in these areas as needs arise — any extensions will be documented as project-specific additions distinct from the base spec.

### Contextual Inheritance: How Dynamic Loading Works

When an agent loads a skill at runtime, it performs **contextual inheritance** — the skill's instructions and tools are merged into the agent's own runtime context:

1. **Discovery** — the agent queries a skill registry for a required capability (e.g., "knowledge retrieval")
2. **Manifest read** — the agent reads the skill's `SKILL.md` manifest to understand its instructions, tools, inputs, and outputs
3. **Instructional inheritance** — the skill's instructions are injected into the agent's system prompt
4. **Tool adoption** — the agent registers executable scripts and pre-approved tools from the skill package
5. **Local execution** — the agent executes the capability itself, within its own runtime and context window

```
┌─────────────────────────────────────────────────────────────┐
│                 Agent (at runtime)                           │
│                                                             │
│   Own system prompt                                         │
│      + injected skill instructions  ◄── from SKILL.md body │
│   Own tools                                                 │
│      + adopted skill scripts/tools  ◄── from scripts/      │
│   References loaded on demand       ◄── from references/   │
│                                                             │
│   Result: dynamically-composed agent                        │
└─────────────────────────────────────────────────────────────┘
```

### Example: Skill Folder Structure

```text
/skills
  /core-knowledge/               # Core Skill
    ├── SKILL.md                 # Retrieval instructions and procedures
    ├── scripts/                 # Vector DB search, document fetch scripts
    └── references/              # Retrieval strategy docs, chunking guidelines
  /core-data-query/              # Core Skill
    ├── SKILL.md                 # MCP tool invocation instructions
    ├── scripts/                 # Query execution, result normalization
    └── assets/                  # Schema definitions, ontology files
  /config-best-practice/         # Domain Skill
    ├── SKILL.md                 # Assessment reasoning, chart_hints logic
    ├── scripts/                 # Domain-specific analysis scripts
    └── references/              # Best-practice rule references
```

### Multi-Skill Inheritance

A Domain Agent can load multiple skills simultaneously to handle complex queries:

1. **Load `core-knowledge`** — gains retrieval capabilities
2. **Load `core-data-query`** — gains MCP tool access
3. **Execute** — the agent reasons with the combined context of both skills in a single pass

> **Context budget warning:** Each imported skill adds instructions and tool definitions to the host agent's context window. Importing multiple skills simultaneously requires careful budgeting — see §5 Migration Criteria.

### Deployment Independence

Dynamic skills apply equally to both deployment options:

| | Option 1: Multi-Agent Services | Option 2: Graph-Embedded Agents |
| --- | --- | --- |
| **Who loads skills?** | Each agent service loads skills from the registry into its own runtime | Each graph node loads skills from the registry into its execution context |
| **Registry access** | Over network (skill registry is a service) | Over network or local file system (skills bundled with graph or fetched at startup) |
| **Scaling impact** | Skills loaded per-agent — shared services still serve all domains | Skills loaded per-node — duplicated per graph, but loaded dynamically rather than hardcoded |

---

## 4. Comparison: Static vs Dynamic

| Dimension | **Static Composition** | **Dynamic Skills** |
| --- | --- | --- |
| **Performance** | No runtime discovery overhead | Skill loading adds startup or invocation latency |
| **Context Management** | Predictable — all context is known at design time | Risk of context bloat — each loaded skill consumes context window budget |
| **Model Flexibility** | Each agent can use its own model regardless | Loaded skills execute under the host agent's model |
| **Maintenance** | Update per-agent — change code, redeploy | Update the skill package once — all consuming agents pick up the change |
| **Capability Sharing** | Manual duplication or shared libraries | Native — any agent can load any skill from the registry |
| **Versioning** | Per-agent deployment versioning | Per-skill versioning — agents can pin to specific skill versions |
| **Security** | Access control at the agent level | Loaded skills inherit the host agent's permissions — tenant isolation requires explicit guardrails |
| **Observability** | Each agent call is a discrete trace span | Skill execution is interleaved with host agent reasoning — harder to isolate |
| **Testability** | Fixed inputs/outputs, straightforward | Must test skill behavior across different host agents and combinations |
| **Runtime Adaptation** | Requires redeployment | Agent can acquire new capabilities without redeployment |

---

## 5. Migration Criteria: Static to Dynamic

The default posture is **static composition first**. A capability may be promoted to a dynamic skill when all of the following conditions are met:

### 5.1 Readiness Criteria

| Criterion | Threshold | Rationale |
| --- | --- | --- |
| **Skill catalog exists** | A skill registry is deployed and operational | Dynamic loading requires infrastructure to discover and fetch skills |
| **Packaging complete** | The capability's prompts, tools, and instructions are packaged in a valid `SKILL.md` | Cannot load what isn't packaged |
| **Context-fit verified** | The skill's instructions + typical tool outputs fit within 30% of the host agent's context window | Prevents context bloat from crowding out the agent's own reasoning |
| **Security reviewed** | The skill operates within a single trust zone with no cross-tenant data exposure | Skills inherit the host agent's permissions — cross-tenant capabilities must remain service-based |
| **Output equivalence tested** | The dynamic-skill version produces structurally identical outputs to the static version | See §6 Compatibility Contract |

### 5.2 Security Decision Gate

Before promoting any capability to a dynamic skill, apply this hard gate:

> **Can this capability cross a tenant or security boundary?**
> - **Yes →** The capability must remain a service (static composition with service-based invocation). Network-level isolation, API permissions, and data masking are required.
> - **No →** Dynamic skill loading is permitted. Proceed with the remaining criteria in §5.1.

This gate is non-negotiable. No amount of latency or context-window benefit justifies importing a skill that handles cross-tenant data into a single agent's runtime.

### 5.3 Migration Policy

A capability transitions from static to dynamic when:

1. **Duplication evidence:** The same capability (prompts, tools, patterns) is manually maintained in 2+ agents
2. **Maintenance burden:** Keeping duplicated copies in sync has caused incidents or measurable overhead
3. **All readiness criteria met:** Per §5.1
4. **Latency acceptable:** Dynamic skill loading does not degrade user experience below Service Level Objective thresholds

Until these conditions are met, the capability remains statically composed.

---

## 6. Compatibility Contract

Both static and dynamic modes must be **output-equivalent**. Regardless of how a capability is composed into an agent, the result written into the planner's task state must conform to the same schema.

```
┌─────────────────────────────────────────┐
│   Domain Agent (caller)                 │
│                                         │
│   ┌──── Dynamic Skill Path ────┐       │
│   │ Load SKILL.md from registry│       │
│   │ Execute locally            │──┐    │
│   └────────────────────────────┘  │    │
│                                   ▼    │
│                     ┌─────────────────┐│
│                     │  task.outputs   ││ ← same schema
│                     └─────────────────┘│
│   ┌──── Static Path ──────────┐  ▲    │
│   │ Hardcoded prompt + tools  │  │    │
│   │ Execute as designed       │──┘    │
│   └───────────────────────────┘       │
└─────────────────────────────────────────┘
```

**Why this matters:** The Planner and any downstream consumers must not need to know *how* a capability was composed. They read from `task.outputs` regardless. If a capability is migrated from static to dynamic (or vice versa), nothing upstream or downstream should change.

This also applies to the SOA/LOA dimension: a capability can be invoked as a remote service (SOA via A2A) or loaded as a local skill (LOA via Agent Skills), and the output schema remains identical. The two dimensions — **composition method** (static vs. dynamic) and **invocation method** (service vs. local) — are independent:

| | Service Invocation (SOA) | Local Execution (LOA) |
| --- | --- | --- |
| **Static Composition** | Agent calls a remote service with hardcoded integration code | Agent executes hardcoded logic locally |
| **Dynamic Skills** | Agent discovers a service via skill registry, delegates via A2A | Agent loads skill from registry, executes locally |

---

## 7. Project Recommendation

For the Assessment Agentic Architecture — Phase 1:

| Capability | Composition Mode | Rationale |
| --- | --- | --- |
| **Intent Classifier** | Static | Well-defined, stable contract. Single implementation. |
| **Planner** | Static | Orchestration logic is tightly coupled to the graph/execution model. |
| **SLIC Agent** | Static | Single implementation, tightly coupled to SLIC DB interface. |
| **Knowledge Agent** | Static (candidate for dynamic skill) | RAG retrieval pattern is reusable across domains. First candidate for extraction into a skill package when a second domain ships. |
| **Data Query Agent** | Static (candidate for dynamic skill) | MCP tool orchestration is reusable. Large/variable output size requires context-fit validation before skill promotion. |
| **Config Best Practice Agent** | Static | Domain-specific reasoning — consumes core capabilities via Planner-mediated data flow. |
| **Security Assessment Agent** | Static | Same pattern as Config Best Practice Agent. |

### Phase 1 to Phase N Evolution

```text
Phase 1 (Static)                              Phase N (Selective Dynamic Skills)
┌──────────────────────┐                      ┌──────────────────────────────────┐
│ Config Best Practice │                      │ Config Best Practice Agent       │
│ Agent                │                      │  ├─ core-knowledge (dynamic)     │
│  ├─ hardcoded prompt │  ── evolves to ──▶   │  ├─ assessment-rules (dynamic)   │
│  ├─ hardcoded tools  │                      │  └─ data-query (service/SOA)     │◄── large outputs stay service-based
│  └─ wired context    │                      └──────────────────────────────────┘
└──────────────────────┘
                                               Skills loaded from registry:
                                               /skills/core-knowledge/SKILL.md
                                               /skills/assessment-rules/SKILL.md
```

In Phase N, `core-knowledge` has been extracted into a dynamic skill (reusable, small output, single-tenant). `data-query` remains service-based (variable output size, benefits from context control). `assessment-rules` is a new skill that packages best-practice evaluation logic for reuse across domain agents.

---

## 8. Relationship to SOA and LOA

This document reframes the SOA (Service-Oriented Architecture) vs. LOA (Library-Oriented Architecture) discussion from [earlier architectural strategy work](Architectural%20Strategy%20v4.md) in terms of capability composition:

| Earlier Term | Reframed As | What It Means |
| --- | --- | --- |
| **SOA** | Service invocation | A capability is executed by a separate service — the agent delegates over a network boundary (A2A, RemoteGraph) |
| **LOA** | Dynamic skill loading (contextual inheritance) | A capability is packaged as a `SKILL.md` and loaded into the agent's own runtime |

The key insight: **SOA vs. LOA is about invocation method, not composition method.** Both SOA and LOA can coexist with either static or dynamic composition:

- **Static + SOA:** Agent has hardcoded integration code that calls a remote service (Phase 1 default)
- **Static + LOA:** Agent has hardcoded skill-loading code for a locally bundled skill
- **Dynamic + SOA:** Agent discovers services via a registry and delegates at runtime
- **Dynamic + LOA:** Agent discovers and loads skills from a registry at runtime (full evolution)

The migration path combines both dimensions:
1. Start with **static + SOA** (Phase 1 — simple, proven)
2. Package reusable capabilities as skills (**dynamic** composition)
3. For capabilities that benefit from local execution, switch to **LOA** invocation
4. Capabilities requiring isolation, different models, or large outputs remain **SOA**

---

## Open Questions

- **Skill registry design:** What serves as the skill registry? A Git repository, a package registry, or a dedicated skill discovery service? How do agents authenticate and fetch skills?
- **Runtime validation:** How does an agent validate that a loaded skill is compatible with its context budget and tool environment before executing?
- **Skill versioning and rollback:** How are skill versions pinned? Can an agent roll back to a previous skill version if a new one degrades performance?
- **Cross-domain skill sharing:** Can a domain agent load skills from another domain (e.g., Config Best Practice Agent loading a Security Assessment reasoning skill)? What are the boundaries?
- **Composition depth:** If Agent A loads Skill B, and Skill B itself declares a dependency on Skill C, how deep does the inheritance chain go before context budget or coherence degrades?

---

## Sources

- Agent Skills format: [agentskills.io](https://agentskills.io/) — open format by Anthropic
- Agent Skills reference library: [github.com/agentskills/agentskills](https://github.com/agentskills/agentskills)
- A2A Protocol: [github.com/a2aproject/A2A](https://github.com/a2aproject/A2A)
- LangGraph: [langchain-ai.github.io/langgraph](https://langchain-ai.github.io/langgraph/)

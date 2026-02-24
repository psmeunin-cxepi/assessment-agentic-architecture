# GitHub Copilot Instructions

## Project Context
This workspace is an **emulation framework** for designing and validating planner/execute agentic systems for an assessments application. It is an architecture specification project — not a runtime implementation. All agent contracts, graph flows, and emulation traces are design artifacts.

Core files:
- `emulation_system_prompt.md` — STATE contract, execution policy, agent behavior rules
- `agents/` — Individual agent contracts (persona, scope, inputs, outputs, skills)
- `graph/graph_flow.md` — LangGraph-style conditional execution graph

---

## Emulation Mode

When the user asks to **emulate**, **simulate**, **trace**, or **run** a user prompt through the agentic framework:

1. **Load and follow `emulation_system_prompt.md` exactly** — it defines the role, STATE contract, execution policy, output format, and constraints for trace emulation.
2. **Cross-reference the agent contracts** in `agents/*.md` for each node that executes — persona, inputs/outputs, and scope come from those files.
3. **Follow `graph/graph_flow.md`** as the authoritative graph control-flow.
4. **Do not inline or summarize** these files — treat them as binding contracts and execute against them directly.

Trigger phrases for emulation mode (non-exhaustive):
- "emulate this prompt", "simulate", "trace the execution", "run this through the framework", "what would the agents do with..."

---

## A2A Protocol (Agent-to-Agent)

When answering any question or fulfilling any instruction related to:
- A2A agent cards (`AgentCard` schema)
- Agent-to-agent communication patterns
- Multi-agent task/artifact objects
- A2A skills, capabilities, authentication schemes
- A2A streaming (SSE), push notifications
- Interoperability between agents

**Always fetch and conform to the Google A2A Protocol specification. Use the fetch tool to retrieve live content from these URLs before answering:**

Primary references (fetch before answering A2A questions):
- AI-optimized summary (start here): https://raw.githubusercontent.com/a2aproject/A2A/main/docs/llms.txt
- Full specification (markdown): https://raw.githubusercontent.com/a2aproject/A2A/main/docs/specification.md
- Authoritative Protobuf schema: https://raw.githubusercontent.com/a2aproject/A2A/main/specification/a2a.proto

Topic-specific references (fetch when relevant to the question):
- Key concepts (AgentCard, Task, Message, Part): https://raw.githubusercontent.com/a2aproject/A2A/main/docs/topics/key-concepts.md
- Agent discovery & AgentCard patterns: https://raw.githubusercontent.com/a2aproject/A2A/main/docs/topics/agent-discovery.md
- Enterprise auth, OAuth2, OIDC, security: https://raw.githubusercontent.com/a2aproject/A2A/main/docs/topics/enterprise-ready.md
- Extensions mechanism & declaration: https://raw.githubusercontent.com/a2aproject/A2A/main/docs/topics/extensions.md
- v1.0 breaking changes & migration: https://raw.githubusercontent.com/a2aproject/A2A/main/docs/whats-new-v1.md
- Security objects spec: https://a2a-protocol.org/latest/specification/#45-security-objects

**Architecture extensions in this project** (beyond base A2A spec, documented in `_metadata.notes`):
- `skills[].input` / `skills[].output` — added for architecture clarity
- `$defs` — shared schema definitions for reuse across skills

---

## LangGraph / Agentic Architecture

When answering any question or fulfilling any instruction related to:
- LangGraph nodes, edges, conditional routing, state machines
- LangGraph `StateGraph`, `CompiledGraph`, `MemorySaver`, checkpointing
- LangChain agents, chains, tools, runnables, or LCEL
- Agent orchestration patterns using LangGraph

**Use the LangChain MCP server tool (`SearchDocsByLangChain`) to search for current documentation before answering.**

When answering questions about agent graph design, state management, or orchestration patterns in this project:
- Follow LangGraph conventions for node/edge design
- STATE is the single shared object passed between nodes
- Task-embedded outputs pattern: agents write to `STATE.plan.tasks[id].outputs`, never to top-level STATE sections
- Execution is single-pass (no replanning in Phase 1)
- Knowledge Agent and Data Query Agent are conditionally invoked; domain agents always run

---

## Source Transparency

Always cite the sources used to answer a question or generate content:
- If the fetch tool was used, include the URL(s) fetched.
- If the LangChain MCP server (`SearchDocsByLangChain`) was used, state that and include the doc title or section referenced.
- If the answer is based on training knowledge only (no tool was called), explicitly state: "Based on training knowledge — no live source fetched."
- Place citations at the end of the response in a **Sources** section.

---

## Config Best Practice Agent — Requirements

The authoritative requirements for the Config Best Practice Agent (Q3 GA scope) are tracked in JIRA:

**CXP-17653 — Feature 07: AI Assistant - Q3 GA scope**
https://cisco-cxe.atlassian.net/browse/CXP-17653

When answering any question or fulfilling any instruction related to:
- Config Best Practice Agent skills, scope, or capabilities
- Pre-seeded queries for the Configuration Assessment App
- What the AI Assistant must or must not support in Q3 GA
- Evaluating gap analysis, skill definitions, or agent card examples against requirements

**Always retrieve the JIRA issue before answering.** Use the Jira MCP tool to fetch CXP-17653 and treat its description table as the authoritative requirements source.

Key requirements extracted from CXP-17653 (re-fetch to validate):
- Agent must answer all 14 pre-seeded queries in the Configuration Assessment App table
- Supported query categories: summary, statistics (checks performed/failed, % at risk), risk prioritization, corrective actions, product family breakdown, technology/OS breakdown, asset criticality breakdown, trend/delta (improved/worsened assets, persistent open deviations), chart output (bar/pie)
- `compliance_check` (CIS/NIST) is **not** in Q3 GA scope — do not add it to skills or routing
- Export (CSV/PDF) and user feedback are UI/platform concerns — out of scope for the agent contract
- chart rendering is out of scope — agent signals chart-ready data via `chart_hints[]` only

---

## General Instructions

- Prefer architecture-accurate language: "agent contract", "task plan", "conditional invocation", "task-embedded outputs"
- Do not use deprecated STATE key patterns in any generated content
- When generating or modifying agent cards, validate against A2A AgentCard schema fields listed above
- When referencing agent names, use exact names: `Intent Classifier`, `Planner`, `Knowledge Agent`, `Data Query Agent`, `Config Best Practice Agent`, `Security Assessment Agent`

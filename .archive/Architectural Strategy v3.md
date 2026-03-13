# Architectural Strategy: Agentic Ecosystem Design

**Subject:** Service-Oriented (SOA) vs. Library-Oriented (LOA) Architectures

**Goal:** Defining a scalable, extensible framework where any agent capability is an importable skill.

---

## 1. Executive Summary

To build an agentic system that is both scalable and maintainable, we are moving toward a **Library-Oriented Architecture (LOA)**. In this model, every agentic capability—whether a core utility (Knowledge, Data) or a specialized domain (Tax, Legal)—is packaged as an importable **Agent Skill** using the `SKILL.md` standard. This approach enables **Contextual Inheritance**, allowing a primary agent to dynamically adopt new expertise without the latency or state-fragmentation common in traditional **Service-Oriented Architectures (SOA)**.

---

## 2. Defining the Paradigms

### A. Service-Oriented Architecture (SOA): The "Delegator"

In an SOA, capabilities are independent "Services." Agents talk to each other over a network.

* **Mechanism:** Communication via A2A Protocol (HTTP/JSON-RPC).
* **Analogy:** A General Contractor hiring a specialized subcontractor. You send a request, wait, and get a result.
* **Best For:** Compute-heavy tasks or when using different LLM models for different steps.

### B. Library-Oriented Architecture (LOA): The "Expert"

In an LOA, every capability is an importable "Skill Package." An agent "becomes" the expert by loading the skill's instructions and tools into its own execution loop.

* **Mechanism:** Dynamic prompt injection and local tool execution via the `SKILL.md` standard.
* **Analogy:** An artisan with a library of manuals. When they need to perform a task, they reference the manual and use their own hands to execute.
* **Best For:** Procedural logic, context acquisition, and low-latency reasoning.

---

## 3. How It Works: Implementation & Evolution

### A. From "Core" to "Universal" Skills

Initially, we treat Core Agents (Knowledge, Data) as skills to provide context. However, the architecture evolves so that **any** specialized agent is also a skill.

**Example Folder Structure:**

```text
/skills
  /core-knowledge/       # Core Skill
    ├── SKILL.md         # Retrieval instructions
    └── tools/           # VectorDB tools
  /tax-compliance/       # Domain Skill (Formerly "Tax Agent")
    ├── SKILL.md         # Audit procedures & logic
    └── tools/           # Calculation & filing tools

```

### B. Contextual Inheritance

When a primary agent needs to solve a tax problem involving enterprise data, it doesn't "call" two agents. It **inherits** two skills:

1. **Inherit Core:** Loads `core-knowledge` to access the database.
2. **Inherit Domain:** Loads `tax-compliance` to access the audit logic.
3. **Execute:** The agent now has the combined "brain" of both, executing the task in a single, unified state.

### C. The Hybrid "A2A" Layer

A2A is repurposed from a "communication" protocol to a **"Discovery and Transfer"** protocol:

* **Discovery:** Agents use A2A Agent Cards (`agent.json`) to advertise their available skill libraries.
* **Negotiation:** If Agent A needs a "Legal" capability it doesn't have, it uses A2A to find Agent B and "imports" the skill package to run it locally.
* **Heavy-Lift:** If a skill requires massive resources (e.g., 1,000 document parses), the agent uses A2A to delegate that specific step as an SOA service.

---

## 4. Comparative Analysis

| Feature | **Service-Oriented (SOA)** | **Library-Oriented (LOA)** |
| --- | --- | --- |
| **Performance** | **Higher Latency:** Network hops add overhead. | **Zero Latency:** Local execution is instant. |
| **Context** | **Clean:** Only results enter the context window. | **Dense:** Full skill instructions enter the context. |
| **State** | **Fragmented:** Hard to sync "memory" across agents. | **Unified:** Single `AgentState` across all skills. |
| **Flexibility** | **Model-Specific:** Each agent can use a different LLM. | **Host-Specific:** Tied to the primary agent's LLM. |
| **Maintenance** | **Decoupled:** Services update independently. | **Modular:** Skills are portable "folders." |

---

## 5. Summary Decision Framework

| Use **LOA (Import a Skill)** when... | Use **SOA (Call a Service)** when... |
| --- | --- |
| The task provides **data, context, or procedures**. | The task is **compute-heavy** (ETL, heavy scraping). |
| You need **low latency** for a complex chain. | You need to use a **specialized model** (e.g., long-context). |
| You want a **unified state** and history. | You need to **hide proprietary logic** behind an API. |
| The capability is **procedural** (Domain logic). | The task is an **isolated, massive bulk operation**. |

---

### Final Reflection

By treating "Everything as a Skill," we eliminate the overhead of managing a swarm of independent agents. Instead, we build a **standardized library of expertise** that any agent can pull from, making the system infinitely extensible and much easier to debug.

**Next Step:** Since any agent can now be a skill, would you like me to create a **Template for a Domain-Level Skill** (like a "Tax" or "Legal" skill) that includes a multi-step agentic workflow within its `SKILL.md`?
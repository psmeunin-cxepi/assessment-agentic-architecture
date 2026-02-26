# Architectural Strategy: Agentic Ecosystem Design

**Subject:** Service-Oriented (SOA) vs. Library-Oriented (LOA) Architectures

**Goal:** Defining a scalable and extensible framework for Core and Domain Agents.

---

## 1. Executive Summary

To build an agentic system that scales, we must decide how "Domain Agents" (the specialists) interact with "Core Agents" (the utilities like Knowledge and Data). We categorize these interactions into two primary patterns: **Service-Oriented (The Delegator)** and **Library-Oriented (The Expert)**.

---

## 2. Defining the Paradigms

### A. Service-Oriented Architecture (SOA)

**The "Delegator" Model** In an SOA, capabilities are treated as independent, live "Services." A Domain Agent does not possess core logic; instead, it sends a request to a specialist via a network protocol (A2A).

* **Mechanism:** Agent-to-Agent communication (HTTP, JSON-RPC, or message queues).
* **Analogy:** A General Contractor (Domain Agent) hiring a specialized Electrician (Core Agent) to handle the wiring. The Contractor doesn't need to know *how* to wire; they just need to know *who* to call.

### B. Library-Oriented Architecture (LOA)

**The "Expert" Model** In an LOA, capabilities are treated as modular "Skill Packages" (leveraging the `SKILL.md` standard). A Domain Agent **imports** the expertise, instructions, and tools of the core agent directly into its own runtime.

* **Mechanism:** Context injection and local tool execution.
* **Analogy:** An Expert Artisan who owns a vast library of technical manuals. When they need to perform "wiring," they pull the specific manual from the shelf and use their own tools to do the work.

---

## 3. Refined Pros and Cons

| Feature | **Service-Oriented (SOA)** | **Library-Oriented (LOA)** |
| --- | --- | --- |
| **Performance** | **Lower:** Subject to network latency and "chatty" API overhead. | **Higher:** Zero-latency execution; logic runs in the same process. |
| **Context Management** | **Efficient:** Only the *final result* enters the agent's context. | **Risky:** Loading complex skills can "bloat" the context window. |
| **State Consistency** | **Challenging:** Requires passing state back and forth between runtimes. | **Native:** Uses a single, unified `AgentState` throughout the task. |
| **Model Flexibility** | **Superior:** The Data Agent can use a cheap model while the Domain Agent uses an expensive one. | **Limited:** Typically tied to the host agent's primary model. |
| **Maintenance** | **Decoupled:** Update the Knowledge Service without touching Domain Agents. | **Coupled:** Skill updates may require re-validating the agents that import them. |
| **Security** | **Granular:** Strict API-level permissions and data masking. | **Broad:** Skills usually have full access to the agent's memory/tools. |

---

## 4. Implementation: Contextual Inheritance & A2A

The most extensible systems use **LOA for expertise** and **A2A for discovery**.

### Contextual Inheritance (The LOA Mechanism)

"Contextual Inheritance" allows a Domain Agent to adopt the persona and tools of a Core Agent dynamically without leaving its reasoning loop.

1. **Identification:** The Domain Agent (e.g., "Tax-Agent") identifies a need for "Enterprise Knowledge."
2. **Skill Loading:** It reads the `knowledge-retriever/SKILL.md`.
3. **Instructional Inheritance:** The specific instructions for how to query the company database are injected into its system prompt.
4. **Tool Adoption:** The agent registers the retrieval tools defined in the skill folder.
5. **Local Execution:** The agent performs the retrieval itself, meaning the "context" is acquired and used immediately without a secondary agent hand-off.

### The Role of A2A (The Connectivity Tissue)

A2A remains the protocol for **Discovery and Negotiation**:

* **Discovery:** Agents use A2A (and Agent Cards) to broadcast which "Skills" they have available in their library.
* **Skill Transfer:** An agent could theoretically use A2A to "download" a new skill package from a central repository.
* **Delegation:** For compute-heavy tasks that exceed local resources, the agent reverts to SOA, using A2A to delegate the entire task to a remote service.

---

## 5. Summary Decision Matrix

Use the table below to determine if a new capability should be a **Skill (LOA)** or a **Service (SOA)**.

| Criteria | **Choose Skill (LOA)** | **Choose Service (SOA)** |
| --- | --- | --- |
| **Primary Goal** | Providing data, context, or procedures. | Compute-heavy processing (ETL, massive scraping). |
| **Latency Tolerance** | Low (Needs to be "instant"). | High (User is okay with a "thinking" spinner). |
| **Model Requirements** | Can run on the Domain Agent's model. | Requires a specialized or different LLM. |
| **Workflow** | Part of a larger reasoning chain. | A standalone, isolated task. |

---

**Next Step:** Would you like me to draft a sample **`SKILL.md` manifest** for a "Core Knowledge Agent" so you can test how this inheritance would look in your code?
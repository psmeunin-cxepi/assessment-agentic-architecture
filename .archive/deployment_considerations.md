# Deployment Considerations

## Deployment Model Analysis

### Options Evaluated

| # | Model | Description |
|---|---|---|
| 1 | **1 Graph (flat)** | All agents as simple nodes in a single `StateGraph` |
| 2 | **1 Graph + Sub-Graphs** | Parent orchestrator graph with domain/execution agents as compiled subgraphs |
| 3 | **Multiple Graphs (RemoteGraph)** | Agents deployed as separate Agent Server instances, composed via `RemoteGraph` |
| 4 | **Multiple Graphs (A2A)** | Agents as independent A2A-compliant services communicating via Task/Message/Artifact |

### Comparison Matrix

| Criterion | 1 Graph (flat) | 1 Graph + Sub-Graphs | Multi-Graph (RemoteGraph) | Multi-Graph (A2A) |
|---|---|---|---|---|
| Matches STATE contract | Yes | **Yes** (shared keys + private state) | Partial (state serialized over HTTP) | No (A2A uses Task/Message/Artifact, not shared STATE) |
| Team-parallel development | No (single file) | **Yes** (subgraph interface = contract) | Yes | Yes |
| Operational complexity | Lowest | **Low** (single deployment) | High (N deployments, version drift) | Highest (A2A servers, discovery, auth) |
| Independent scaling | No | No (same process) | Yes | Yes |
| Phase 2 extensibility | Hard (flat graph grows) | **Easy** (add subgraph node) | Easy | Easy |
| Checkpointing | Built-in | **Auto-propagated** to subgraphs | Manual (thread-ID coordination) | Out of scope |
| Debugging (LangGraph Studio) | Full | **Full** (subgraph drill-down) | Per-deployment only | Not applicable |
| Migration path to Option 3 | Refactor | **Promote subgraph → RemoteGraph** | Already there | Different paradigm |

---

## Recommendation: 1 Graph with Sub-Graphs (Option 2)

### Proposed Graph Decomposition

```
Parent Graph (orchestrator)
├── Node: Intent Classifier          ← simple node (lightweight, no internal graph needed)
│   └── Conditional edge: clarification gate → short-circuit to user
├── Node: Planner                     ← simple node
├── Sub-Graph: Knowledge Agent        ← private state: RAG query formulation, chunk retrieval pipeline
├── Sub-Graph: Data Query Agent       ← private state: MCP tool selection, SQL query planning
├── Sub-Graph: Config Best Practice   ← private state: finding generation pipeline, evidence matching
├── Sub-Graph: Security Assessment    ← private state: risk scoring pipeline
└── Future: Sub-Graph: Assessment X   ← drop in as new subgraph node
```

### Why Simple Nodes for IC and Planner

Intent Classifier and Planner are thin orchestration nodes — they read STATE, write a few keys, and exit. No internal multi-step pipeline justifies a subgraph.

### Why Sub-Graphs for Execution and Domain Agents

1. **Private internal state** — DQA can maintain MCP tool selection reasoning, SQL query drafts, and retry logic without polluting parent STATE. CBP can maintain intermediate finding candidates before committing to `task.outputs.findings[]`.
2. **Clean interface** — each subgraph shares only the keys it needs (`plan.tasks[]`, `intent.*`, `trace.*`) while keeping internal processing opaque to the parent. This maps directly to existing agent contract "Inputs (STATE read)" / "Outputs (STATE write)" sections.
3. **Independent development** — different team members own different subgraphs. As long as the shared-key interface is respected, the parent graph doesn't change.
4. **`Command.PARENT`** — subgraph nodes can signal completion back to the parent graph for routing decisions.

### State Communication Pattern

Per LangGraph docs, two patterns are available:

| Pattern | When to use | Assessment framework applicability |
|---|---|---|
| **Add subgraph as node** (shared keys) | Parent and subgraph share state keys | `plan.tasks[]`, `intent.*`, `trace.*` — agent reads/writes directly |
| **Call subgraph inside node** (mapped state) | Different schemas, need transformation | If a domain agent needs private scratch space not in parent STATE |

**Recommended**: Start with shared-key subgraphs (simpler). If an agent needs private working memory, wrap it in a node function that maps parent STATE → subgraph input and subgraph output → parent STATE writes.

---

## Why Not Options 3 or 4

### RemoteGraph (Option 3)

Justified when agents need independent scaling or run on different infrastructure. The current architecture has 4 intent classes, 6 agents, and single-pass execution. All agents share STATE and execute in dependency order. Deploying each as a separate Agent Server adds HTTP latency per hop, thread-ID coordination overhead, and deployment version drift risk — for no current benefit.

**Important**: `RemoteGraph` must not be used to call graphs on the same deployment (deadlock and resource exhaustion risk per LangGraph docs). Use local graph composition or subgraphs for graphs within the same deployment.

### A2A (Option 4)

The right choice when agents cross organizational boundaries, use different frameworks, or need discovery/interoperability with external systems. The A2A agent cards in this project are valuable as **architecture specification artifacts** and should be retained, but using A2A as the runtime protocol would mean abandoning shared STATE in favor of Task/Message/Artifact exchange — a fundamentally different execution model that doesn't match the planner/execute pattern.

---

## Migration Path

```
Phase 1 (Q3 GA):  1 Graph + Sub-Graphs  ←  recommended
Phase 2+:          Promote high-load subgraphs to RemoteGraph if scaling demands it
Future:            Expose domain agents via A2A for cross-platform interop (agent cards already exist)
```

The sub-graph → RemoteGraph promotion is low-friction: `RemoteGraph` provides API parity with `CompiledGraph`, so a subgraph deployed to its own Agent Server can be swapped in as a `RemoteGraph` node with minimal parent graph changes.

---

## Sources

- LangGraph docs: [Subgraphs](https://docs.langchain.com/oss/python/langgraph/use-subgraphs) — subgraph communication patterns, shared keys vs mapped state
- LangGraph docs: [RemoteGraph](https://docs.langchain.com/langsmith/use-remote-graph) — API parity, deadlock warning, subgraph embedding
- LangGraph docs: [Workflows and agents](https://docs.langchain.com/oss/python/langgraph/workflows-agents) — workflow vs agent patterns
- LangGraph docs: [Supervisor pattern](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents-personal-assistant) — supervisor + subagent architecture
- LangGraph docs: [Subgraph persistence](https://docs.langchain.com/oss/python/langgraph/add-memory) — auto-propagated checkpointing
- LangGraph docs: [Command.PARENT navigation](https://docs.langchain.com/oss/python/langgraph/graph-api) — subgraph-to-parent routing
- LangGraph docs: [Multiple agent subgraph handoffs](https://docs.langchain.com/oss/python/langchain/multi-agent/handoffs) — context engineering between subgraphs

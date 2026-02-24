# Assessment Agentic AI — Architecture Emulation Framework

Design, validate, and iterate on a **planner/execute multi-agent architecture** for an assessments application — without writing runtime code. Agent contracts, graph flows, and shared-state rules are the deliverables; emulated execution traces prove they work together.

## Project Structure

```
agents/                        # Agent contracts (persona, scope, inputs/outputs, rules)
  01_intent_classifier.md        Intent classification + entity extraction
  02_planner.md                  Deterministic task planning + routing
  03_knowledge_agent.md          Enterprise knowledge retrieval (RAG)
  04_data_query_agent.md         Assessment data retrieval (MCP tools / SQL)
  05_config_best_practice_agent.md   Configuration validation domain agent
  06_security_assessment_agent.md    Security posture domain agent

graph/
  graph_flow.md                # Authoritative graph: transitions, conditions, Mermaid diagram

tools/mcp/                     # MCP tool contracts (signatures, parameters, response schemas)
  assessment_analysis_tool.md
  assessment_comparison_tool.md
  issue_tracking_tool.md

eval/
  eval_dataset.json            # 30 eval cases with expected classifications + outputs
  emulation_readiness.md       # Gap analysis for emulation fidelity
  traces/                      # Emulation trace outputs
    EVAL-001_cbp_assessment_summary.md

emulation_system_prompt.md     # System prompt that drives trace emulation
deployment_considerations.md   # Deployment model analysis (sub-graphs vs RemoteGraph vs A2A)
architecture_decisions.md       # Architecture decisions, trade-offs, future work
```

## Agents

| Layer | Agent | Role |
|---|---|---|
| Orchestration | Intent Classifier | Classifies intent, extracts entities, gates ambiguity |
| Orchestration | Planner | Builds conditional task plan with dependencies |
| Execution | Knowledge Agent | RAG retrieval of enterprise policies/standards |
| Execution | Data Query Agent | MCP tool or SQL data retrieval |
| Domain | Config Best Practice Agent | Configuration assessment, trends, expert insights |
| Domain | Security Assessment Agent | Security posture analysis |

## Running an Emulation

An emulation simulates a user prompt through the full agent graph and produces a trace document.

### Prerequisites
- An LLM that can follow a system prompt (e.g., GitHub Copilot, Claude, GPT-4)
- The files in this repo as context

### Steps

1. **Load the system prompt** — open `emulation_system_prompt.md` and provide it as the system prompt (or paste it as context).
2. **Pick an eval case** — choose a prompt from `eval/eval_dataset.json`:
   ```json
   {
     "id": "EVAL-001",
     "user_prompt": "Can you provide a summary of my recent Configuration Best Practice assessment?",
     "expected": { "intent_class": "cbp_assessment", ... }
   }
   ```
3. **Run the prompt** — send the `user_prompt` to the LLM. It will produce:
   - Intent classification (intent_class, entities, confidence)
   - Mermaid graph of the executed path
   - Node-by-node trace with STATE deltas
   - Final assessment outcome
4. **Save the trace** — write the output to `eval/traces/EVAL-XXX_<name>.md`.
5. **Compare against expected** — check intent_class, agents invoked, MCP tools used, and output types match the eval case.

### Example Trace
See [eval/traces/EVAL-001_cbp_assessment_summary.md](eval/traces/EVAL-001_cbp_assessment_summary.md) for a complete example.

## Key Design Decisions

- **Single-pass execution** — no replanning; agents handle data gaps with assumptions and confidence scores
- **Task-embedded outputs** — all agent outputs live in `tasks[].outputs`, not top-level STATE sections
- **Clarification gate** — ambiguous prompts (confidence < 0.5) short-circuit before the Planner
- **Conditional invocation** — Knowledge Agent and Data Query Agent run only when needed per intent_class
- **Anti-hallucination** — agents must not fabricate tool results, RAG chunks, or assessment data

---

**Status**: Architecture complete — ready for implementation and eval expansion  
**Last Updated**: 2026-02-24

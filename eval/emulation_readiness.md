# Emulation Readiness: Gap Analysis

## Purpose

This document identifies what is **present**, what is **missing**, and what is **stale** in the framework — evaluated against the requirements for running a complete emulation trace for any eval case in `eval/eval_dataset.json`.

---

## Execution Path (what happens during an emulation)

```
User Prompt + context_kv
  → Intent Classifier (reads: input.*, writes: intent.*)
    → [clarification gate: short-circuit if unknown_or_needs_clarification]
  → Planner (reads: intent.*, writes: plan.tasks[], plan.routing[])
  → Knowledge Agent [conditional] (reads: intent.*, plan.*, writes: task.outputs.enterprise_context)
  → Data Query Agent [conditional] (reads: intent.*, plan.*, schema, ontology, writes: task.outputs.assessment_context)
  → Domain Agent (reads: intent.*, upstream task.outputs.*, writes: task.outputs.findings/summary/etc.)
  → Final STATE returned
```

Each step requires **input data** or **context artifacts** to produce realistic output. The emulator (an LLM following `emulation_system_prompt.md`) must either have these artifacts provided or explicitly document them as `unknown`/simulated.

---

## 1. What EXISTS and is ready

| Artifact | Location | Status |
|---|---|---|
| Emulation system prompt | `emulation_system_prompt.md` | Ready — STATE contract, execution policy, output format defined |
| Graph flow contract | `graph/graph_flow.md` | Ready — transitions, conditions, Mermaid diagram |
| Agent contracts (6) | `agents/*.md` | Ready — persona, scope, inputs/outputs, rules |
| Agent cards (6) | `agents/*_card.json` | Ready — validated JSON, A2A-compliant |
| MCP tool contracts (3) | `tools/mcp/*.md` | Ready — signatures, parameters, response schemas, caps |
| Eval dataset | `eval/eval_dataset.json` | Ready — 30 cases with expected classification, routing, outputs |
| Clarification gate | IC + graph + emulation prompt | Ready — `unknown_or_needs_clarification` short-circuit defined |
| Deployment considerations | `deployment_considerations.md` | Informational — not needed for emulation |

---

## 2. What is MISSING (required for realistic emulation)

### GAP-1: No sample MCP tool responses (CRITICAL)

**Problem**: The emulator knows *which* MCP tool to call and *what schema* the response follows (from `tools/mcp/*.md`), but has no **sample data** to populate `task.outputs.assessment_context`. Without this, every DQA node produces `"report_data": "unknown"` and domain agents operate on empty data.

**Impact**: Domain agents cannot generate realistic findings, statistics, trends, or chart hints. The trace devolves into "would produce findings if data were available."

**What's needed**: Sample response fixtures for each MCP tool — realistic JSON payloads matching the response schemas in `tools/mcp/*.md`:

| Fixture needed | MCP tool | Response schema source | Used by eval cases |
|---|---|---|---|
| `assessment_analysis_summary.json` | `assessment_analysis_tool` (query_type=summary) | [assessment_analysis_tool.md](tools/mcp/assessment_analysis_tool.md) | EVAL-001–010, 014–015, 026–027, 029 |
| `assessment_analysis_filtering.json` | `assessment_analysis_tool` (query_type=filtering) | [assessment_analysis_tool.md](tools/mcp/assessment_analysis_tool.md) | severity/product filtered queries |
| `assessment_analysis_assets.json` | `assessment_analysis_tool` (query_type=assets) | [assessment_analysis_tool.md](tools/mcp/assessment_analysis_tool.md) | EVAL-005, 009, 010 |
| `assessment_analysis_technology.json` | `assessment_analysis_tool` (query_type=technology) | [assessment_analysis_tool.md](tools/mcp/assessment_analysis_tool.md) | EVAL-006, 014 |
| `assessment_comparison_assets.json` | `assessment_comparison_tool` (comparison_focus=assets) | [assessment_comparison_tool.md](tools/mcp/assessment_comparison_tool.md) | EVAL-011, 012, 028 |
| `issue_tracking_compliance.json` | `issue_tracking_tool` (tracking_scope=compliance) | [issue_tracking_tool.md](tools/mcp/issue_tracking_tool.md) | EVAL-013 |

**Recommendation**: Create `eval/fixtures/` directory with sample response files. Each fixture should:
- Conform exactly to the response schema in the tool contract
- Contain realistic Cisco assessment data (product families, rule IDs, severity distribution)
- Include enough records to support statistical, chart, and trend queries
- Be referenced by eval cases via a `fixture` field

---

### GAP-2: No sample enterprise context / RAG chunks (HIGH)

**Problem**: Knowledge Agent retrieves enterprise context from Vector DB (RAG), but no sample RAG chunks exist for emulation. When KA is invoked (cbp_expert_insights always, cbp_generic always, cbp_assessment conditionally), it has no corpus to draw from.

**Impact**: Expert insights and generic knowledge queries produce generic/assumed outputs rather than enterprise-specific responses.

**What's needed**: Sample RAG chunk fixtures matching the Knowledge Agent output structure:

```json
{
  "retrieved_chunks": [
    {
      "content": "...",
      "metadata": {
        "source": "policies/...",
        "topic": "approved_configurations|approved_exceptions|policies|escalation_procedures|organizational_context",
        "domain": "cbp_assessment|security_assessment|general",
        "relevance_score": 0.0-1.0,
        "timestamp": "YYYY-MM-DD"
      }
    }
  ],
  "query_used": "..."
}
```

**Recommended fixtures**:

| Fixture | Topic coverage | Used by eval cases |
|---|---|---|
| `enterprise_context_cbp.json` | approved_configurations, approved_exceptions, policies for config BP domain | EVAL-016, 017, 020, 030 |
| `enterprise_context_security.json` | security policies, escalation_procedures, organizational_context | EVAL-021 |
| `enterprise_context_generic.json` | general best practices, vendor recommendations, industry standards | EVAL-018, 019, 020 |

---

### GAP-3: No `context_kv` examples defined (MEDIUM)

**Problem**: `STATE.input.context_kv` is declared in the STATE contract and consumed by the Intent Classifier, but there is no specification of what key-value pairs it contains or example payloads. The existing trace shows `"context_kv": {}`.

**Impact**: Emulations cannot test context-aware intent classification (e.g., active assessment context, user role, customer ID).

**What's needed**: Define the `context_kv` schema:

```json
{
  "context_kv": {
    "customer_id": "string — active customer/tenant identifier",
    "active_assessment_id": "string — currently viewed assessment ID (from UI)",
    "user_role": "string — admin|analyst|viewer",
    "app_context": "string — configuration_assessment|security_assessment",
    "session_id": "string — conversation session identifier"
  }
}
```

**Recommendation**: Add `context_kv` schema to `emulation_system_prompt.md` under the STATE contract. Create 2–3 sample `context_kv` payloads in eval fixtures.

---

### GAP-4: No `STATE.schema` / `STATE.ontology` sample data (LOW — SQL Path only)

**Problem**: The DQA SQL Path requires `STATE.schema` (table structures) and `STATE.ontology` (semantic metadata) to generate queries. The DQA contract contains a *conceptual* ontology example, but no usable fixture exists.

**Impact**: SQL Path emulations cannot produce realistic queries. However, all Q3 GA eval cases use the **MCP Path** — none currently exercise the SQL Path.

**What's needed** (Phase 2):
- Full `schema.json` fixture with table definitions matching the assessment database
- Full `ontology.json` fixture with semantic descriptions, value enumerations, entity-column mappings
- At least 1 eval case exercising the SQL Path

**Recommendation**: Defer to Phase 2. Document as known gap. All Q3 GA queries are well-served by MCP tools.

---

### GAP-5: No conversation history / multi-turn context (LOW — Future)

**Problem**: `STATE.conversation.history[]` appears in Knowledge Agent inputs as "future: for follow-up query augmentation." Memory node exists in graph but has no contract.

**Impact**: Multi-turn follow-up scenarios (e.g., "What about security?" after a config assessment) cannot be emulated.

**What's needed** (Phase 2+):
- Define `STATE.conversation.history[]` schema
- Define Memory node contract (or incorporate into existing agent)
- Add multi-turn eval cases

**Recommendation**: Defer to Phase 2+. Q3 GA treats every prompt as a standalone turn.

---

## 3. What is STALE (exists but outdated)

### STALE-1: Existing emulation trace uses deprecated intent_class

[emulation_trace_001_config_summary.md](emulation_trace_001_config_summary.md) uses `intent_class: "assessment_summary"` — this is not a valid intent_class. Current valid values: `cbp_assessment`, `cbp_expert_insights`, `cbp_generic`, `security_assessment`, `unknown_or_needs_clarification`.

The trace also uses deprecated internal function name `get_assessment_summary` instead of MCP tool name `assessment_analysis_tool`.

**Recommendation**: Regenerate trace 001 using the current architecture. The same prompt ("Provide me a summary of the configuration best practice assessment, show the top 5 issues") maps to EVAL-001 with expected `intent_class: "cbp_assessment"`.

### STALE-2: Trace missing new STATE.intent fields

The existing trace omits `meta_intent`, `domain_details`, and `clarification_question` from the intent output. These were added during the clarification gate implementation.

---

## 4. Prioritised action plan

| Priority | Action | Files to create/modify | Prerequisite |
|---|---|---|---|
| **P0** | Create MCP tool response fixtures | `eval/fixtures/assessment_analysis_summary.json` + 5 more | MCP tool response schemas (exist) |
| **P0** | Create enterprise context (RAG) fixtures | `eval/fixtures/enterprise_context_cbp.json` + 2 more | KA output structure (exists) |
| **P1** | Define `context_kv` schema | Update `emulation_system_prompt.md` | UI/platform team input on available KV pairs |
| **P1** | Link eval cases to fixtures | Add `fixture_ref` field to eval_dataset.json entries | Fixtures exist (P0) |
| **P1** | Regenerate trace 001 | Rewrite `emulation_trace_001_config_summary.md` | Fixtures exist (P0) |
| **P2** | Schema + Ontology fixtures (SQL Path) | `eval/fixtures/schema.json`, `ontology.json` | Database schema available |
| **P2** | Conversation history schema | Update STATE contract, KA contract | Multi-turn design decisions |
| **P2** | SQL Path eval cases | Add to `eval/eval_dataset.json` | Schema fixtures (P2) |

---

## 5. Summary

**Ready to emulate (with simulated/unknown data)**: All 30 eval cases can be run through the emulator today. The graph execution path is fully specified — IC classifies, Planner routes, agents execute in order, task outputs are written.

**Ready to emulate (with realistic data)**: Requires P0 fixtures (MCP response samples + RAG chunk samples). Once created, the emulator can produce realistic findings, statistics, trend data, and chart hints instead of `"unknown"` placeholders.

**Blocked**: Nothing is blocked. The architecture contracts are complete. Gaps are data/fixture gaps, not design gaps.

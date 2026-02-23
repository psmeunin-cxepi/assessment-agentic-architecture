# Assessment Agentic AI Architecture - Emulation Framework

An **emulation framework** for designing and testing **planner/execute agentic systems** for an assessments application. This framework allows architects to define agent contracts, execution flows, and state management patterns, then emulate execution traces to validate the architecture before implementation.

## Overview

This project provides a **complete architectural specification and emulation system** for building multi-agent assessment frameworks that:
- Understand user intent and extract semantic entities
- Plan conditional task execution based on data requirements
- Retrieve enterprise knowledge and assessment data from multiple sources
- Perform specialized domain analysis (configuration best practices, security assessment)
- Provide evidence-based findings with explicit data gaps and assumptions

**Purpose**: Design, document, and validate agentic architectures through emulated execution traces without writing implementation code.

## What This Framework Provides

### 1. Agent Contract Definitions
Formal specifications for each agent including:
- Scope and responsibilities
- Input/output contracts
- Data dependencies
- Behavioral guidelines
- Anti-hallucination constraints

### 2. Graph Flow Specification
Deterministic execution flow defining:
- Agent transition rules
- Conditional invocation logic
- Task dependency ordering
- State management patterns

### 3. Emulation System
Trace generation capability that:
- Simulates agent execution
- Shows state evolution
- Validates architectural decisions
- Identifies design gaps

### 4. Architecture Documentation
Comprehensive specifications including:
- Decision rationale (caveats.md)
- State contracts
- Data access patterns
- Future enhancement roadmap

## Architecture Pattern (Designed for Assessments App)

**Plan/Execute with Task-Embedded Outputs**
- **Single-pass execution** (no replanning in current phase)
- **Conditional agent invocation** (agents only run when needed)
- **Task-based provenance** (all outputs embedded in task objects)
- **Clear separation of concerns** (orchestration vs. execution vs. validation)

This pattern is specified through agent contracts and graph flow definitions, validated through emulation traces.

## Core Agents (Architectural Specification)

### Orchestration Layer
- **Intent Classifier** - Classifies user queries and extracts semantic entities (site, device, timeframe, severity, etc.)
- **Planner** - Creates conditional task plans with dependency ordering based on requirements

### Execution Layer
- **Knowledge Agent** - Retrieves enterprise knowledge via RAG (policies, standards, organizational context)
- **Data Query Agent** - Fetches assessment data via MCP tools or SQL queries with semantic entity binding

### Domain Layer
- **Config Best Practice Agent** - Validates configurations, identifies misconfigurations, provides remediation recommendations
- **Security Assessment Agent** - Analyzes security posture, identifies vulnerabilities and risks

*Each agent defined through formal contracts in `/agents/*.md`*

## Data Access Patterns

### MCP Path (Well-Known Intents)
- Pre-built assessment APIs (`get_assessment_summary`, `compare_assessments`, `get_unresolved_issues`)
- Semantic entity-to-parameter binding
- Fast retrieval of pre-computed results

### SQL Path (Complex/Bespoke Queries)
- Direct database access with schema + ontology guidance
- Flexible query generation for custom analysis
- Raw configuration and assessment data

### RAG Path (Enterprise Knowledge)
- Vector database retrieval of enterprise context
- Filtered by domain, topic, and relevance
- Contextualizes assessments with organizational knowledge

## State Management

**Shared STATE Object** with task-embedded outputs:

```json
{
  "intent": { "intent_class", "entities[]", "confidence" },
  "plan": {
    "tasks": [
      {
        "id": "T1",
        "owner": "Agent Name",
        "depends_on": [],
        "required_data": [],
        "status": "completed",
        "outputs": { /* Agent-specific outputs */ }
      }
    ]
  },
  "schema": { /* Database structure */ },
  "ontology": { /* Data semantics */ },
  "trace": { "node_run_order[]", "state_deltas[]" }
}
```

**Key Principle**: All agent outputs written to `tasks[].outputs` for clear provenance and lineage.

## Documentation Structure

```
/agents/               # Agent contract definitions
  01_intent_classifier.md
  02_planner.md
  03_knowledge_agent.md
  04_data_query_agent.md
  05_config_best_practice_agent.md
  06_security_assessment_agent.md
  README.md           # Agent overview

/graph/
  graph_flow.md       # Graph execution flow contract

emulation_system_prompt.md  # System prompt for trace emulation
emulation_trace_*.md        # Example execution traces
caveats.md                  # Architecture decisions and future work
```

---

**Framework Purpose**: Design and validate agentic architectures for assessments app
**Implementation Status**: Architecture complete, ready for implementation  
**Last Updated**: 2026-02-23

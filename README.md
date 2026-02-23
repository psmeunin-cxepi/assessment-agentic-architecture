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

## Key Architectural Decisions

### 1. Task-Embedded Outputs (Architecture Decision #7)
All agent outputs stored in task objects rather than separate STATE sections, providing clear task-to-output traceability.

### 2. Conditional Agent Invocation
- **Knowledge Agent**: Only invoked when enterprise context interpretation needed
- **Data Query Agent**: Only invoked when domain agents declare data dependencies
- **Domain Agents**: Always invoked to perform assessment/analysis

### 3. Single-Pass Execution (Current Phase)
No replanning or cycles - agents handle data gaps with explicit assumptions, confidence scores, and documented limitations.

### 4. Semantic Entity Extraction
Intent Classifier extracts generic semantic entities (not tool-specific parameters), enabling flexible downstream mapping.

### 5. Schema + Ontology for SQL Path
Two-layer approach: structural schema (HOW to query) + semantic ontology (WHAT data means).

## Current Phase: Architecture Definition Complete

**Framework Includes**:
- ✅ Conditional planning with task dependencies (specified)
- ✅ Task-embedded outputs architecture (documented)
- ✅ Entity extraction and semantic matching (defined)
- ✅ Dual-path data retrieval - MCP + SQL (specified)
- ✅ Enterprise knowledge integration - RAG (contract defined)
- ✅ Configuration and security domain agents (contracts complete)
- ✅ Evidence-based findings with data gap handling (documented)
- ✅ Emulation system for trace generation

**Future Enhancements (Phase 2+)**:
- Assessment strategy planning (Knowledge Agent enhancement)
- Replanning and multi-iteration support
- Additional domain agents (compliance, performance, etc.)
- Multi-turn conversation support

**Status**: Ready for implementation teams to build against these specifications

## Anti-Hallucination Principles

1. **Evidence-based findings** - Every finding references specific data sources
2. **Explicit assumptions** - All inferences documented in findings
3. **Data gap transparency** - Missing data acknowledged, not fabricated
4. **Confidence scoring** - Findings scored based on evidence quality
5. **Source citation** - Enterprise context and data sources referenced

## Getting Started with the Emulation Framework

### Understanding the Architecture
1. Read [agents/README.md](agents/README.md) for agent overview
2. Review [graph/graph_flow.md](graph/graph_flow.md) for execution flow
3. See [emulation_trace_001_config_summary.md](emulation_trace_001_config_summary.md) for example execution
4. Review [caveats.md](caveats.md) for architectural decisions and trade-offs

### Using the Emulation System
1. Define user prompt
2. Use [emulation_system_prompt.md](emulation_system_prompt.md) to generate execution trace
3. Validate agent behavior against contracts
4. Identify gaps or issues in architecture
5. Refine contracts and flows as needed

### Agent Contract Structure
Each agent contract (`agents/*.md`) defines:
- **Persona** - Behavioral characteristics
- **Primary Objective** - Core responsibility
- **Scope (in)** - What the agent handles
- **Out of Scope** - What the agent must not do
- **Data Dependencies** - Required inputs per skill
- **Inputs (STATE read)** - What state keys the agent reads
- **Outputs (STATE write)** - What the agent writes to task outputs
- **Processing Logic** - How the agent operates

### Extending the Framework
**Adding a new domain agent:**
1. Create contract file: `agents/0X_new_agent.md`
2. Define scope, inputs, outputs, data dependencies
3. Update `graph/graph_flow.md` with transition rules
4. Update `emulation_system_prompt.md` agent list
5. Generate emulation trace to validate behavior

**Adding a new intent class:**
1. Update Intent Classifier contract with entity patterns
2. Update Planner with routing logic
3. Define which agents should handle the intent
4. Generate emulation trace to validate flow

## Design Philosophy

- **Simplicity over complexity** - Start with minimal viable architecture, enhance incrementally
- **Explicit over implicit** - Clear contracts, documented assumptions, transparent limitations
- **Separation of concerns** - Orchestration ≠ Execution ≠ Validation ≠ Data Retrieval
- **Evidence-based reasoning** - No hallucination, all claims backed by data
- **Flexible semantics** - Semantic matching over rigid mappings
- **Clear provenance** - Every output traceable to specific task and agent

## Use Cases for This Framework

### Architecture Design
- Define agent responsibilities and boundaries
- Specify data flow and state management
- Document conditional logic and routing rules
- Plan for future enhancements

### Validation and Testing
- Generate execution traces for user scenarios
- Validate agent behavior against contracts
- Identify architectural gaps or conflicts
- Test conditional invocation logic

### Communication and Collaboration
- Share architectural vision with stakeholders
- Document design decisions and rationale
- Provide implementation teams with clear specifications
- Maintain architectural consistency across development

### Before Implementation
- Validate architectural approach
- Identify edge cases and error conditions
- Refine agent contracts and data structures
- Ensure separation of concerns

## Example User Prompts (for Emulation)
- "Provide me a summary of the configuration best practice assessment, show the top 5 issues."
- "What are the critical security findings for DataCenter-1?"
- "Compare configuration compliance between site A and site B."
- "What's our approved configuration standard for core routers?"

## Next Steps: Implementation

This framework provides the **architectural specification**. To implement:

1. **Choose Framework**: LangGraph, LlamaIndex Workflows, or custom orchestration
2. **Implement Agents**: Use contracts as implementation guides
3. **Build State Manager**: Implement STATE object with task-embedded outputs
4. **Create Graph Flow**: Implement conditional routing and task execution
5. **Integrate Data Sources**: Connect MCP tools, SQL databases, RAG vector stores
6. **Test Against Emulations**: Validate implementation matches emulated behavior

## Contributing to the Architecture

When modifying the framework:
1. Update relevant `.md` files in `/agents/` or `/graph/`
2. Generate emulation trace to verify behavior
3. Update `caveats.md` for architectural decisions
4. Maintain consistency across all contracts
5. Document rationale for changes

---

**Framework Purpose**: Design and validate agentic architectures for assessments app
**Implementation Status**: Architecture complete, ready for implementation  
**Last Updated**: 2026-02-23

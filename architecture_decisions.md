# Architecture Decisions & Future Considerations

This document tracks known limitations in the current architecture and planned enhancements for future phases.
This is a collection of notes captured during the development of this project.

---

## Current Considerations

### 1. No Replanning Mechanism
**Status:** Intentionally removed for initial implementation

**Current Behavior:**
- Planner produces a single, deterministic plan based on intent
- No iteration tracking (`STATE.plan.iteration` removed)
- Graph executes plan: Knowledge Agent (if enterprise context needed) → Data Query Agent (if data required) → Domain Agent

**Implications:**
- If Data Query Agent fails to retrieve required data, domain agents proceed with partial data and qualify findings
- No retry logic for data fetching failures
- No dynamic plan adjustment based on intermediate results

**Future Consideration:**
- Add replanning triggers (e.g., data fetch failures, validator reports insufficient data)
- Implement iteration counter with configurable loop limits
- Define replan strategies based on failure modes

---

### 2. No Multi-Turn Conversation Handling
**Status:** Major caveat, planned for later phase

**Current Behavior:**
- Planner only reads `STATE.intent.*`
- No access to conversation history or previous execution state
- Each invocation is stateless

**Missing Capabilities:**
- Followup questions (e.g., "What about the security posture?")
- Context carryover (e.g., "Now check the same devices for compliance")
- Refinement requests (e.g., "Focus only on firewalls")
- Conversational memory

**Implications:**
- User must provide full context in every request
- Cannot reference previous assessments or findings
- No incremental assessment workflows

**Future Consideration:**
- Add conversation state tracking
- Define followup detection in Intent Classifier
- Implement context carryover mechanisms
- Add `STATE.conversation.*` with history and references

---

### 3. No Stop Conditions
**Status:** Removed as redundant in current model

**Current Behavior:**
- Plan execution stops when all tasks complete or fail
- No explicit stop condition declarations
- Graph execution layer handles termination implicitly

**Implications:**
- Works for simple linear flows
- May need explicit conditions for complex scenarios (parallel branches, optional tasks)

**Future Consideration:**
- Reintroduce at graph orchestration level (not Planner level)
- Define conditions like "proceed with partial data after N attempts"
- Add timeout-based termination for long-running assessments

---

### 4. No State-Based Optimization
**Status:** Simplified for deterministic planning

**Current Behavior:**
- Planner produces conditional task sequence based on requirements (not fixed 3-agent flow)
- Knowledge Agent: Included if strategy/enterprise context needed
- Data Query Agent: Included if domain agent has data dependencies
- Domain Agent(s): Always included to perform assessment
- Does not check if data already exists in `STATE.data.assessment_context`
- Cannot skip Data Query Agent even if all required data is present

**Implications:**
- Minor inefficiency: May fetch data that's already available
- More predictable execution (easier to debug)
- Simpler Planner logic

**Trade-off Accepted:**
- Simplicity > optimization for initial phase
- Data Query Agent can implement its own check for existing data

**Future Consideration:**
- Add state evaluation to conditionally skip agents
- Planner could read `STATE.data.*` and skip Data Query task if data fresh
- Requires defining freshness/staleness criteria for cached data

---

### 5. Single-Pass Execution Model
**Status:** Current design, may need enhancement

**Current Behavior:**
- Plan executes once from start to finish
- No loop-back to earlier stages
- No conditional branching based on intermediate results

**Limitations:**
- Cannot handle "assess then deep-dive" workflows
- No progressive refinement (coarse → detailed)
- No conditional paths (e.g., if security issues found, run compliance check)

**Future Consideration:**
- Add conditional task execution based on domain agent output
- Implement decision nodes in plan structure
- Support for dynamic task injection mid-execution

---

### 6. No Loop Exhaustion Handling
**Status:** Removed with replanning logic

**Current Behavior:**
- No concept of "max attempts" or "give up and proceed"
- If Data Query fails, domain agents run with empty data

**Missing:**
- Graceful degradation after N failed retry attempts
- Explicit "proceed with gaps" decision points
- User notification of partial execution

**Future Consideration:**
- Reintroduce iteration limits (e.g., max 2 replan cycles)
- Define exhaustion behavior: qualify findings, populate `final.missing_inputs[]`
- Add quality thresholds for proceeding vs. aborting

---

### 7. No Assessment Strategy Planning
**Status:** Future enhancement (Phase 2), Knowledge Agent provides enterprise context only

**Current Behavior:**
- Knowledge Agent retrieves enterprise context (policies, standards, known exceptions, compliance requirements)
- Knowledge Agent formulates RAG queries for follow-up questions and policy clarification
- Planner conditionally invokes Knowledge Agent when enterprise context needed
- Domain agents use `STATE.knowledge.enterprise_context` to apply organization-specific rules

**Missing Capabilities:**
- No assessment strategy planning (approach, focus areas, priority order)
- No data acquisition strategy guidance (what to fetch and why)
- Knowledge Agent doesn't guide Planner on assessment methodology
- No adaptive assessment approach based on discovered conditions

**Implications:**
- Planner uses deterministic intent-to-plan mapping (no strategy input)
- Data Query Agent doesn't receive guidance on prioritization
- Domain agents apply generic assessment approaches
- Cannot dynamically adjust focus based on risk profile or enterprise priorities

**Future Consideration (Phase 2):**
- Add `STATE.knowledge.assessment_strategy` field
- Knowledge Agent provides assessment methodology guidance
- Planner uses strategy to inform task creation and ordering
- Data Query Agent uses strategy to prioritize data fetching
- Define when assessment strategy is needed (complex scenarios, ambiguous scope)

---

### 8. Enterprise Context Schema Not Fully Defined
**Status:** Structure partially defined, needs formalization

**Current Behavior:**
- `STATE.knowledge.enterprise_context` - Partially defined with example fields
- `STATE.knowledge.assessment_strategy` - Declared but marked as future (Phase 2)
- Knowledge Agent outputs enterprise_context
- Domain agents consume enterprise_context

**Missing:****
- Formal schema definition for `enterprise_context` (field validation, required vs optional)
- RAG retrieval implementation details (which knowledge base? query format?)
- Examples of populated enterprise_context for different assessment types
- Validation rules for enterprise context content
- Confidence scoring for retrieved information

**Impact:**
- Domain agents need to handle flexible enterprise_context structure
- No consistency guarantees for retrieved information across assessments
- Unclear how to handle missing or incomplete enterprise context

**Next Steps:**
- Formalize `enterprise_context` schema with required/optional fields
- Define RAG integration approach (knowledge base format, retrieval method)
- Add comprehensive examples to Knowledge Agent contract
- Update domain agent contracts with enterprise_context consumption patterns
- Consider versioning for schema evolution

---

### 9. Entity-to-Parameter Mapping as Semantic Matching
**Status:** Current implementation uses semantic matching, may need explicit mapping

**Current Behavior:**
- Intent Classifier extracts generic entity types (site, device_type, timeframe, severity, etc.)
- Data Query Agent semantically matches entities to tool-specific parameters
- Mapping is implicit within agent logic based on tool parameter schemas
- Entity `site` → maps to `site_id`, `assessment_scope`, or `location` depending on tool
- Entity `timeframe` → transforms to `lookback_days`, `start_date`/`end_date`, or date range

**Rationale for Current Approach:**
- Flexible: Agent adapts to different tool schemas
- No maintenance of static mapping tables
- Agent applies context-aware transformations
- Semantic understanding rather than rigid rules

**Potential Issues:**
- Inconsistent mapping across different tools
- Difficult to debug when entity doesn't map correctly
- No explicit documentation of supported entity-parameter combinations
- Hard to validate entity coverage for new tools
- Ambiguity when multiple parameters could match same entity

**Future Consideration:**
- **Option A: Explicit Mapping Registry**
  - Maintain declarative mapping table: entity type → tool parameter mappings
  - Define priority rules when multiple parameters match
  - Document supported entity types per tool
  - Validation: Check if all tool parameters can be satisfied by available entity types
  
- **Option B: Hybrid Approach**
  - Maintain explicit mappings for common patterns
  - Fall back to semantic matching for edge cases
  - Document both explicit mappings and semantic rules
  
- **Option C: Tool Schema Annotation**
  - Annotate tool schemas with entity type hints
  - Example: `assessment_id (entity: assessment_id, required: true)`
  - Agent uses annotations to guide mapping
  - Formalize relationship between entities and parameters in tool definitions

**Trade-offs:**

| Approach | Pros | Cons |
|----------|------|------|
| **Current (Semantic)** | Flexible, no maintenance, adapts to new tools | Opaque logic, potential inconsistency, hard to debug |
| **Explicit Registry** | Clear documentation, consistent, validatable | Maintenance overhead, rigid, doesn't adapt to new patterns |
| **Hybrid** | Balance flexibility and predictability | More complex, need clear rules for which approach when |
| **Schema Annotations** | Self-documenting tools, validation at tool level | Requires tool schema modifications, tooling support needed |

**May Need Revision When:**
- Adding many new MCP tools (consistency becomes critical)
- Debugging entity mapping failures frequently
- Need to validate entity coverage before tool invocation
- Supporting user-defined tools (no semantic knowledge)
- Building entity extraction validation pipeline

---

### 10. Schema and Ontology Management for SQL Path
**Status:** Conceptual definition, implementation details undefined

**Current Behavior:**
- Data Query Agent SQL Path references `STATE.schema` (database structure) and `STATE.ontology` (data semantics)
- Agent contract assumes these are available but doesn't specify how they're populated
- Two-layer approach: structural schema (HOW to query) + semantic ontology (WHAT the data means)

**Missing Specifications:**
- **Schema Source**: Pre-loaded static config, dynamic discovery via information_schema, or hybrid?
- **Ontology Format**: What structure for semantic data dictionary (table/column descriptions, value enumerations)?
- **Schema Scope**: Single database or multi-database support?
- **Schema Updates**: How to handle schema changes (versioning, refresh strategy)?
- **Ontology Maintenance**: Who maintains semantic descriptions? Manual curation or auto-generated?
- **Query Generation**: Text-to-SQL approach (LLM-based with schema+ontology context, template-based, or hybrid)?

**Current Assumptions:**
- `STATE.schema` contains table structures, columns, types, constraints, relationships
- `STATE.ontology` contains semantic meanings:
  - Table descriptions (business purpose, what data represents)
  - Column descriptions (semantic meaning, business context)
  - Value enumerations (valid values and their meanings)
  - Entity-to-column mappings (e.g., `site` entity → `configs.site_id` column)
  - Business rules and data relationships
- Agent combines both to understand user intent and generate semantically correct SQL queries

**Architectural Questions:**

1. **Who populates STATE.schema and STATE.ontology?**
   - Option A: Pre-loaded at system initialization (static configuration)
   - Option B: Data Query Agent retrieves on first SQL Path invocation
   - Option C: Knowledge Agent provides as part of enterprise context
   - Option D: Separate Schema Service populates STATE before agents run

2. **Schema Retrieval Strategy:**
   - Option A: Full schema loaded upfront (complete database catalog)
   - Option B: On-demand schema retrieval (query information_schema per request)
   - Option C: Lazy loading with caching (retrieve first time, cache for session)
   - Option D: Pre-indexed schema metadata service

3. **Ontology Structure:**
   - Option A: Declarative data dictionary (JSON/YAML with table/column descriptions and value enumerations)
   - Option B: Embedded in database metadata (column comments, table descriptions, extended properties)
   - Option C: Separate ontology service with versioning and rich semantic metadata
   - Option D: LLM-inferred semantics from schema introspection and sample data analysis

4. **Query Generation Approach:**
   - Option A: LLM-based text-to-SQL (flexible, requires prompting)
   - Option B: Template-based with parameter substitution (deterministic, limited)
   - Option C: Query builder DSL (programmatic, type-safe)
   - Option D: Hybrid (templates for common patterns, LLM for complex cases)

**Trade-offs:**

| Aspect | Pre-loaded Schema | Dynamic Schema | Ontology Service |
|--------|-------------------|----------------|------------------|
| **Latency** | Fastest (no retrieval) | Slower (query metadata) | Medium (service call) |
| **Freshness** | May be stale | Always current | Depends on sync |
| **Complexity** | Simple agent, complex setup | Complex agent, simple setup | Complex architecture |
| **Adaptability** | Requires reconfig for changes | Auto-adapts to changes | Versioned, controlled |

**Implications:**
- Without ontology, agent cannot understand what data means (e.g., that category='security' represents security findings)
- Without ontology, agent cannot map user entities to appropriate database columns
- Without schema, agent cannot generate syntactically valid SQL
- Query generation quality depends heavily on ontology completeness and accuracy
- Rich semantic descriptions enable better intent-to-query translation
- Schema changes can break queries if not versioned/validated

**Future Considerations:**
- Define STATE.schema and STATE.ontology formal structures
- Specify schema population mechanism (pre-load, dynamic, service)
- Choose ontology format (declarative dictionary, embedded metadata, service, LLM-inferred)
- Choose query generation approach (LLM with schema+ontology context, templates, DSL, hybrid)
- Implement schema versioning and change management
- Add ontology maintenance workflow (manual curation, auto-generation from DB, LLM analysis)
- Define semantic richness requirements (minimal descriptions vs comprehensive data dictionary)
- Consider multiple database support (different schemas, dialects, ontologies)
- Define fallback behavior when schema/ontology incomplete
- Standardize value enumeration format for categorical columns

**Related Decisions:**
- Relates to Caveat #9 (entity-to-parameter mapping for MCP tools)
- Both MCP Path and SQL Path need semantic understanding of entities and data
- Ontology provides the semantic layer connecting user intent to data structures
- Could unify entity semantic understanding across both paths

---

## Architecture Decisions for Future Review

### 1. Planner as Pure Translator
**Current Decision:** Planner only reads `STATE.intent.*`, produces deterministic plan

**Rationale:** 
- Simplicity and predictability
- Clear separation: Planner plans, graph executes
- Easier to test and debug

**May Need Revision When:**
- Adding multi-turn support (need to read conversation state)
- Optimizing for data availability (need to read `STATE.data.*`)
- Implementing adaptive planning (need to read intermediate results)

---

### 2. Conditional Task Creation
**Current Decision:** Planner creates tasks based on actual requirements, not fixed sequence

**Rationale:**
- More accurate planning (only invoke agents that are needed)
- Knowledge Agent: Conditional (if enterprise context needed for follow-ups or policy clarification)
- Data Query Agent: Conditional (if domain agent has data dependencies)
- Domain Agent(s): Always included (performs the assessment)
- Better resource utilization

**May Need Revision When:**
- Need more sophisticated conditional logic (if/else branching)
- Implementing progressive workflows (coarse assessment → detailed dive)
- Adding optimization that requires state evaluation

---

### 3. Task Dependencies as Execution Control
**Current Decision:** `tasks[].depends_on[]` drives execution order

**Rationale:**
- Explicit, declarative dependency graph
- Graph executor can parallelize independent tasks
- Clear failure propagation

**May Need Revision When:**
- Need conditional execution (if/else branches)
- Implementing dynamic task injection
- Adding retry logic with backoff strategies

---

### 4. Data Requirements as Declarations
**Current Decision:** Planner declares requirements without checking availability

**Rationale:**
- Planner is stateless with respect to data
- Data Query Agent handles actual evaluation
- Cleaner separation of concerns

**May Need Revision When:**
- Need to differentiate "data exists but stale" vs "data missing"
- Implementing smart caching strategies
- Optimization requires availability awareness

---

### 5. Data-Task Tracking: Embedded Approach
**Current Decision:** Embed `required_data[]` within each task object

**Rationale:**
- Clear ownership: Each task explicitly declares its data needs
- Handles multiple tasks using same agent (data requirements per invocation)
- Simplifies graph execution (all task information in one place)
- No ambiguity about which task needs which data

**Structure:**
```json
{
  "id": "T3",
  "owner": "Config Best Practice Agent",
  "required_data": [
    {"data_path": "assessment_context.assets.configs", "priority": "required"}
  ]
}
```

**Alternative Approaches Considered:**

**Option A: Task Reference in Separate Array**
- Add `required_for_task: "T3"` field to `STATE.plan.required_data[]`
- Pros: Central view of all data requirements for analysis
- Cons: Duplication of references, potential for orphaned entries if task deleted

**Option B: Agent-Level Tracking Only**
- Use `required_by: "Config Best Practice Agent"` without task reference
- Pros: Simpler structure, works if one task per agent type
- Cons: Ambiguous when multiple tasks use same agent, doesn't scale to complex plans

**May Need Revision When:**
- Need centralized data requirement analysis across all tasks
- Implementing data prefetching optimization (fetch all requirements upfront)
- Building dependency visualization tools that need flat data view

---

### 6. RAG Query Formulation: Knowledge Agent Responsibility
**Current Decision:** Knowledge Agent formulates retrieval queries internally

**Rationale:**
- Knowledge Agent reads `STATE.intent.*` and conversation context directly
- Handles query augmentation for follow-ups within agent logic
- Keeps Planner focused on task orchestration, not RAG specifics
- Agent can apply domain expertise to formulate better retrieval queries

**Current Implementation:**
- Knowledge Agent reads intent and formulates RAG query
- Outputs `STATE.knowledge.retrieval_query` for traceability
- Handles follow-up augmentation when conversation history available
- Example: "What about security?" → "What are security assessment considerations for HQ site?"

**Alternative Approach: Planner Formulates Queries**

**Option A: Planner-Driven Query Formulation**
- Planner reads `STATE.intent.*` and conversation history
- Formulates retrieval query as part of planning process
- Includes query in Knowledge Agent task specification
- Knowledge Agent executes retrieval with provided query

**Structure:**
```json
{
  "id": "T1",
  "owner": "Knowledge Agent",
  "retrieval_query": "What are security assessment best practices for DataCenter-1?",
  "query_context": {
    "intent_class": "security_assessment",
    "scope": ["DataCenter-1"],
    "is_followup": false
  }
}
```

**Trade-offs:**

| Aspect | Knowledge Agent Formulates | Planner Formulates |
|--------|---------------------------|-------------------|
| **Separation of Concerns** | RAG logic in one place | Clear planning/execution split |
| **Context Awareness** | Needs conversation access | Planner already has context |
| **Follow-up Handling** | Agent handles augmentation | Planner augments during planning |
| **Testability** | Test retrieval separately | Test query formulation separately |
| **Complexity** | Agent more complex | Planner more complex |

**May Need Revision When:**
- Implementing advanced query routing (multiple knowledge sources)
- Need centralized query audit trail for all RAG calls
- Planner needs to optimize based on query complexity
- Multi-agent RAG scenarios (different agents, same query)

---

### 7. Task-Output Tracking: Embedded Outputs Approach
**Current Decision:** Agents write outputs to their own task object in `STATE.plan.tasks[]`

**Rationale:**
- Clear provenance: Each task contains both inputs (dependencies, required_data) and outputs
- No data duplication: Task object is single source of truth for agent outputs
- Traceability: All task execution artifacts in one place
- Simplifies state management: No separate STATE.knowledge.*, STATE.data.*, STATE.findings.* sections
- Natural access pattern: Downstream agents read from upstream task outputs via task.depends_on[]

**Current Implementation:**
```json
{
  "id": "T1",
  "owner": "Knowledge Agent",
  "status": "completed",
  "outputs": {
    "enterprise_context": {...},
    "retrieval_query": "..."
  }
}
```

Agents locate their task by: `tasks.find(t => t.owner === "<Agent Name>" && t.status === "in_progress")`

Downstream agents access outputs: `tasks.find(t => t.id === depends_on[0]).outputs`

**Alternative Approaches Considered:**

**Option A: Output References in Tasks**
- Tasks declare `output_refs[]` pointing to STATE sections
- Separate STATE sections still exist (STATE.knowledge.*, STATE.data.*, etc.)
- Tasks contain lightweight references instead of full outputs

Structure:
```json
{
  "id": "T1",
  "owner": "Knowledge Agent",
  "status": "completed",
  "output_refs": ["STATE.knowledge.enterprise_context", "STATE.knowledge.retrieval_query"]
}
```

**Pros:**
- Lightweight task objects
- Separate STATE sections easier to query/filter
- Familiar pattern (similar to traditional STATE management)

**Cons:**
- Data duplication (task refs + STATE sections)
- Potential orphaned data if task deleted
- Consumers need to dereference (read task, then read STATE section)
- Harder to track which task produced which output

**Option B: Enhanced Trace with Task-Output Mapping**
- Tasks remain minimal (no outputs embedded)
- STATE.trace.state_deltas[] tracks task-to-output mapping
- Separate STATE sections still exist

Structure:
```json
STATE.trace.state_deltas[] = [
  {
    "task_id": "T1",
    "agent": "Knowledge Agent",
    "fields_written": ["knowledge.enterprise_context", "knowledge.retrieval_query"],
    "timestamp": "2024-01-15T10:30:00Z"
  }
]
```

**Pros:**
- Clean separation (STATE for current data, trace for provenance)
- Bidirectional traceability (task → outputs, outputs → task)
- Supports audit and debugging
- STATE sections remain for traditional access patterns

**Cons:**
- Requires trace query to link tasks and outputs
- More complex to understand data flow
- Trace becomes essential (not optional)
- Still have separate STATE sections to manage

**May Need Revision When:**
- Need centralized output aggregation (all findings from all tasks)
- Building visualization tools that need flat output views
- Implementing output caching/reuse across assessments
- Task objects become too large (deeply nested outputs)

---

## Validation & Testing Gaps

### 1. No Error Propagation Strategy
- What happens if Knowledge Agent fails?
- Can Data Query proceed without strategy?
- Should domain agents run if data fetch fails?

### 2. No Timeout Definitions
- How long can each agent run before timeout?
- What's the maximum end-to-end assessment duration?
- How to handle stuck/hanging agents?

### 3. No Partial Success Handling
- If 1 of 3 tasks fails, does plan continue or abort?
- Can domain agents produce partial findings?
- How to surface partial results to user?

### 4. No State Rollback Mechanism
- If domain agent crashes mid-execution, is state corrupted?
- Can we revert to last known good state?
- How to handle concurrent state modifications?

---

## Documentation Needed

1. **Graph Execution Layer Specification**
   - How does graph interpret `plan.tasks[]`?
   - Task status lifecycle (pending → in-progress → completed/failed)
   - Failure propagation rules
   - Parallelization policies

2. **Agent Communication Contracts**
   - Knowledge Agent → Data Query Agent handoff
   - Data Query Agent → Domain Agent handoff
   - Domain Agent → FINAL orchestration

3. **State Management Rules**
   - Who can read/write which STATE fields?
   - Conflict resolution for concurrent writes
   - State schema versioning

4. **Error Catalog**
   - Standard error codes for each agent
   - Retry vs. fail-fast policy per error type
   - User-facing error messages

5. **STATE.knowledge.enterprise_context Schema**
   - Formalize schema with required and optional fields
   - Define field types and validation rules
   - Provide examples for each assessment type (configuration, security, compliance, vulnerability)
   - Document how domain agents consume and interpret enterprise_context
   - Define RAG integration approach (knowledge base format, retrieval method)
   - Consider versioning strategy for schema evolution

6. **STATE.knowledge.assessment_strategy Schema (Phase 2)**
   - Define structure for assessment planning guidance (future)
   - Document Planner consumption patterns (future)
   - Examples showing strategy influence on task creation (future)

---

## Next Phase Priorities

**Phase 2: Assessment Strategy Planning**
1. Add `STATE.knowledge.assessment_strategy` field and schema
2. Knowledge Agent generates assessment strategy based on intent and context
3. Planner consults assessment_strategy to inform task creation and ordering
4. Data Query Agent uses strategy to prioritize data fetching
5. Define criteria for when assessment strategy is needed

**Phase 3: Multi-Turn Support**
1. Add conversation state tracking
2. Update Intent Classifier to detect followups
3. Define context carryover mechanism
4. Enhance Knowledge Agent query augmentation with conversation history
5. Implement incremental assessment workflows

**Phase 4: Replanning & Resilience**
1. Add iteration tracking back to Planner
2. Define replan triggers
3. Implement loop limits with exhaustion handling
4. Add timeout and retry policies

**Phase 5: Optimization**
1. State-based task generation (skip Data Query if data exists)
2. Parallel task execution where possible
3. Caching strategies for Knowledge Agent output
4. Progressive assessment (coarse → detailed)

**Phase 6: Advanced Workflows**
1. Conditional branching based on intermediate results
2. Dynamic task injection mid-execution
3. Cross-assessment correlation
4. Scheduled/automated assessments

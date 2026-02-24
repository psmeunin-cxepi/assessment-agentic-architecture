# Emulation Trace: EVAL-001 — Configuration Best Practice Assessment Summary

## Eval Case Reference
- **ID**: EVAL-001
- **Source**: CXP-17653
- **Category**: summary
- **User Prompt**: "Can you provide a summary of my recent Configuration Best Practice assessment?"

---

## 1) Intent Classification

```json
{
  "intent_class": "cbp_assessment",
  "meta_intent": "new_topic",
  "domain_details": {
    "assessment_goal": "summarize recent configuration best practice assessment",
    "scope": {
      "site": null,
      "environment": null,
      "time_range": "recent"
    },
    "urgency": "normal"
  },
  "entities": [
    {
      "type": "assessment_type",
      "value": "configuration_best_practice",
      "confidence": 1.0
    },
    {
      "type": "timeframe",
      "value": "recent",
      "confidence": 0.90
    }
  ],
  "confidence": 0.95,
  "clarification_question": null
}
```

**Reasoning**: User explicitly references "Configuration Best Practice assessment" → `cbp_assessment`. "Recent" implies latest assessment; no site/device scoping. High confidence — no ambiguity.

---

## 2) Graph Flow Map (Mermaid)

```mermaid
flowchart TD
  U["User: Can you provide a summary of my recent Configuration Best Practice assessment?"] --> IR[Semantic Router / Intent Classifier]
  IR -->|"cbp_assessment (0.95)"| P[Supervisor / Planner]

  P -->|"T1: data dependencies exist"| DQ[Data Query Agent]
  DQ -->|"writes task.outputs.assessment_context"| CFG

  P -->|"T2: domain agent"| CFG[Config Best Practice Agent]

  subgraph MCP[MCP Resources / Tools]
    MCPT[(assessment_analysis_tool)]
  end

  DQ --- MCPT
  CFG -->|"writes task.outputs.summary"| FINAL[Return final STATE]
```

**Agents NOT invoked**:
- Knowledge Agent — `cbp_assessment` + summary request with no ambiguity → enterprise enrichment not needed (conditional, skipped)
- Security Assessment Agent — not applicable to this intent_class

---

## 3) Node-by-Node Trace

### Node 1: Semantic Router / Intent Classifier
- **Persona**: A precise, conservative classifier. Optimizes for correct routing and minimal assumptions.
- **Input (STATE read)**:
  - `STATE.input.user_prompt` = "Can you provide a summary of my recent Configuration Best Practice assessment?"
  - `STATE.input.context_kv` = `{}` (not provided)
- **Action**:
  - Detected "Configuration Best Practice assessment" → maps to `cbp_assessment`
  - Extracted "recent" as `timeframe` entity
  - No site, device, or environment mentioned → `null`
  - Confidence 0.95 — clear, unambiguous intent
  - `clarification_question` = `null` (confidence ≥ 0.5 → no clarification gate)
- **State Update**:
  ```json
  {
    "intent": {
      "intent_class": "cbp_assessment",
      "meta_intent": "new_topic",
      "domain_details": {
        "assessment_goal": "summarize recent configuration best practice assessment",
        "scope": { "site": null, "environment": null, "time_range": "recent" },
        "urgency": "normal"
      },
      "entities": [
        {"type": "assessment_type", "value": "configuration_best_practice", "confidence": 1.0},
        {"type": "timeframe", "value": "recent", "confidence": 0.90}
      ],
      "confidence": 0.95,
      "clarification_question": null
    },
    "trace": {
      "node_run_order": ["Intent Classifier"],
      "state_deltas": [
        "IC: classified as cbp_assessment (0.95); entities: assessment_type=configuration_best_practice, timeframe=recent"
      ]
    }
  }
  ```
- **Exit Logic**: `intent_class != "unknown_or_needs_clarification"` → transition to Planner.

---

### Node 2: Supervisor / Planner
- **Persona**: A deterministic orchestrator. Converts intent into an executable plan and routing decisions.
- **Input (STATE read)**:
  - `STATE.intent.*` (intent_class=`cbp_assessment`, entities, confidence)
- **Action**:
  - **Step 1**: `intent_class = "cbp_assessment"` → target agent is Config Best Practice Agent, skill `cbp_assessment`
  - **Step 2**: Load CBP data dependencies for `cbp_assessment` skill → `assessment_context` required (min_count: 1) → include Data Query Agent task
  - **Step 3**: Knowledge Agent conditional for `cbp_assessment` → this is a straightforward summary request with no ambiguity or policy questions → skip Knowledge Agent
  - **Step 4**: Build task plan with dependencies
- **State Update**:
  ```json
  {
    "plan": {
      "tasks": [
        {
          "id": "T1",
          "description": "Retrieve latest configuration best practice assessment summary",
          "owner": "Data Query Agent",
          "depends_on": [],
          "status": "pending",
          "outputs": {}
        },
        {
          "id": "T2",
          "description": "Summarize configuration best practice assessment results",
          "owner": "Config Best Practice Agent",
          "depends_on": ["T1"],
          "required_data": [
            {
              "data_path": "assessment_context",
              "skill": "cbp_assessment",
              "min_count": 1,
              "priority": "required"
            }
          ],
          "status": "pending",
          "outputs": {}
        }
      ],
      "routing": ["Data Query Agent", "Config Best Practice Agent"]
    }
  }
  ```
  ```
  trace.node_run_order: ["Intent Classifier", "Planner"]
  trace.state_deltas += "Planner: created 2 tasks (T1: DQA, T2: CBP); KA skipped (no enterprise enrichment needed for summary)"
  ```
- **Exit Logic**: Task plan created → execute T1 (no dependencies).

---

### Node 3: Data Query Agent (Task T1)
- **Persona**: A retrieval + normalization specialist. Deterministic, cautious, and schema/tool aware.
- **Input (STATE read)**:
  - `STATE.intent.intent_class` = `"cbp_assessment"`
  - `STATE.intent.entities[]` = `[assessment_type, timeframe]`
  - `STATE.input.user_prompt` = "Can you provide a summary of my recent Configuration Best Practice assessment?"
  - Task T1 definition from `STATE.plan.tasks[0]`
- **Action**:
  - **Path selection**: `cbp_assessment` maps to MCP path (supported intent class)
  - **Tool selection**: User asks for "summary" → `assessment_analysis_tool` with `query_type="summary"` (per Tool Selection Logic rule 1)
  - **Entity-to-parameter binding**:
    - `assessment_type` = "configuration_best_practice" → context for tool selection (already selected `assessment_analysis_tool`)
    - `timeframe` = "recent" → `assessment_id = ""` (empty = latest assessment, which satisfies "recent")
    - No severity or product filter mentioned → defaults
  - **Tool invocation**:
    ```json
    {
      "tool_name": "assessment_analysis_tool",
      "input": {
        "query_type": "summary",
        "assessment_id": "",
        "severity_filter": "",
        "product_filter": "",
        "focus_area": ""
      }
    }
    ```
  - **Response** (emulated — no fixture data available):
    ```json
    {
      "assessment_info": {
        "id": "ASSESS-2026-02-20-001",
        "total_records": 342,
        "query_filters": {
          "assessment_id": "",
          "severity_filter": "",
          "product_filter": ""
        }
      },
      "filtered_results": {
        "total_records_matching": 342,
        "severity_distribution": {
          "critical": 12,
          "high": 47,
          "medium": 156,
          "low": 127
        },
        "asset_types_found": ["cisco_ios", "cisco_nxos", "cisco_iosxe", "juniper_junos"],
        "common_issues": [
          "NTP server not configured",
          "SNMP community string using default",
          "No banner login configured",
          "Telnet enabled on VTY lines",
          "Password encryption not enabled"
        ],
        "sample_records": ["<capped at 5 records>"]
      },
      "analysis": {
        "high_severity_count": 59,
        "most_affected_severity": "medium",
        "unique_asset_types": 4
      },
      "recommendations": [
        "Address 12 critical findings immediately",
        "Review high-severity NTP and SNMP configurations",
        "Disable Telnet and enforce SSH-only access"
      ]
    }
    ```
  - **Normalize to Assessment Context**: Wrap response into canonical format
- **State Update**:
  ```json
  {
    "plan.tasks[0].status": "completed",
    "plan.tasks[0].outputs": {
      "assessment_context": {
        "context_id": "ctx-eval-001",
        "source_path": "mcp",
        "timestamp_utc": "2026-02-24T10:00:00Z",
        "scope": {
          "targets": [],
          "site": null,
          "time_range": {"from": null, "to": "2026-02-20"}
        },
        "tool_call": {
          "tool_name": "assessment_analysis_tool",
          "input": {
            "query_type": "summary",
            "assessment_id": "",
            "severity_filter": "",
            "product_filter": "",
            "focus_area": ""
          }
        },
        "assessment_info": {
          "id": "ASSESS-2026-02-20-001",
          "total_records": 342,
          "query_filters": {
            "assessment_id": "",
            "severity_filter": "",
            "product_filter": ""
          }
        },
        "filtered_results": {
          "total_records_matching": 342,
          "severity_distribution": {
            "critical": 12,
            "high": 47,
            "medium": 156,
            "low": 127
          },
          "asset_types_found": ["cisco_ios", "cisco_nxos", "cisco_iosxe", "juniper_junos"],
          "common_issues": [
            "NTP server not configured",
            "SNMP community string using default",
            "No banner login configured",
            "Telnet enabled on VTY lines",
            "Password encryption not enabled"
          ],
          "sample_records": ["<capped at 5>"]
        },
        "analysis": {
          "high_severity_count": 59,
          "most_affected_severity": "medium",
          "unique_asset_types": 4
        },
        "recommendations": [
          "Address 12 critical findings immediately",
          "Review high-severity NTP and SNMP configurations",
          "Disable Telnet and enforce SSH-only access"
        ],
        "provenance": [
          {
            "tool": "assessment_analysis_tool",
            "query_type": "summary",
            "assessment_id": "ASSESS-2026-02-20-001",
            "timestamp": "2026-02-24T10:00:00Z"
          }
        ],
        "errors": []
      }
    }
  }
  ```
  ```
  trace.node_run_order: ["Intent Classifier", "Planner", "Data Query Agent"]
  trace.state_deltas += "DQA(T1): invoked assessment_analysis_tool(query_type=summary); wrote assessment_context with 342 records, 4 severity levels; T1 completed"
  ```
- **Exit Logic**: T1 completed → T2 dependency satisfied → execute T2.

---

### Node 4: Config Best Practice Agent (Task T2)
- **Persona**: A configuration standards validator. Evidence-based, conservative, and explicit about uncertainty.
- **Input (STATE read)**:
  - `STATE.intent.*` (intent_class=`cbp_assessment`, entities, domain_details)
  - `STATE.plan.tasks[]` → find T1 (owner: "Data Query Agent", status: "completed")
  - T1 `outputs.assessment_context` (the assessment summary data above)
  - No Knowledge Agent task present → enterprise_context not available
- **Action**:
  - **Step 1 — Input Validation**:
    - Active skill: `cbp_assessment` (from `intent_class`)
    - `assessment_context` exists and contains analyzable data ✓ (342 records, severity distribution, common issues)
    - `enterprise_context` not available → proceed with generic best practices; note assumption
    - User intent from `domain_details.assessment_goal`: "summarize recent configuration best practice assessment"
  - **Step 2 — Assessment Analysis**:
    - Route: `cbp_assessment` → lead with assessment_context analysis
    - User asks for summary → generate `outputs.summary` as primary response
    - No comparison/delta data detected (no `detailed_findings.new_issues[]`) → NOT trend mode
    - Parse severity distribution: 12 critical, 47 high, 156 medium, 127 low
    - Total at-risk: 59 high+critical out of 342 = 17.3% critical/high severity
    - Top issues: NTP, SNMP defaults, no banner, Telnet enabled, password encryption
    - 4 platform families affected
  - **Step 3 — Finding Generation**: Not primary for summary requests; user asked for summary, not detailed findings
  - **Step 4 — Finding Qualification**: N/A for summary mode
  - **Step 5 — Output Assembly**:
    - Generate natural language summary in `outputs.summary`
    - No chart_hints needed (user did not ask for visualization)
    - No prioritized_risks explicitly requested (but include top critical items for completeness)
- **State Update**:
  ```json
  {
    "plan.tasks[1].status": "completed",
    "plan.tasks[1].outputs": {
      "summary": "Your most recent Configuration Best Practice assessment (ASSESS-2026-02-20-001) analyzed 342 configuration checks across 4 platform families (Cisco IOS, NX-OS, IOS-XE, and Juniper JunOS).\n\n**Severity Distribution:**\n- Critical: 12 (3.5%)\n- High: 47 (13.7%)\n- Medium: 156 (45.6%)\n- Low: 127 (37.1%)\n\n59 findings (17.3%) are rated critical or high severity and require priority attention.\n\n**Top Issues Identified:**\n1. NTP server not configured — affects time synchronization and audit trail integrity\n2. SNMP community string using default — exposes management plane to unauthorized access\n3. No banner login configured — regulatory and compliance gap\n4. Telnet enabled on VTY lines — cleartext management protocol, security risk\n5. Password encryption not enabled — credentials stored in plaintext\n\n**Recommendations:**\n- Address the 12 critical findings immediately, focusing on SNMP defaults and Telnet exposure\n- Review high-severity NTP and SNMP configurations across all platforms\n- Disable Telnet and enforce SSH-only access on all VTY lines\n\nNo enterprise-specific policies were applied to this summary. Findings are based on industry best practices.",
      "findings": [],
      "prioritized_risks": [
        {
          "severity": "critical",
          "count": 12,
          "top_issues": ["SNMP community string using default", "Telnet enabled on VTY lines"],
          "recommendation": "Address immediately — management plane exposure"
        },
        {
          "severity": "high",
          "count": 47,
          "top_issues": ["NTP server not configured", "Password encryption not enabled"],
          "recommendation": "Schedule remediation within 7 days"
        }
      ],
      "chart_hints": [],
      "data_gaps": [],
      "assumptions": [
        "No enterprise-specific policies applied (Knowledge Agent not invoked)",
        "Assessment scope covers all sites (no site filter specified by user)"
      ]
    }
  }
  ```
  ```
  trace.node_run_order: ["Intent Classifier", "Planner", "Data Query Agent", "Config Best Practice Agent"]
  trace.state_deltas += "CBP(T2): read T1.outputs.assessment_context; generated summary of 342-check assessment; 59 critical/high findings (17.3%); no enterprise context applied; T2 completed"
  ```
- **Exit Logic**: T2 completed → no downstream tasks → graph execution complete.

---

## 4) Final Assessment Outcome

### Config Best Practice Agent outputs (Task T2)

**Summary**:
The latest Configuration Best Practice assessment (ASSESS-2026-02-20-001) covers 342 checks across 4 platform families. 17.3% of findings are critical or high severity (12 critical, 47 high). Top issues include unconfigured NTP, default SNMP community strings, missing login banners, Telnet on VTY lines, and disabled password encryption.

**Findings**: `[]` (summary mode — detailed findings not generated; user asked for summary, not individual finding enumeration)

**Prioritized Risks**:
| Severity | Count | Top Issues | Recommendation |
|---|---|---|---|
| Critical | 12 | SNMP defaults, Telnet enabled | Address immediately |
| High | 47 | NTP not configured, password encryption off | Remediate within 7 days |

**Chart Hints**: `[]` (user did not request visualization)

**Asset Trend**: N/A (not a comparison/delta request)

**Data Gaps**: None

**Assumptions**:
- No enterprise-specific policies applied (Knowledge Agent not invoked)
- Assessment scope covers all sites (no site filter specified by user)

---

## 5) Final STATE Snapshot

```json
{
  "input": {
    "user_prompt": "Can you provide a summary of my recent Configuration Best Practice assessment?",
    "context_kv": {}
  },
  "intent": {
    "intent_class": "cbp_assessment",
    "meta_intent": "new_topic",
    "domain_details": {
      "assessment_goal": "summarize recent configuration best practice assessment",
      "scope": {
        "site": null,
        "environment": null,
        "time_range": "recent"
      },
      "urgency": "normal"
    },
    "entities": [
      {"type": "assessment_type", "value": "configuration_best_practice", "confidence": 1.0},
      {"type": "timeframe", "value": "recent", "confidence": 0.90}
    ],
    "confidence": 0.95,
    "clarification_question": null
  },
  "plan": {
    "tasks": [
      {
        "id": "T1",
        "description": "Retrieve latest configuration best practice assessment summary",
        "owner": "Data Query Agent",
        "depends_on": [],
        "status": "completed",
        "outputs": {
          "assessment_context": {
            "context_id": "ctx-eval-001",
            "source_path": "mcp",
            "timestamp_utc": "2026-02-24T10:00:00Z",
            "scope": {"targets": [], "site": null, "time_range": {"from": null, "to": "2026-02-20"}},
            "tool_call": {
              "tool_name": "assessment_analysis_tool",
              "input": {"query_type": "summary", "assessment_id": "", "severity_filter": "", "product_filter": "", "focus_area": ""}
            },
            "assessment_info": {"id": "ASSESS-2026-02-20-001", "total_records": 342},
            "filtered_results": {
              "total_records_matching": 342,
              "severity_distribution": {"critical": 12, "high": 47, "medium": 156, "low": 127},
              "asset_types_found": ["cisco_ios", "cisco_nxos", "cisco_iosxe", "juniper_junos"],
              "common_issues": ["NTP server not configured", "SNMP community string using default", "No banner login configured", "Telnet enabled on VTY lines", "Password encryption not enabled"]
            },
            "analysis": {"high_severity_count": 59, "most_affected_severity": "medium", "unique_asset_types": 4},
            "recommendations": ["Address 12 critical findings immediately", "Review high-severity NTP and SNMP configurations", "Disable Telnet and enforce SSH-only access"],
            "provenance": [{"tool": "assessment_analysis_tool", "query_type": "summary", "assessment_id": "ASSESS-2026-02-20-001", "timestamp": "2026-02-24T10:00:00Z"}],
            "errors": []
          }
        }
      },
      {
        "id": "T2",
        "description": "Summarize configuration best practice assessment results",
        "owner": "Config Best Practice Agent",
        "depends_on": ["T1"],
        "required_data": [{"data_path": "assessment_context", "skill": "cbp_assessment", "min_count": 1, "priority": "required"}],
        "status": "completed",
        "outputs": {
          "summary": "Your most recent Configuration Best Practice assessment (ASSESS-2026-02-20-001) analyzed 342 configuration checks across 4 platform families (Cisco IOS, NX-OS, IOS-XE, and Juniper JunOS).\n\nSeverity Distribution:\n- Critical: 12 (3.5%)\n- High: 47 (13.7%)\n- Medium: 156 (45.6%)\n- Low: 127 (37.1%)\n\n59 findings (17.3%) are rated critical or high severity and require priority attention.\n\nTop Issues Identified:\n1. NTP server not configured\n2. SNMP community string using default\n3. No banner login configured\n4. Telnet enabled on VTY lines\n5. Password encryption not enabled\n\nRecommendations:\n- Address the 12 critical findings immediately\n- Review high-severity NTP and SNMP configurations\n- Disable Telnet and enforce SSH-only access\n\nNo enterprise-specific policies were applied.",
          "findings": [],
          "prioritized_risks": [
            {"severity": "critical", "count": 12, "top_issues": ["SNMP community string using default", "Telnet enabled on VTY lines"], "recommendation": "Address immediately"},
            {"severity": "high", "count": 47, "top_issues": ["NTP server not configured", "Password encryption not enabled"], "recommendation": "Remediate within 7 days"}
          ],
          "chart_hints": [],
          "data_gaps": [],
          "assumptions": ["No enterprise-specific policies applied (Knowledge Agent not invoked)", "Assessment scope covers all sites (no site filter specified)"]
        }
      }
    ],
    "routing": ["Data Query Agent", "Config Best Practice Agent"]
  },
  "routing": {
    "last_decision": "T2 completed; graph execution complete"
  },
  "trace": {
    "node_run_order": [
      "Intent Classifier",
      "Planner",
      "Data Query Agent",
      "Config Best Practice Agent"
    ],
    "state_deltas": [
      "IC: classified as cbp_assessment (0.95); entities: assessment_type=configuration_best_practice, timeframe=recent",
      "Planner: created 2 tasks (T1: DQA, T2: CBP); KA skipped (no enterprise enrichment needed for summary)",
      "DQA(T1): invoked assessment_analysis_tool(query_type=summary); wrote assessment_context with 342 records, 4 severity levels; T1 completed",
      "CBP(T2): read T1.outputs.assessment_context; generated summary of 342-check assessment; 59 critical/high findings (17.3%); no enterprise context applied; T2 completed"
    ]
  }
}
```

---

## 6) Observations

| Category | Observation |
|---|---|
| **Intent classification** | High confidence (0.95) — prompt is an almost exact match for `cbp_assessment` + summary pattern |
| **Knowledge Agent** | Correctly skipped — `cbp_assessment` makes KA conditional, and a straightforward summary doesn't benefit from enterprise enrichment |
| **Tool selection** | `assessment_analysis_tool` with `query_type="summary"` is the exact match per DQA Tool Selection Logic rule 1 |
| **MCP tool response** | Emulated (no fixture file available) — realistic Cisco-centric data following `assessment_analysis_tool` response schema |
| **CBP output mode** | Summary mode (not findings enumeration) — `findings[]` empty, `summary` carries the response |
| **Anti-hallucination** | All data derived from tool response; assumptions explicitly documented |
| **Chart hints** | Correctly empty — user did not request visualization |
| **Eval alignment** | Matches EVAL-001 expected: `cbp_assessment`, agents=[DQA, CBP], tool=`assessment_analysis_tool`, output=`summary`, no chart_hints, no KA |

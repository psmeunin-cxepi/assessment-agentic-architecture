# issue_tracking_tool

## Source
`/mcp_server/server.py` — registered via `@mcp.tool()`
Internal service: `tools_service.get_unresolved_issues()` — called for all `tracking_scope` values.

## Purpose
Report on unresolved configuration compliance issues and persistently failing rules across assessments. Supports scoped filtering by site/environment, rule category, and minimum severity priority.

## Signature
```python
@mcp.tool()
def issue_tracking_tool(
    tracking_scope: str = "compliance",
    assessment_scope: str = "all",
    rule_filter: str = "",
    priority: str = "high",
) -> str
```

## Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `tracking_scope` | str | No | `"compliance"` | Logical focus of tracking. Values: `"compliance"`, `"rules"`, `"persistent"`, `"all"`. All values call the same internal service — `tracking_scope` is a semantic label only. |
| `assessment_scope` | str | No | `"all"` | Site name, environment, or `"all"` to include everything |
| `rule_filter` | str | No | `""` | Filter by rule ID or category, e.g. `"AUTH"`, `"ENCRYPTION"` |
| `priority` | str | No | `"high"` | Minimum severity threshold: `"critical"`, `"high"`, `"medium"`, `"low"` |

**Note on `tracking_scope`**: All four values (`"compliance"`, `"rules"`, `"persistent"`, `"all"`) route to the same `get_unresolved_issues` service call. The value is used by the calling agent as semantic context — it does not change the query logic.

## Response Schema
```json
{
  "unresolved_issues_summary": {
    "total_unresolved": "number",
    "scope": "string",
    "priority_filter": "string",
    "rule_filter": "string",
    "affected_assets_count": "number"
  },
  "severity_breakdown": {
    "critical": "number",
    "high": "number",
    "medium": "number",
    "low": "number"
  },
  "most_persistent_rules": [
    {
      "rule_name": "string",
      "occurrence_count": "number",
      "sample_issues": []
    }
  ],
  "sample_unresolved_issues": [],
  "recommendations": ["string"],
  "risk_assessment": {
    "risk_level": "high|medium|low",
    "critical_issues": "number",
    "high_issues": "number",
    "total_unresolved": "number",
    "high_severity_ratio": "number",
    "description": "string"
  }
}
```

## Response Limits

| Field | Cap |
|---|---|
| `sample_unresolved_issues[]` | **15 records** |
| `most_persistent_rules[]` | **10 rules** |
| `most_persistent_rules[].sample_issues[]` | **3 issues per rule** |

## Use When
- User requests open issues, action items, or compliance gaps
- Identifying which rules persistently fail across assets
- Prioritising remediation — what needs to be fixed first
- Scoped issue queries: "unresolved issues in production", "open AUTH failures"

## Do Not Use When
- Comparing two assessments for trend/delta → use `assessment_comparison_tool`
- Single-assessment summary or statistics → use `assessment_analysis_tool`

## Tool Call Record Example
```json
{
  "tool_name": "issue_tracking_tool",
  "input": {
    "tracking_scope": "compliance",
    "assessment_scope": "production",
    "rule_filter": "AUTH",
    "priority": "high"
  }
}
```

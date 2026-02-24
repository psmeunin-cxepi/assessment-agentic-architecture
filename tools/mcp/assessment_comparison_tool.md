# assessment_comparison_tool

## Source
`/mcp_server/server.py` — registered via `@mcp.tool()`
Internal service: `tools_service.compare_assessments()`

## Purpose
Compare a current assessment against a previous one. Produces asset-level change tracking, severity trend analysis, and new/resolved/persistent issue identification.

## Signature
```python
@mcp.tool()
def assessment_comparison_tool(
    current_assessment_id: str,
    previous_assessment_id: str = "",
    comparison_focus: str = "assets",
) -> str
```

## Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `current_assessment_id` | str | **Yes** | — | The current (newer) assessment ID to analyse |
| `previous_assessment_id` | str | No | `""` | The prior assessment ID to compare against. If empty, uses most recent prior assessment. |
| `comparison_focus` | str | No | `"assets"` | Focus of comparison. Values: `"assets"`, `"overall"`, `"changes"`. Internally maps to `asset_focus=True` when `"assets"`. |

## comparison_focus Behaviour

| `comparison_focus` | `asset_focus` passed internally | Effect |
|---|---|---|
| `"assets"` | `True` | `asset_level_changes` is populated with per-asset improved/worsened lists |
| `"overall"` | `False` | `asset_level_changes` is `{}` — trend and metrics only |
| `"changes"` | `False` | `asset_level_changes` is `{}` — new/resolved/persistent issues focus |

## Response Schema
```json
{
  "comparison_info": {
    "current": { "id": "string", "total_records": "number", "timestamp": "string" },
    "previous": { "id": "string", "total_records": "number", "timestamp": "string" }
  },
  "metrics_comparison": {
    "severity_changes": {},
    "record_count_change": "number",
    "asset_type_changes": {
      "new_asset_types": ["string"],
      "removed_asset_types": ["string"],
      "common_asset_types": ["string"]
    }
  },
  "asset_level_changes": {
    "improved_assets": [{ "asset_id": "string", "issues_reduced": "number" }],
    "worsened_assets": [{ "asset_id": "string", "issues_increased": "number" }],
    "new_assets": ["string"],
    "removed_assets": ["string"],
    "summary": "string"
  },
  "trend_analysis": {
    "total_records_trend": "increasing|decreasing|stable",
    "critical_issues_trend": "increasing|decreasing|stable",
    "overall_assessment": "improving|needs_attention",
    "record_count_change": "number",
    "critical_change": "number",
    "high_change": "number"
  },
  "detailed_findings": {
    "new_issues": [],
    "resolved_issues": [],
    "persistent_issues": []
  }
}
```

## detailed_findings Array Shapes

All three arrays are **capped at 10 records**.

| Array | Record shape |
|---|---|
| `new_issues[]` | Full Trino record: `{ asset_id, rule_id, severity, ... }` |
| `resolved_issues[]` | Full Trino record: `{ asset_id, rule_id, severity, ... }` |
| `persistent_issues[]` | `{ current: {}, previous: {}, identifier: "asset_id_rule_id" }` |

## Use When
- User asks what changed since the last assessment
- Identifying which assets improved or worsened
- Surfacing new, resolved, or persistent deviations
- Trend queries: "are we improving?"

## Do Not Use When
- Single-assessment analysis → use `assessment_analysis_tool`
- Open issue tracking without comparison → use `issue_tracking_tool`

## Tool Call Record Example
```json
{
  "tool_name": "assessment_comparison_tool",
  "input": {
    "current_assessment_id": "ASSESS-2026-002",
    "previous_assessment_id": "ASSESS-2026-001",
    "comparison_focus": "assets"
  }
}
```

# assessment_analysis_tool

## Source
`/mcp_server/server.py` — registered via `@mcp.tool()`
Internal service: `tools_service.get_assessment_summary()` / `get_assessment_details()`

## Purpose
Comprehensive current assessment analysis. Handles summary, filtering, asset-level, technology/platform, and exception queries against the latest or a specific assessment.

## Signature
```python
@mcp.tool()
def assessment_analysis_tool(
    query_type: str = "summary",
    assessment_id: str = "",
    severity_filter: str = "",
    product_filter: str = "",
    focus_area: str = "",
) -> str
```

## Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `query_type` | str | No | `"summary"` | Analysis mode. Values: `"summary"`, `"filtering"`, `"assets"`, `"technology"`, `"exceptions"` |
| `assessment_id` | str | No | `""` | Specific assessment ID. If empty, returns latest assessment. |
| `severity_filter` | str | No | `""` | Filter by severity: `"critical"`, `"high"`, `"medium"`, `"low"` |
| `product_filter` | str | No | `""` | Filter by product/platform, e.g. `"cisco_ios"`, `"juniper_junos"` |
| `focus_area` | str | No | `""` | Reserved for future use |

## query_type Routing

| `query_type` | Internal calls | Response shape |
|---|---|---|
| `"summary"` | `get_assessment_summary` only | JSON summary object |
| `"filtering"` | `get_assessment_summary` only | JSON summary object (with filters applied) |
| `"assets"` | `get_assessment_summary` + `get_assessment_details` | `"SUMMARY:\n...\n\nDETAILS:\n..."` combined string |
| `"technology"` | `get_assessment_summary` + `get_assessment_details` | Combined string |
| `"exceptions"` | `get_assessment_summary` + `get_assessment_details` | Combined string |

## Response Schema (summary / filtering path)
```json
{
  "assessment_info": {
    "id": "string",
    "total_records": "number",
    "query_filters": {
      "assessment_id": "string",
      "severity_filter": "string",
      "product_filter": "string"
    }
  },
  "filtered_results": {
    "total_records_matching": "number",
    "severity_distribution": {
      "critical": "number",
      "high": "number",
      "medium": "number",
      "low": "number"
    },
    "asset_types_found": ["string"],
    "common_issues": ["string"],
    "sample_records": []
  },
  "analysis": {
    "high_severity_count": "number",
    "most_affected_severity": "string",
    "unique_asset_types": "number"
  },
  "recommendations": ["string"]
}
```

**Note**: `sample_records` is capped at **5 records**.

## Use When
- Assessment overview, summary narrative, statistics (checks performed/failed, % at risk)
- Severity-filtered results (`query_type="filtering"`)
- Asset-level breakdown or most-affected assets (`query_type="assets"`)
- Technology/product-family/OS analysis (`query_type="technology"`)
- Exception detail queries (`query_type="exceptions"`)

## Do Not Use When
- Comparing two assessments → use `assessment_comparison_tool`
- Tracking unresolved/persistent issues → use `issue_tracking_tool`

## Tool Call Record Example
```json
{
  "tool_name": "assessment_analysis_tool",
  "input": {
    "query_type": "filtering",
    "assessment_id": "ASSESS-2026-001",
    "severity_filter": "high",
    "product_filter": ""
  }
}
```

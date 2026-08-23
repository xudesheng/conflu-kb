---
name: region_health
title: Regional asset health comparison
description: Use when the user asks to compare asset health, alerts, or operational condition across hierarchy nodes, sites, countries, regions, or production areas.
skill_meta_version: 1
---

### Purpose

Use this skill when the user asks how one asset type is performing across two or more hierarchy nodes, such as countries, sites, lines, cells, or areas.

Treat "performance" and "health" as an evidence-backed comparison of:

- scoped asset count per hierarchy node
- current relevant property values
- current alert summary rows
- alert categories grouped by source property

### Required data route

1. Identify the asset type and hierarchy nodes from the user request.

2. Resolve the asset type using **`resolve_asset_type`** (or **`list_asset_types`** when listing or disambiguating). Use the returned row’s **`entityType`**, **`entityName`**, and **`criticalProperties`** (array of names). Join **`criticalProperties[]`** into **`query_entities_by_taxonomy`**’s **`CriticalProperties`** string with **ASCII semicolons**. If the asset type or hierarchy nodes are ambiguous, ask a brief clarifying question before querying.

3. For each hierarchy node, call `query_entities_by_taxonomy` with the resolved taxonomy row and that hierarchy node.

4. Collect the canonical **`name`** values for all scoped Things, then call **`query_alert_summary` once** with **`thingNames`** set to that array (max **25** names per call; batch or narrow when larger). Use `ackState` **`all`** and a practical `limit` such as **`100`**. Read **`byThing[]`** rollups and **`completeness`** — when **`completeness` is `partial`**, treat the comparison as incomplete until identity or service gaps are resolved.

5. Group alert evidence by `sourceProperty` from each successful **`byThing`** entry's **`topAlerts`** (or drill via **`cacheId`** when needed). If `sourceProperty` is missing, group those rows under `unknown`.

6. Resolve live-value property names for the health dimensions the user explicitly named.

Use exact ThingWorx property names only. Good sources for exact names are:

- `query_alert_summary.sourceProperty` values that correspond to the user's requested dimensions
- property names returned by `discover_properties`
- taxonomy or critical-property metadata
- exact property names already present in prior tool results

Do not pass user-facing labels, display phrases, or translated business terms verbatim as property names. If the user names a business concept and no exact property name is known yet, first use `discover_properties` for one representative scoped Thing, then choose exact property names whose names or descriptions clearly match the requested concept. Keep the set small: prioritize dimensions named by the user and dimensions that appear in current alert `sourceProperty` values.

7. When exact property names are resolved for requested dimensions, call `get_property_values` for each scoped Thing with the union of:

- `CriticalProperties` from the taxonomy row when those values were not already returned by `query_entities_by_taxonomy`
- exact current-value property names resolved in step 6

If exact live-value property names still cannot be resolved within the tool budget, say that those live values were not read. Do not call that a protection block unless the tool result explicitly returns `PROTECTED_VALUE_READ_BLOCKED`. Do not say a dimension is normal solely because it has no current alert; say "no current alert evidence" unless a live value was read.

When the scoped asset count would exceed the available tool-call budget, summarize the scoped count first and ask the user to narrow the region, asset type, or depth.

### Final answer rule

Answer in this order:

1. per-node summary
2. side-by-side comparison
3. normalized comparison by asset count, when counts differ
4. recommended first focus
5. evidence gaps, if any

Use only evidence from tool results. Do not invent missing properties, missing alert rows, thresholds, or causes.

```parler-task-checklist-v1
{
  "schemaVersion": 1,
  "requiredEvidence": [
    {
      "id": "taxonomy_asset_resolution",
      "description": "Call resolve_asset_type (or list_asset_types when listing/disambiguating); obtain entityType, entityName, and criticalProperties[] for the requested asset type."
    },
    {
      "id": "hierarchy_node_scopes",
      "tool": "query_entities_by_taxonomy",
      "description": "List matching assets under each requested hierarchy node."
    },
    {
      "id": "asset_current_values",
      "tool": "get_property_values",
      "description": "Read current critical and relevant operational properties for scoped assets."
    },
    {
      "id": "asset_alert_summaries",
      "tool": "query_alert_summary",
      "description": "Read current alert summary rollups for scoped assets in one or few query_alert_summary thingNames[] calls."
    },
    {
      "id": "regional_health_comparison",
      "description": "Compare hierarchy nodes by alert category, alert rows per asset, and current readings."
    }
  ]
}
```

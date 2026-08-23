# All model-facing tools and context cost

This note is the working reference for context-budget and tool-admission decisions. It combines a **current built-in inventory** (measured **2026-07-08** @ extension **0.1.209**) with **historical incident evidence** from **2026-06-28** (different tool populations — see counting rules below).

**History overlay (2026-07):** model-facing **`build_period_over_period_chart`** and **`build_multi_series_history_chart`** are retired. Use **`build_history_overlay_chart`** for same-window cross-Thing comparison (`absolute_time`) and shifted-window overlays (`elapsed_time`). Design: [`history-overlay-chart.md`](./history-overlay-chart.md).

## Counting populations (read before comparing numbers)

Different figures in this file measure **different surfaces**. Do not compare them without the population label.

| Population | Count (baseline) | What it includes | How measured |
| --- | ---: | --- | --- |
| **(a) Built-in registry `registerAll`** | **24** | All first-party tools registered by **`BuiltInTools.registerAll`**, including **`get_agent_skill`** — **excludes** executor-only compatibility tools (`get_entity`, legacy discovery trio). The rank table and per-tool payload sections below measure this inventory. | Per-tool wire objects via **`ToolSchemaSizer`** + **`BuiltInToolMergedDefinitionFootprintTest`** (`anthropic-messages-v1`) |
| **(b) Conditional / meta merge additions** | varies | **`start_playbook`** (when playbook catalog loaded), document-knowledge tools when host context enables them, **`load_tool_schemas`** in `lazy` admission mode — added at runtime merge, **not** counted in the (a) rank table. **`get_agent_skill`** is **in (a)** but may be **omitted** from the merged model-facing list when no skills are loaded (`ModelFacingSkillAdmission.hasModelFacingSkills(...)` in **`AgentThing.getMergedToolDefinitions()`**). | Runtime merge + admission policy |
| **(c) Repository extended tools** | **manifest-dependent** | Entries from deployment **`/tools/extended_tools.json`** (sample: **4** post-LLM-friendly utilization tools in **`dev_data/scpa_utilization/tools/extended_tools.json`**) | Reconstructed from manifest + target service schemas at deploy time |
| **(d) Historical incident full surface** | **34** @ **`toolSchemaChars=68049`** | **2026-06-28** live run: (a)-like built-ins **plus** seven **pre-LLM-friendly** utilization extended tools **plus** **`start_playbook`** when playbooks loaded — **not** comparable to (a) alone | Live **`LLM_CONTEXT_PLAN_FAIL`** log (below) |

Training note — **utilization extended tools:** early workshop stages taught a **seven-tool** `utilization_*` surface (**pre-LLM-friendly**); the **post-LLM-friendly** / Day 4 training manifest consolidates to **four** tools (`list_utilization_machines`, `get_utilization_records`, `get_utilization_state_summary`, `get_utilization_overview`). See **`training-stage-configuration-contracts.md`** and §Repository Extended Tool Payloads below. The **2026-06-28** incident row (d) used the **seven-tool** generation.

## Historical incident (2026-06-28) — population **(d)**

The immediate incident was a live request on `SCPA_Agent_Sonnet` where the model call failed before history could be admitted:

```text
LLM_CONTEXT_PLAN_FAIL reason=OVERHEAD_EXCEEDS_CAP
messages=28 tools=34 stableChars=31638 toolSchemaChars=68049 ephemeralChars=5242
currentUserChars=59 activeBatchReserveChars=13857 transcriptChars=0
evidenceRawChars=8372 evidenceChars=8372
configuredCapChars=750000 effectiveRequestCapChars=118622
historyBudgetChars=-223 historyClampedToZero=1
```

The important signal is not the raw configured cap. The provider/runtime effective request cap was `118622` chars, and non-history overhead alone exceeded that limit:

- stable prompt/context: `31638`
- all tool schemas: `68049`
- ephemeral context: `5242`
- active batch reserve: `13857`
- current user prompt: `59`
- evidence: `8372`

That leaves a negative history budget before conversation history is considered.

## Measurement Semantics

`toolSchemaChars` is not token count. It is the Java-side compact JSON character length used by the context planner for Anthropic-compatible tool wire objects:

```json
{
  "name": "...",
  "description": "...",
  "input_schema": {}
}
```

The full logged `toolSchemaChars=68049` includes:

- every model-facing tool object,
- JSON array punctuation,
- the provider cache marker on the last tool,
- small runtime serialization details.

The per-tool sizes in population **(d)** below are measured as individual compact tool objects. The sum of individual rows was `67938` chars; the logged full request total was `68049` chars.

## Current built-in default surface — population **(a)**

Measured **2026-07-08** with **`ToolSchemaSizer`** (`anthropic-messages-v1`) over population **(a)** — all **`BuiltInTools.registerAll`** definitions (**24** tools, including **`get_agent_skill`**). Array framing + Anthropic cache marker adds **`58268 − 58206 = 62`** chars vs the per-tool sum.

| Rank | Tool | Size chars | Source |
|---:|---|---:|---|
| 1 | `query_entities` | 6,549 | exact |
| 2 | `tabulate_cached_result` | 6,241 | exact |
| 3 | `query_entities_by_taxonomy` | 4,594 | exact |
| 4 | `build_chart_from_tabular_result` | 4,272 | exact |
| 5 | `query_property_history` | 4,152 | exact |
| 6 | `invoke_service` | 3,667 | exact |
| 7 | `build_history_overlay_chart` | 3,397 | exact |
| 8 | `query_alert_history` | 2,901 | exact |
| 9 | `describe_entity_schema` | 2,868 | exact |
| 10 | `discover_thing_members` | 2,515 | exact |
| 11 | `query_alert_summary` | 2,357 | exact |
| 12 | `analyze_entity_set` | 2,251 | exact |
| 13 | `list_entities_by_type` | 2,137 | exact |
| 14 | `acknowledge_alerts` | 1,723 | exact |
| 15 | `query_stream_data` | 1,522 | exact |
| 16 | `get_property_values` | 1,126 | exact |
| 17 | `fetch_cached_result` | 1,034 | exact |
| 18 | `summarize_cached_result` | 956 | exact |
| 19 | `set_property_value` | 921 | exact |
| 20 | `get_agent_skill` | 866 | exact |
| 21 | `resolve_thing` | 678 | exact |
| 22 | `spotlight_search` | 610 | exact |
| 23 | `resolve_asset_type` | 436 | exact |
| 24 | `list_asset_types` | 433 | exact |

Summary (population **(a)** only):

- tool count: **24**
- per-tool object sum: **58,206** chars
- serialized tools array total (with framing): **58,268** chars
- top 10 tools total: **39,016** chars (~67% of per-tool sum)
- top 5 tools total: **24,804** chars (~43% of per-tool sum)

Executor-only tools are **not** in population **(a)** but remain callable via **`executeTool`** / replay:

- `discover_properties`
- `discover_services`
- `get_entity`
- `get_service_definition`

## Historical runtime surface — population **(d)** @ 2026-06-28

This live run exposed **34** model-facing tools:

- 26 always/conditionally available built-ins from `AgentThing` runtime configuration (pre-overlay merge; **no** `build_history_overlay_chart`),
- `start_playbook`, because playbooks were loaded,
- **7** repository extended tools from `/tools/extended_tools.json` (**pre-LLM-friendly** utilization generation).

### Per-tool context cost (2026-06-28 incident census)

`exact` means the size was computed directly from Java `ToolDefinition` serialization. `reconstructed` means the tool came from repository extended-tool configuration and was reconstructed from the live target service definitions, including service parameter schemas and INFOTABLE DataShape expansion.

| Rank | Tool | Size chars | Source |
|---:|---|---:|---|
| 1 | `query_entities` | 6,549 | exact |
| 2 | `tabulate_cached_result` | 6,241 | exact |
| 3 | `query_entities_by_taxonomy` | 4,594 | exact |
| 4 | `build_chart_from_tabular_result` | 4,232 | exact |
| 5 | `query_property_history` | 4,152 | exact |
| 6 | `invoke_service` | 3,667 | exact |
| 7 | `query_alert_history` | 2,901 | exact |
| 8 | `describe_entity_schema` | 2,868 | exact |
| 9 | `discover_thing_members` | 2,515 | exact |
| 10 | `analyze_entity_set` | 2,251 | exact |
| 11 | `list_entities_by_type` | 2,137 | exact |
| 12 | `utilization_machine_listing_with_dates` | 2,087 | reconstructed |
| 13 | `query_alert_summary` | 2,156 | exact |
| 14 | `acknowledge_alerts` | 1,723 | exact |
| 15 | `utilization_records_by_machine` | 1,537 | reconstructed |
| 16 | `query_stream_data` | 1,522 | exact |
| 17 | `search_document_chunks` | 1,516 | exact |
| 18 | `utilization_aggregate_by_state_time_fence` | 1,438 | reconstructed |
| 19 | `utilization_records` | 1,386 | reconstructed |
| 20 | `utilization_aggregate_by_state` | 1,170 | reconstructed |
| 21 | `get_property_values` | 1,126 | exact |
| 22 | `start_playbook` | registry-driven | varies with loaded playbooks |
| 23 | `utilization_machine_listing` | 1,082 | reconstructed |
| 24 | `utilization_stats_for_aggregate` | 1,037 | reconstructed |
| 25 | `fetch_cached_result` | 1,034 | exact |
| 26 | `summarize_cached_result` | 956 | exact |
| 27 | `set_property_value` | 921 | exact |
| 28 | `get_agent_skill` | 866 | exact |
| 29 | `resolve_document_set` | 759 | exact |
| 30 | `resolve_thing` | 678 | exact |
| 31 | `spotlight_search` | 610 | exact |
| 32 | `get_document_chunk` | 527 | exact |
| 33 | `resolve_asset_type` | 436 | exact |
| 34 | `list_asset_types` | 433 | exact |

Summary (population **(d)**):

- tool count: `34`
- individual tool object total: `67,938` chars
- logged full `toolSchemaChars`: `68,049` chars
- top 10 tools total: `39,970` chars, about 59% of all tool schema cost
- top 5 tools total: `25,768` chars, about 38% of all tool schema cost

## What This Means

The current context problem is not caused by conversation history alone. A large request can fail even with `transcriptChars=0` when these are all true:

- many model-facing tools are advertised at once,
- stable prompt/context is already large,
- evidence or host context is present,
- active batch reserve is held,
- the provider's effective request cap is much lower than the nominal configured cap.

This is why "just compact history" cannot solve every context failure. Tool schema is fixed overhead for a round; compaction can only reclaim transcript and retained evidence.

## High-Cost Tool Classes

The largest fixed costs come from four classes.

### Broad query and table tools

These tools are expensive because their schemas describe flexible predicates, table/cache contracts, or chart-ready data contracts:

- `query_entities`
- `tabulate_cached_result`
- `query_entities_by_taxonomy`
- `build_chart_from_tabular_result`
- `build_history_overlay_chart`
- `query_property_history`

They are valuable, but they should not all be assumed necessary for every turn.

### Generic metadata and invocation tools

These tools carry broader safety and input-shape guidance:

- `invoke_service`
- `describe_entity_schema`
- `discover_thing_members`

They are useful for exploratory turns, but they are rarely all needed once the route is clear.

### Repository extended tools

Utilization manifests differ by **training stage** (see counting table above):

| Stage | Tool count | Sample manifest | Notes |
| --- | ---: | --- | --- |
| **Pre-LLM-friendly** (early workshop) | **7** | legacy `utilization_*` names in §Historical utilization payloads | Population **(d)** / 2026-06-28 incident; ~**9.7K** chars combined |
| **Post-LLM-friendly** (Day 4 / current sample) | **4** | `dev_data/scpa_utilization/tools/extended_tools.json` | Population **(c)** for the shipped training bundle |

Extended-tool sizes come from manifest `whenToUse` text, target service parameter definitions, INFOTABLE DataShape expansion, and synthetic natural-time fields. They are acceptable on utilization turns but wasteful when unrelated tools are co-advertised (population **(d)** lesson).

### Document tools

The document tools are individually small, but they should still be conditional:

- `resolve_document_set`
- `search_document_chunks`
- `get_document_chunk`

They are not useful for normal ThingWorx operational questions unless document knowledge is in play.

## Direction for a Real Fix

A durable fix should avoid returning to "remove more tools" as the first answer. The better target is admission control: expose the right tool set for the current turn.

### 1. Add per-tool schema telemetry

The runtime should log per-tool schema sizes directly, not require offline reconstruction. A future `LLM_TOOL_SCHEMA_USAGE` line should include either:

- `toolSchemaSizes=name:size,...`, or
- a compact JSON map in diagnostics-only telemetry.

This makes live-debug immediate:

```text
LLM_TOOL_SCHEMA_USAGE tools=34 toolSchemaChars=68049
toolSchemaSizes=query_entities:6549,tabulate_cached_result:6241,...
```

The telemetry should be diagnostic-only and should not change wire contracts.

### 2. Introduce turn-level tool admission

The model-facing tool list should be selected by turn intent before the LLM request is built.

Suggested first-level buckets:

| Bucket | Typical tools |
|---|---|
| Identity / routing | `resolve_thing`, `resolve_asset_type`, `list_asset_types`, `list_entities_by_type` |
| Entity set query | `query_entities`, `query_entities_by_taxonomy`, `analyze_entity_set` |
| Current values / trends | `get_property_values`, `query_property_history`, `query_stream_data`, chart/table tools |
| Alerts | `query_alert_summary`, `query_alert_history`, `acknowledge_alerts` |
| Metadata exploration | `describe_entity_schema`, `discover_thing_members`, `invoke_service` |
| Documents | `resolve_document_set`, `search_document_chunks`, `get_document_chunk` |
| Skills / playbooks | `get_agent_skill`, `start_playbook` |
| Utilization extension (manifest-dependent) | **4** current (`get_utilization_*`, `list_utilization_machines`) or **7** historical (`utilization_*`) — see §Repository Extended Tool Payloads |

The default should still be conservative: do not hide a tool unless a reliable route decision says it is not relevant for this turn.

### 3. Make extended tools conditional

Repository extended tools should have enough metadata to support admission. Current `whenToUse` text helps the LLM, but it does not reduce the schema payload.

Possible additions:

```json
{
  "name": "utilization_records",
  "admission": {
    "keywords": ["utilization", "uptime", "downtime", "state duration"],
    "hostContextKeys": ["utilization"],
    "defaultAdvertise": false
  }
}
```

The first implementation can be simple and auditable. It does not need to be a second LLM router.

### 4. Keep skill and playbook advertisement conditional

`start_playbook` and `get_agent_skill` are only useful when the agent has loaded playbooks or skills. **`get_agent_skill`** is always registered in population **(a)** but may be filtered from the merged LLM list when no model-facing skills are configured. The same conditional-admission idea should be extended:

- if no skills are loaded, do not advertise skill retrieval;
- if no playbooks are loaded, do not advertise `start_playbook`;
- if the user directly invokes `/playbook_id`, bypass broad discovery and use the requested playbook path;
- if only one route-specific playbook is relevant, prefer advertising that path through prompt text rather than keeping all unrelated tool families visible.

### 5. Separate "executor capability" from "model-facing capability"

Several tools are already executor-only. That distinction should be kept and expanded when appropriate:

- executor can still call a capability from playbook/runtime code,
- model does not need to see that capability on every turn,
- playbook validation can still know whether the tool exists.

This is the right pattern for legacy discovery tools and can also apply to future helper-only tools.

### 6. Apply context cap before final tool admission

Tool admission should be budget-aware:

1. Build candidate tool set from runtime configuration.
2. Apply route/intent narrowing.
3. Estimate tool schema chars.
4. If non-history overhead still exceeds cap, apply deterministic degradation:
   - drop unrelated extended tools,
   - drop document tools outside document turns,
   - drop broad metadata tools outside metadata turns,
   - keep minimum recovery tools such as `resolve_thing` and the currently intended executor path.

The degradation path must be logged. Silent tool removal will make live-debug confusing.

## Immediate Reading of This Incident

For the captured failure, the main fixed-cost contributors were:

- broad query/table/chart tools: about 25.8K chars for the top 5 alone,
- utilization extended tools: about 9.7K chars,
- metadata/invoke tools: about 9.0K chars for `invoke_service`, `describe_entity_schema`, and `discover_thing_members`,
- `activeBatchReserveChars=13857`, which made an already tight request impossible.

The key issue is that utilization tools, document tools, broad entity tools, metadata tools, playbook tools, and chart/table tools were all advertised together. That is operationally convenient but not sustainable under smaller effective provider caps.

## Recommended Next Topic

Use this document as the basis for a focused implementation topic:

```text
tool-schema-admission-control
```

Minimum scope:

- add per-tool schema-size telemetry,
- implement deterministic tool buckets,
- conditionally advertise repository extended tools,
- preserve executor-only behavior,
- add a budget-aware final admission pass,
- include live diagnostics proving reduced `toolSchemaChars` for unrelated turns.

Success criteria:

- normal asset-health turns do not advertise utilization tools unless the prompt or host context asks for utilization;
- utilization turns keep the utilization tools;
- document tools only appear for document-relevant turns;
- broad metadata tools are preserved for exploration but not for straightforward operational questions;
- `LLM_CONTEXT_PLAN_FAIL reason=OVERHEAD_EXCEEDS_CAP` becomes rare and explainable from telemetry.


## Tool Context Payloads

This appendix lists the model-facing JSON content for every tool in the captured runtime surface. It is intentionally verbose: the goal is to inspect exactly which descriptions, enums, and parameter schemas consume context.

The built-in payloads are generated from Java `ToolDefinition` objects. The extended-tool payloads are reconstructed from the live `/tools/extended_tools.json` plus the live target service definitions and DataShape expansion.

### Reading Guide

For each tool:

- `description` is always sent to the model with the tool.
- `input_schema` is always sent to the model with the tool.
- long `enum` values, repeated natural-time guidance, large table contracts, and INFOTABLE DataShape expansion are the most obvious optimization candidates.

## Built-In And Playbook Tool Payloads

### invoke_service

Size chars: `3667`

```json
{
  "name" : "invoke_service",
  "description" : "Generic service invocation on any ThingWorx entity. Prefer this when: (1) you discovered the service via discover_services / get_service_definition on that entity, or (2) a previous tool result or the user explicitly points to a specific service to run next. Do NOT use invoke_service to list metadata entities by collection type — use list_entities_by_type (EntityServices GetEntityList*). Do not list all Things this way — use query_entities or spotlight_search. Remote Service (edge-only) calls are blocked. All service arguments belong under **parameters**; never at the top level beside entityType / entityName / serviceName. Large INFOTABLE results return sampleRows and **may** include **cacheId** when the server stored a paged copy; **fetch_cached_result** is for displaying or browsing a page over **cacheId** only — not for scanning the full table into the model. For full-table computation use **tabulate_cached_result** or **summarize_cached_result**. When **cacheId** is absent, answer from **sampleRows** / **hint** only. Do not invoke platform full-metadata services such as GetMetadata, GetMetadataAsJSON, GetEntityDescription, GetPropertyDefinitions, GetServiceDefinitions, or similar full-metadata dump services — those return large platform metadata bodies that exceed the per-request size cap. Use discover_thing_members (concrete Thing members) or describe_entity_schema (ThingTemplate / ThingShape / DataShape facets), then get_property_values when reading live values. Legacy discover_properties / discover_services / get_service_definition remain valid for replay or continuation of an existing tool sequence. A combined full-metadata dump for ThingTemplate / ThingShape / DataShape is for **Tier B / persisted replay** (**get_entity**, off the merged LLM tool list) when **describe_entity_schema** facets are not enough — never the default greenfield inspection path. The agent rejects oversized invoke_service results with INVOKE_SERVICE_RESULT_TOO_LARGE.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "entityType" : {
        "type" : "string",
        "enum" : [ "ApplicationKey", "Authenticator", "Dashboard", "DataShape", "DataTagVocabulary", "DirectoryService", "ExtensionPackage", "Group", "LocalizationTable", "Log", "Mashup", "MediaEntity", "Menu", "ModelTagVocabulary", "Network", "NotificationContent", "NotificationDefinition", "Organization", "PersistenceProvider", "PersistenceProviderPackage", "Project", "Resource", "ScriptFunctionLibrary", "StateDefinition", "StyleDefinition", "StyleTheme", "Subsystem", "Thing", "ThingGroup", "ThingPackage", "ThingShape", "ThingTemplate", "User", "Widget" ],
        "description" : "Root ThingWorx entity type (from platform ThingworxEntityTypes). Concrete DataTable / Stream / ValueStream **Things** use entityType \"Thing\" and the instance name; the server may also normalize a GenericThing-derived ThingTemplate name (e.g. DataTable) to Thing when it appears in the dependency catalog."
      },
      "entityName" : {
        "type" : "string",
        "description" : "Name of the entity"
      },
      "serviceName" : {
        "type" : "string",
        "description" : "Name of the service to invoke"
      },
      "parameters" : {
        "type" : "object",
        "description" : "All target service inputs MUST be nested inside **parameters**. Do not put service inputs such as maxItems, values, query, tags, startDate, or endDate at the invoke_service top level — use {\"parameters\":{\"maxItems\":3}}, not {\"maxItems\":3}. Optional when the service has no inputs. INFOTABLE: array of row objects (or one object). TAGS: JSON array of {\"vocabulary\":\"...\",\"vocabularyTerm\":\"...\"} (see docs/agent/tags_in_agent_tools.md)."
      }
    },
    "required" : [ "entityType", "entityName", "serviceName" ],
    "additionalProperties" : false
  }
}
```

### fetch_cached_result

Size chars: `1034`

```json
{
  "name" : "fetch_cached_result",
  "description" : "After a large tabular tool (**invoke_service** INFOTABLE_LARGE, query_entities, query_entities_by_taxonomy, or list_entities_by_type) returned a **cacheId**, fetch a **page** from the in-memory cache for **this conversation** for UI display or browsing. Read-only paging — does not sort or aggregate. Do **not** call repeatedly to read the entire table into the model — use **tabulate_cached_result** or **summarize_cached_result** for full-table computation. If **cacheId** was omitted, **do not** call this tool — answer from **sampleRows** / **hint**.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "cacheId" : {
        "type" : "string",
        "description" : "cacheId from a prior large tabular tool (e.g. invoke_service INFOTABLE_LARGE **when** the response included cacheId — omit paging when cacheId was absent)."
      },
      "offset" : {
        "type" : "integer",
        "description" : "Zero-based row offset (default 0)"
      },
      "limit" : {
        "type" : "integer",
        "description" : "Max rows to return (default 50, max 200)"
      }
    },
    "required" : [ "cacheId" ]
  }
}
```

### tabulate_cached_result

Size chars: `6241`

```json
{
  "name" : "tabulate_cached_result",
  "description" : "Deterministic transforms over a **conversation-cached** tabular result (same cache as fetch_cached_result). Always reads the **full** table for cacheId — fetch_cached_result is only for paging/browsing. Success includes sourceCacheId. LARGE results return a **new** cacheId for the transformed table. Modes include sort/group transforms plus **cached-table decision** filters: **filter_count**, **filter_rows**, **filter_sort_topn**, **group_metric** (see docs/agent/cached-table-decision-tools.md). Use for Top-N, sort, group counts, group aggregates, threshold counts, filtered tables, and ranked subsets instead of guessing from prior tool text.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "cacheId" : {
        "type" : "string",
        "description" : "Source cacheId — entire cached InfoTable is transformed. May be __PARLER_LAST_QUALIFYING_TABULAR_CACHE__ (see docs/agent/p2_last_tabular_cache.md)."
      },
      "mode" : {
        "enum" : [ "sort_topn", "group_count", "group_aggregate", "filter_count", "filter_rows", "filter_sort_topn", "group_metric" ],
        "type" : "string",
        "description" : "sort_topn: required sorts (1–3 keys), optional maxItems/offset/fields. group_count / group_aggregate: groupBy + optional maxItems on group output. filter_count: required filters; optional groupBy (string). filter_rows: required filters; optional sorts, maxItems, offset, fields. filter_sort_topn: required sorts; optional filters, maxItems, offset, fields. group_metric: required measures; optional filters, groupBy, derived, having, sorts, maxItems, offset, fields. Normative: docs/agent/query-spec.md."
      },
      "filters" : {
        "type" : "object",
        "description" : "Row predicate (filter_* modes; optional for filter_sort_topn). ThingWorx-shaped filter object (docs/agent/query-spec.md §3): leaves use uppercase \"type\" (EQ, NE, LT, LE, GT, GE, LIKE, IN, BETWEEN, …) and \"fieldName\" plus type-specific keys (e.g. \"value\", \"from\"/\"to\", \"values\"). Composites: {\"type\":\"AND\",\"filters\":[...]} or OR/NOT. Optional \"isCaseSensitive\" on string leaves/sorts."
      },
      "sorts" : {
        "type" : "array",
        "description" : "1–3 sort keys for sort_topn / filter_rows / filter_sort_topn / group_metric output.",
        "items" : {
          "type" : "object",
          "properties" : {
            "fieldName" : {
              "type" : "string"
            },
            "isAscending" : {
              "type" : "boolean",
              "description" : "Optional; default true (ascending) per query-spec §4.1."
            },
            "isCaseSensitive" : {
              "type" : "boolean",
              "description" : "Optional; default true for string sort keys (query-spec §4)."
            }
          },
          "required" : [ "fieldName" ]
        }
      },
      "maxItems" : {
        "type" : "integer",
        "description" : "Output row cap (1–500) after sort/filter; default 50 for sort-style modes."
      },
      "offset" : {
        "type" : "integer",
        "description" : "Non-negative slice offset after sort (default 0)."
      },
      "fields" : {
        "type" : "array",
        "description" : "Optional output column projection (query-spec §5.3): array of source or output column names, order preserved. Allowed on sort_topn, filter_rows, filter_sort_topn, group_metric only.",
        "items" : {
          "type" : "string"
        }
      },
      "measures" : {
        "type" : "array",
        "description" : "group_metric: up to 10 measures.",
        "items" : {
          "type" : "object",
          "properties" : {
            "name" : {
              "type" : "string"
            },
            "op" : {
              "enum" : [ "count", "count_non_null", "sum", "avg", "min", "max", "count_distinct", "weighted_avg", "median", "percentile", "variance", "stddev", "first", "last", "mode" ],
              "type" : "string"
            },
            "column" : {
              "type" : "string"
            },
            "filters" : {
              "type" : "object",
              "description" : "Optional per-measure row filter over source rows. For an unconditional measure, omit this field entirely — canonical \"no filter\" is absence of filters. Do not send placeholder predicates such as {\"type\":\"TRUE\"}, {\"type\":\"ALL\"}, or {\"type\":\"MATCH_ALL\"}. ThingWorx-shaped filter object (docs/agent/query-spec.md §3): leaves use uppercase \"type\" (EQ, NE, LT, LE, GT, GE, LIKE, IN, BETWEEN, …) and \"fieldName\" plus type-specific keys (e.g. \"value\", \"from\"/\"to\", \"values\"). Composites: {\"type\":\"AND\",\"filters\":[...]} or OR/NOT. Optional \"isCaseSensitive\" on string leaves/sorts."
            },
            "weightColumn" : {
              "type" : "string"
            },
            "p" : {
              "type" : "number",
              "description" : "percentile only: p in [0,100]."
            },
            "orderBy" : {
              "type" : "string",
              "description" : "first / last only: sortable column name."
            },
            "direction" : {
              "enum" : [ "asc", "desc", "ascending", "descending" ],
              "type" : "string",
              "description" : "first / last only; default asc when omitted or blank."
            }
          },
          "required" : [ "name", "op" ]
        }
      },
      "derived" : {
        "type" : "array",
        "description" : "group_metric: derived metrics (max 10).",
        "items" : {
          "type" : "object",
          "properties" : {
            "name" : {
              "type" : "string"
            },
            "op" : {
              "enum" : [ "ratio", "ratio_percent", "difference", "sum_values", "multiply", "scale" ],
              "type" : "string"
            },
            "numerator" : {
              "type" : "string"
            },
            "denominator" : {
              "type" : "string"
            },
            "left" : {
              "type" : "string"
            },
            "right" : {
              "type" : "string"
            },
            "inputs" : {
              "type" : "array",
              "items" : {
                "type" : "string"
              },
              "description" : "sum_values / multiply: non-blank measure or derived names (max 10)."
            },
            "input" : {
              "type" : "string",
              "description" : "scale: operand measure/derived name."
            },
            "factor" : {
              "type" : "number",
              "description" : "scale: numeric factor."
            }
          },
          "required" : [ "name", "op" ]
        }
      },
      "having" : {
        "type" : "object",
        "description" : "group_metric only: ThingWorx-shaped predicate applied **after** measures/derived — **filters the grouped output rows** to the true answer set. **Prefer `having`** for questions that ask which groups match a metric **equality or threshold** (e.g. utilization = 0, below 30%, no running time); do **not** rely on sorting plus inspecting **`sampleRows`** — samples are previews only and **`totalRows`** is the output size of the query you executed, not proof that every row matches values seen in the sample. ThingWorx-shaped filter object (docs/agent/query-spec.md §3): leaves use uppercase \"type\" (EQ, NE, LT, LE, GT, GE, LIKE, IN, BETWEEN, …) and \"fieldName\" plus type-specific keys (e.g. \"value\", \"from\"/\"to\", \"values\"). Composites: {\"type\":\"AND\",\"filters\":[...]} or OR/NOT. Optional \"isCaseSensitive\" on string leaves/sorts."
      },
      "groupBy" : {
        "oneOf" : [ {
          "type" : "string"
        }, {
          "type" : "array",
          "items" : {
            "type" : "string"
          }
        } ],
        "description" : "group_count / group_aggregate: non-empty string column. filter_count: optional string. group_metric: array of 0–5 source column names (or [])."
      },
      "aggregateColumn" : {
        "type" : "string"
      },
      "fn" : {
        "enum" : [ "count", "sum", "avg", "min", "max" ],
        "type" : "string"
      }
    },
    "required" : [ "cacheId", "mode" ]
  }
}
```

### analyze_entity_set

Size chars: `2251`

```json
{
  "name" : "analyze_entity_set",
  "description" : "Deterministic set algebra over two **already cached** entity/list tables: **difference**, **intersection**, **union**, or **symmetric_difference**. Does **not** query ThingWorx. Operands must supply explicit **cacheId** values (never the last-tabular sentinel). Every success returns a **new cacheId** for the full transformed table (including small inline rows) so the next step can call **tabulate_cached_result** / **build_chart_from_tabular_result** without paging the set. Do **not** pass **groupBy** here — use **tabulate_cached_result(mode=group_count)** on the returned cacheId. Typical flows: template-minus-taxonomy → **difference**; overlap → **intersection**; combine two lists → **union**; items in exactly one list → **symmetric_difference**; then **tabulate_cached_result** → chart.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "operation" : {
        "type" : "string",
        "description" : "Set operation: **difference**, **intersection**, **union**, or **symmetric_difference**."
      },
      "left" : {
        "type" : "object",
        "properties" : {
          "cacheId" : {
            "type" : "string",
            "description" : "Explicit conversation cache id from a prior list/tabular tool."
          },
          "keyColumn" : {
            "type" : "string",
            "description" : "Identity column on this operand (default **name**)."
          },
          "label" : {
            "type" : "string",
            "description" : "Optional diagnostic label."
          }
        },
        "required" : [ "cacheId" ]
      },
      "right" : {
        "type" : "object",
        "properties" : {
          "cacheId" : {
            "type" : "string",
            "description" : "Explicit conversation cache id from a prior list/tabular tool."
          },
          "keyColumn" : {
            "type" : "string",
            "description" : "Identity column on this operand (default **name**)."
          },
          "label" : {
            "type" : "string",
            "description" : "Optional diagnostic label."
          }
        },
        "required" : [ "cacheId" ]
      },
      "projectColumns" : {
        "type" : "array",
        "description" : "Optional projection. For **difference** / **intersection**, names columns on the **left** operand only. For **union** / **symmetric_difference**, each name must exist on **at least one** operand; missing cells are **null** for keys present on only one side.",
        "items" : {
          "type" : "string"
        }
      },
      "maxItems" : {
        "type" : "integer",
        "description" : "Page size into output rows (default 50, max 500)."
      },
      "offset" : {
        "type" : "integer",
        "description" : "Zero-based offset into output rows."
      }
    },
    "required" : [ "operation", "left", "right" ]
  }
}
```

### summarize_cached_result

Size chars: `956`

```json
{
  "name" : "summarize_cached_result",
  "description" : "Column-level stats (null counts, numeric min/max/mean and capped p50/p95, categorical cardinality/top) over the **full** cached InfoTable. Success includes sourceCacheId. Use for summaries and null-heavy questions instead of eyeballing row JSON.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "cacheId" : {
        "type" : "string",
        "description" : "Cached table to summarize (full table). May be __PARLER_LAST_QUALIFYING_TABULAR_CACHE__ (same resolution order as tabulate_cached_result: per-turn then conversation mirror)."
      },
      "percentileColumns" : {
        "type" : "array",
        "description" : "Optional JSON array of non-empty strings (column names) for p50/p95 on numeric columns only. Must be an array of strings (no nulls, numbers, or objects); empty strings invalid. If omitted, p50/p95 apply only to the first 8 numeric columns in declaration order; others still get min/max/mean.",
        "items" : {
          "type" : "string"
        }
      }
    },
    "required" : [ "cacheId" ]
  }
}
```

### build_chart_from_tabular_result

Size chars: `4232`

```json
{
  "name" : "build_chart_from_tabular_result",
  "description" : "Build a Parler chart from a **real** tabular tool result (invoke_service INFOTABLE*, query_entities, query_entities_by_taxonomy, list_entities_by_type, fetch_cached_result, tabulate_cached_result). When the user charted from **analyze_entity_set**, call **tabulate_cached_result** (or **fetch_cached_result**) on that tool's **cacheId** first — charts consume tabular tool shapes, not raw entity-set envelopes. Supply **either** explicit `kind` (line|bar|scatter|pie) **or** `intent` (server picks among those kinds) — never both; runtime rejects requests that set both or neither. Root JSON Schema is a single object (provider-safe); mutual exclusion is not expressed as a root-level `oneOf`. For **one** chart from the **most recent** qualifying tabular result, `source: \"last_invoke\"` is fine. When emitting **multiple** charts in **one** turn that each draw from a **different** prior tabular result, use `source: \"cache_id\"` and the top-level `cacheId` string copied from that result's envelope (extended INFOTABLE tool results surface `cacheId`; do not invent ids). Calling multiple builds all with `last_invoke` reuses the same table. See docs/agent/chart-intent.md, docs/architecture/flexible-chart-solution.md, and CONTRACTS/CHART_CONTRACT.md §2.5.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "source" : {
        "type" : "string",
        "description" : "last_invoke — chart from the single qualifying tabular tool this turn (most recent only). cache_id — read the full table from the conversation cache using the top-level cacheId argument (same cache as fetch_cached_result). For multiple charts from different prior results in one turn, use cache_id + each result's cacheId; do not use last_invoke for every chart or they will all share the same source table."
      },
      "cacheId" : {
        "type" : "string",
        "description" : "Required when source is cache_id: opaque id from the prior tabular tool JSON envelope (top-level cacheId on INFOTABLE/INFOTABLE_LARGE success bodies). Not derivable from tool call ids."
      },
      "xColumn" : {
        "type" : "string",
        "description" : "Column name for X (category for bar; numeric or ISO time for line/scatter)."
      },
      "yColumn" : {
        "type" : "string",
        "description" : "Column name for Y (single series). Omit when using series[]."
      },
      "series" : {
        "type" : "array",
        "description" : "Multi-series: [{ name, yColumn }], shared xColumn. Mutually exclusive with yColumn.",
        "items" : {
          "properties" : {
            "name" : {
              "type" : "string"
            },
            "yColumn" : {
              "type" : "string"
            }
          },
          "type" : "object",
          "required" : [ "name", "yColumn" ]
        }
      },
      "title" : {
        "type" : "string"
      },
      "xLabel" : {
        "type" : "string"
      },
      "yLabel" : {
        "type" : "string"
      },
      "yReferenceLines" : {
        "type" : "array",
        "maxItems" : 12,
        "description" : "Horizontal lines (max 12): { y, label?, role? }.",
        "items" : {
          "type" : "object",
          "properties" : {
            "y" : {
              "type" : "number"
            },
            "label" : {
              "type" : "string"
            },
            "role" : {
              "enum" : [ "usl", "ucl", "lcl", "lsl", "target", "limit", "warning" ],
              "type" : "string",
              "description" : "Optional line role (SPC / threshold)."
            }
          },
          "required" : [ "y" ]
        }
      },
      "requestedTimeRange" : {
        "type" : "object",
        "description" : "Optional ISO-8601 bounds for time line/scatter X domain (tabular charts)."
      },
      "seriesColumn" : {
        "type" : "string",
        "description" : "Optional for kind bar: long-format column whose distinct values become multiple series (grouped bar). Mutually exclusive with series[]. Requires yColumn."
      },
      "pieSliceMode" : {
        "enum" : [ "top_with_other", "all_nonzero" ],
        "type" : "string",
        "description" : "Optional for kind pie: slice policy (default top_with_other). Ignored for other kinds."
      },
      "pieMaxSlices" : {
        "type" : "integer",
        "description" : "Optional for kind pie: max slices 2..12 (default 8). Ignored for other kinds.",
        "minimum" : 2,
        "maximum" : 12
      },
      "kind" : {
        "enum" : [ "line", "bar", "scatter", "pie" ],
        "type" : "string",
        "description" : "Explicit chart kind (line|bar|scatter|pie). Optional when using intent instead. Mutually exclusive with intent — supply exactly one of kind or intent; runtime rejects both or neither."
      },
      "intent" : {
        "enum" : [ "time_trend", "rank", "compare_groups", "correlation", "distribution", "status_timeline", "composition" ],
        "type" : "string",
        "description" : "Visual intent; server selects line|bar|scatter (or CHART_FALLBACK for phase-only intents). Optional when using explicit kind instead. Mutually exclusive with kind."
      }
    },
    "required" : [ "source", "xColumn" ]
  }
}
```

### build_history_overlay_chart

Size chars: `3397`

```json
{
  "name" : "build_history_overlay_chart",
  "description" : "Build one chart overlaying **2–6 numeric property history traces**. Each series resolves its own Thing and time window. Use **absolute_time** when comparing Things in the **same** window; **elapsed_time** when windows differ (this window vs last week); **normalized_time** when comparing **shape/profile** across different-duration windows (maps each window to 0..1). Set normalized_time explicitly for shape comparisons. Prefer this over repeated query_property_history plus manual assembly. When data is already in a long cached table, prefer build_chart_from_tabular_result with seriesColumn.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "propertyName" : {
        "type" : "string",
        "description" : "Same numeric property on every series Thing (NUMBER/INTEGER/LONG)."
      },
      "series" : {
        "type" : "array",
        "minItems" : 2,
        "maxItems" : 6,
        "items" : {
          "type" : "object",
          "properties" : {
            "label" : {
              "type" : "string",
              "description" : "Legend label for this trace."
            },
            "thingName" : {
              "type" : "string",
              "description" : "Canonical ThingWorx Thing name (exact platform **Thing** name). If the user supplied a display label, serial number, suffix, or any uncertain asset identifier, call **resolve_thing** first (v3 identity taxonomy), then retry with **matches[0].name** from a **UNIQUE** result."
            },
            "calendarPhrase" : {
              "type" : "string",
              "description" : "Optional single-day phrase resolved from anchorTime + user_timezone."
            },
            "relativeDuration" : {
              "type" : "string",
              "description" : "Optional duration ending at the series anchor (e.g. 2h). Mutually exclusive with calendarPhrase and explicit bounds."
            },
            "anchorOffset" : {
              "type" : "string",
              "description" : "Optional shift back from shared anchorTime before resolving this series window (e.g. 7d). Use with relativeDuration for shifted windows. Mutually exclusive with explicit startTime/endTime."
            },
            "startTime" : {
              "type" : "string",
              "description" : "Optional ISO-8601 window start (alias: start)."
            },
            "endTime" : {
              "type" : "string",
              "description" : "Optional ISO-8601 window end (alias: end)."
            }
          },
          "required" : [ "label", "thingName" ]
        },
        "description" : "2–6 history traces. Each entry resolves its own time window from shared anchorTime."
      },
      "xAxisMode" : {
        "type" : "string",
        "enum" : [ "absolute_time", "elapsed_time", "normalized_time" ],
        "description" : "X-axis mode. absolute_time = real timestamps (same-window cross-Thing). elapsed_time = seconds from each series window start (shifted windows). normalized_time = map each series window to 0..1 for shape/profile comparison (different durations). When omitted, server picks absolute_time when all windows match, else elapsed_time. Set normalized_time explicitly for shape comparisons — not inferred."
      },
      "chart_kind" : {
        "type" : "string",
        "enum" : [ "line", "scatter" ],
        "description" : "Chart kind (default line)."
      },
      "anchorTime" : {
        "type" : "string",
        "description" : "Optional ISO anchor for resolving all series windows (default: agent turn clock now)."
      },
      "title" : {
        "type" : "string",
        "description" : "Optional chart title."
      },
      "yLabel" : {
        "type" : "string",
        "description" : "Optional Y-axis label."
      },
      "yReferenceLines" : {
        "type" : "array",
        "maxItems" : 12,
        "items" : {
          "type" : "object",
          "properties" : {
            "y" : {
              "type" : "number",
              "description" : "Y-axis value (finite numeric)."
            },
            "label" : {
              "type" : "string",
              "description" : "Optional line label."
            },
            "role" : {
              "type" : "string",
              "description" : "Optional role: usl, ucl, lcl, lsl, target, limit, warning (default limit)."
            }
          },
          "required" : [ "y" ]
        }
      }
    },
    "required" : [ "propertyName", "series" ]
  }
}
```

### discover_thing_members

Size chars: `2515`

```json
{
  "name" : "discover_thing_members",
  "description" : "**Preferred** built-in for **concrete Thing** member metadata (properties, **public** services, **events**, and **configured subscriptions** in v1): visibility-aware lookup, bounded pagination, list and singular facets, same success/error envelope family as describe_entity_schema. **discover_properties** and Thing-targeted **discover_services** / **get_service_definition** remain registered and delegate here for compatibility — default to this tool for new work; they do not expose **events** or **subscriptions** lists on their own. Does not read property values, invoke services, fire events, refresh subscriptions, or return schema-entity definitions — for ThingTemplate / ThingShape / DataShape use describe_entity_schema.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "thingName" : {
        "type" : "string",
        "description" : "Canonical Thing name. If uncertain, use resolve_thing first; on no match, try spotlight_search."
      },
      "facet" : {
        "enum" : [ "properties", "property", "services", "service", "events", "event", "subscriptions" ],
        "type" : "string",
        "description" : "Member facet. v1: properties (list), property (singular, requires memberName), services (public list), service (singular public service, requires memberName), events (list), event (singular, requires memberName), subscriptions (configured multi-event subscription list, read-only)."
      },
      "memberName" : {
        "type" : "string",
        "description" : "Required for singular facets property, service, and event — member name (case-insensitive fallback after exact match for property, service, and event). Must be omitted for list facets (properties, services, events, subscriptions); use namePrefix to narrow a list — passing memberName on a list facet returns UNSUPPORTED_TOOL_PARAMETER."
      },
      "namePrefix" : {
        "type" : "string",
        "description" : "Optional list filter on member name prefix."
      },
      "category" : {
        "type" : "string",
        "description" : "Optional category filter where applicable."
      },
      "baseType" : {
        "type" : "string",
        "description" : "Optional filter: property base type, or service result base type for facet=services (ignored for other facets when present)."
      },
      "dataShape" : {
        "type" : "string",
        "description" : "Optional INFOTABLE dataShape filter for facet=properties (property aspect) and facet=events (EventDefinition.getDataShapeName); ignored for services and subscriptions."
      },
      "offset" : {
        "type" : "integer",
        "description" : "Zero-based list offset (default 0)."
      },
      "maxItems" : {
        "type" : "integer",
        "description" : "Max list rows (default 80, max 200)."
      }
    },
    "required" : [ "thingName" ]
  }
}
```

### describe_entity_schema

Size chars: `2868`

```json
{
  "name" : "describe_entity_schema",
  "description" : "Facet-bounded schema read for ThingTemplate, ThingShape, or DataShape: summary, paginated property/service/event lists, DataShape fields, or one public service definition (parameters + result type). Uses visibility-aware entity resolution. Does not return service implementation bodies. Code DESCRIBE_ENTITY_SCHEMA_PLATFORM_UNAVAILABLE means required platform schema APIs were missing, failed, or threw during template/shape member reads (local or effective list facets) — not an authoritative empty schema. **Default** inspection for schema entities: stay on this tool's facets. Tier B / persisted replay may still run **get_entity** (executor-only, not merged into the LLM tool list) when one stamped **parler.entity.metadata.v1** combined payload is required.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "entityType" : {
        "enum" : [ "ThingTemplate", "ThingShape", "DataShape" ],
        "type" : "string",
        "description" : "Schema entity type. Thing instances are not accepted — use discover_thing_members + get_property_values for concrete Things."
      },
      "entityName" : {
        "type" : "string",
        "description" : "Canonical name of the ThingTemplate, ThingShape, or DataShape."
      },
      "facet" : {
        "enum" : [ "summary", "properties", "services", "events", "fields", "service" ],
        "type" : "string",
        "description" : "Facet to return. v1: summary (counts), list facets properties/services/events (ThingTemplate/ThingShape), fields (DataShape), singular service (memberName required). Singular property/event/field and subscriptions are not supported in v1 — use additional **describe_entity_schema** calls per facet; do not treat a single combined metadata dump as the default path. Persisted replay may still invoke **get_entity** (off-list) only when a stamped **parler.entity.metadata.v1** payload is explicitly required."
      },
      "memberName" : {
        "type" : "string",
        "description" : "Required when facet is \"service\" — exact service name."
      },
      "scope" : {
        "enum" : [ "effective", "local" ],
        "type" : "string",
        "description" : "effective (default): inherited definitions via platform effective APIs. local: ThingTemplate/ThingShape — members from getInstanceShape() only (not merged instance service/event lists); DataShape — getDataShape() before effective merge."
      },
      "namePrefix" : {
        "type" : "string",
        "description" : "Optional name prefix filter for list facets."
      },
      "category" : {
        "type" : "string",
        "description" : "Optional category filter for property/service/event list facets."
      },
      "baseType" : {
        "type" : "string",
        "description" : "Optional base type filter for property/field list facets."
      },
      "dataShape" : {
        "type" : "string",
        "description" : "Optional INFOTABLE dataShape filter for property/field list facets."
      },
      "offset" : {
        "type" : "integer",
        "description" : "Zero-based offset for list facets (default 0)."
      },
      "maxItems" : {
        "type" : "integer",
        "description" : "Page size for list facets (default 80, max 200)."
      }
    },
    "required" : [ "entityType", "entityName" ]
  }
}
```

### query_entities

Size chars: `6549`

```json
{
  "name" : "query_entities",
  "description" : "List Things implementing a ThingTemplate or ThingShape. **Model keys** such as **`Stream`**, **`DataTable`**, **`ValueStream`** (App User count/list language) normally refer to platform **ThingTemplate** / **ThingShape** entities — agent Phase 0.5 resolves them via **`GenericThing.GetIncomingDependencies`** after taxonomy (**docs/agent/key-resolution.md** Phase 0.5). When **cached taxonomy rows** or resolver tools supply a Thing parent for taxonomy-scoped questions, do **not** pick thingTemplate/thingShape by guesswork — use **`resolve_asset_type`** then **query_entities_by_taxonomy** when listing implementors of an application asset class (**taxonomy first** beats Phase 0.5). Provide exactly one of thingTemplate or thingShape (both optional in JSON schema but required in practice). Default columnMode is lean: platform basicPropertyNames=name only and empty propertyNames (not all properties). Use includeDescription, includeIsSystemObject, includeTags, includeConcreteTemplate (ThingTemplate parent only), or widePropertyColumns for wide platform defaults. Prefers in-process QueryImplementingThingsOptimizedWithTotalCount (then Optimized, then QueryImplementingThings). Optional modelTags for the service-level tags filter; optional query for field filters. Optional **hierarchyNodeId** (direct NetworkID from Host Context) or **hierarchyNodeName** (conversation resolve path): **GetAssetList** intersect when non-blank (hierarchy-network-services.md §6; precedence **hierarchyNodeId** > **hierarchyNodeName**). Optional **intersectThingNames** + **intersectExpandHasMore** shrink the current page to the ∩ with an expand Thing-name set (entity-hierarchy §6). On success totalRows is the full match count when the platform supplies totalCount, or when the tool infers it (totalRowsInferred=true: last page or empty probe page). If totalRows is null and totalRowsInferred is absent, read note — hasMore is heuristic when returnedRows == maxItems. Do not expect a totalCount key on the tool response. Large row sets use the same cache as fetch_cached_result. Not fuzzy search — use spotlight_search for Spotlight.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "entityType" : {
        "type" : "string",
        "description" : "Use Thing (default semantics: list Thing instances implementing the template or shape)."
      },
      "thingTemplate" : {
        "type" : "string",
        "description" : "ThingTemplate name. Exactly one of thingTemplate or thingShape is required."
      },
      "thingShape" : {
        "type" : "string",
        "description" : "ThingShape name. Exactly one of thingTemplate or thingShape is required."
      },
      "maxItems" : {
        "type" : "integer",
        "description" : "Page size / max rows (default 50, max 200). Maps to platform maxItems / nMaxItems."
      },
      "offset" : {
        "type" : "integer",
        "description" : "Row offset for pagination when using QueryImplementingThingsOptimized* (default 0)."
      },
      "namePrefix" : {
        "type" : "string",
        "description" : "Optional name mask / prefix passed to platform nameMask when supported."
      },
      "withPermissions" : {
        "type" : "boolean",
        "description" : "Maps to platform withPermissions on Optimized services (include read/update/delete permission columns). Default false."
      },
      "withData" : {
        "type" : "boolean",
        "description" : "Deprecated alias: treated like withPermissions for backward compatibility (Optimized services have no withData parameter)."
      },
      "includeDescription" : {
        "type" : "boolean",
        "description" : "Add basic column description (semantic / UI). Default false; default basic columns are name only."
      },
      "includeIsSystemObject" : {
        "type" : "boolean",
        "description" : "Add basic column isSystemObject. Default false."
      },
      "includeTags" : {
        "type" : "boolean",
        "description" : "Add basic column tags. Default false."
      },
      "includeConcreteTemplate" : {
        "type" : "boolean",
        "description" : "Add propertyNames column thingTemplate (leaf template per row). Only when thingTemplate is set; invalid with thingShape."
      },
      "widePropertyColumns" : {
        "type" : "boolean",
        "description" : "If true, omit basicPropertyNames/propertyNames so the platform may return all columns (heavy). Default false: pass name-only basics + empty propertyNames EntityList (docs/agent/query_with_total_count.md §10)."
      },
      "query" : {
        "type" : "object",
        "description" : "Optional platform query object (filters/sorts). See docs/agent/query_capability.md. Only Shape/Template fields for Optimized."
      },
      "modelTags" : {
        "type" : "array",
        "items" : {
          "type" : "object",
          "properties" : {
            "vocabulary" : {
              "type" : "string"
            },
            "vocabularyTerm" : {
              "type" : "string"
            }
          },
          "required" : [ "vocabulary", "vocabularyTerm" ]
        },
        "description" : "Optional platform Model tags on the service `tags` parameter (AND). For tag-based row filters inside a query object use the `query` parameter instead — different mechanism."
      },
      "hierarchyNodeId" : {
        "type" : "string",
        "description" : "Optional **hierarchy node id** (NetworkID) — **direct path** from Host Context or the page; calls **GetAssetList(hierarchyNodeId)** without **ResolveNetworkID**. Precedence after explicit **intersectThingNames**: **hierarchyNodeId** before **hierarchyNodeName**. Failures: **HIERARCHY_ASSET_LIST_FAILED** / **HIERARCHY_SCOPED_EMPTY** — **no** fallback to **hierarchyNodeName**."
      },
      "hierarchyNodeName" : {
        "type" : "string",
        "description" : "Optional **hierarchy node display-name fragment** from the **user message / conversation** (e.g. region or site). **Absolute first priority** for hierarchy-scoped intersect: when non-blank, the server calls **ResolveNetworkID(hierarchyNodeName)** then **GetAssetList** on the **single** resolved **NetworkID** (**0** rows → **HIERARCHY_RESOLVE_NOT_FOUND**; **2+** → **HIERARCHY_RESOLVE_AMBIGUOUS**; service failure → **HIERARCHY_RESOLVE_FAILED**). Omit when no hierarchy scope applies."
      },
      "intersectThingNames" : {
        "type" : "array",
        "maxItems" : 5000,
        "items" : {
          "type" : "string"
        },
        "description" : "Optional expand/hierarchy Thing name set **B**: keep only rows whose **name** is in this set (**String.equals**, no case fold). **preIntersectMatchCount** counts rows on **this QIT page** before ∩ (per-page, not the full platform match count unless that page is the full set). With intersect, success JSON adds **queryHasMore**, **expandHasMore**, and **hasMore** = **queryHasMore** OR **expandHasMore** (see **CONTRACTS/API_CONTRACT.md** and **entity-hierarchy.md** §6). Empty or non-text array entries are dropped and may be noted on **note**."
      },
      "intersectExpandHasMore" : {
        "type" : "boolean",
        "description" : "When **intersectThingNames** is used: sets **expandHasMore** (expand side may list more Things). **hasMore** ORs **queryHasMore** (QIT / internal listing truncation) with **expandHasMore**. Default false."
      }
    },
    "required" : [ "entityType" ]
  }
}
```

### query_entities_by_taxonomy

Size chars: `4594`

```json
{
  "name" : "query_entities_by_taxonomy",
  "description" : "List Things under one ThingTemplate or ThingShape parent with taxonomy-style column projection. Use **resolve_asset_type** first when the user gives application asset type text — copy **entityType**/**entityName** from that tool (do not invent parent names). When the user asks for Thing **names** or columns listed in the resolver row **criticalProperties** array, join them into **CriticalProperties** here as a semicolon-separated list so the tool can project them (along with **name**). Interim tool until the platform can query Things by **both** ThingTemplate and ThingShape together. Calls QueryImplementingThingsOptimizedWithTotalCount on the given ThingTemplate or ThingShape with name-only QIT columns and maxItems 5000, walks rootEntityList, infers projected column baseTypes from the first resolvable Thing (with a live-property fallback when instance metadata omits a field that is still readable), filters rows where **any** LookupProperties entry equals the Thing (**OR**), and returns success JSON per docs/agent/AGENT-TAXONOMY.md §5.2.1 (resultKind EMPTY | INLINE | LARGE; only INLINE/EMPTY include rootEntityList; LARGE adds sampleRootEntityList + cacheId + hint; totalCount is always the filtered count). Optional **hierarchyNodeName** (from the **conversation**): same as **query_entities** when non-blank. Optional **intersectThingNames** / **intersectExpandHasMore** apply ∩ with an expand Thing-name set after projection (CONTRACTS/API_CONTRACT.md; entity-hierarchy §6). LARGE uses the same session cache as fetch_cached_result.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "EntityType" : {
        "type" : "string",
        "description" : "Exactly \"ThingTemplate\" or \"ThingShape\" (case-sensitive). Only one parent dimension per call."
      },
      "EntityName" : {
        "type" : "string",
        "description" : "ThingTemplate or ThingShape name. Prefer the **exact** strings returned by **resolve_asset_type** when the user refers to an application asset class. Existence is not pre-checked; invalid names fail at runtime."
      },
      "CriticalProperties" : {
        "type" : "string",
        "description" : "Semicolon-separated property names (trimmed; empty segments dropped). These columns are included in the projected rootEntityList (along with name). Duplicate \"name\" is allowed."
      },
      "AdditionalProperties" : {
        "type" : "string",
        "description" : "Optional extra semicolon-separated property names; same parsing rules as CriticalProperties."
      },
      "LookupProperties" : {
        "type" : "object",
        "description" : "JSON object: propertyName → value. A Thing matches if **any** entry matches the live property value (**OR**). Exact match only (no wildcards/regex yet). Omit or {} to skip lookup filtering."
      },
      "hierarchyNodeId" : {
        "type" : "string",
        "description" : "Optional **hierarchy node id** (NetworkID) — **direct path** from Host Context or the page; calls **GetAssetList(hierarchyNodeId)** without **ResolveNetworkID**. Precedence after explicit **intersectThingNames**: **hierarchyNodeId** before **hierarchyNodeName**. Failures: **HIERARCHY_ASSET_LIST_FAILED** / **HIERARCHY_SCOPED_EMPTY** — **no** fallback to **hierarchyNodeName**."
      },
      "hierarchyNodeName" : {
        "type" : "string",
        "description" : "Optional **hierarchy node display-name fragment** from the **user message / conversation** (e.g. region or site). **Absolute first priority** for hierarchy-scoped intersect: when non-blank, the server calls **ResolveNetworkID(hierarchyNodeName)** then **GetAssetList** on the **single** resolved **NetworkID** (**0** rows → **HIERARCHY_RESOLVE_NOT_FOUND**; **2+** → **HIERARCHY_RESOLVE_AMBIGUOUS**; service failure → **HIERARCHY_RESOLVE_FAILED**). Omit when no hierarchy scope applies."
      },
      "intersectThingNames" : {
        "type" : "array",
        "maxItems" : 5000,
        "items" : {
          "type" : "string"
        },
        "description" : "Optional expand/hierarchy Thing name set **B**: keep only rows whose **name** is in this set (**String.equals**, no case fold). **preIntersectMatchCount** counts rows in the **full** post-**LookupProperties** filtered set before ∩ (not paginated like a single QIT page). With intersect, success JSON adds **queryHasMore**, **expandHasMore**, and **hasMore** = **queryHasMore** OR **expandHasMore** (see **CONTRACTS/API_CONTRACT.md** and **entity-hierarchy.md** §6). Empty or non-text array entries are dropped and may be noted on **note**."
      },
      "intersectExpandHasMore" : {
        "type" : "boolean",
        "description" : "When **intersectThingNames** is used: sets **expandHasMore** (expand side may list more Things). **hasMore** ORs **queryHasMore** (QIT / internal listing truncation) with **expandHasMore**. Default false."
      }
    },
    "required" : [ "EntityType", "EntityName" ]
  }
}
```

### list_asset_types

Size chars: `433`

```json
{
  "name" : "list_asset_types",
  "description" : "List application asset types from `/taxonomies/asset-types.json` when that v3 object map is configured and loaded (v3 identity array may be absent). For v2 object identity-types.json (version 2) this lists flattened rows from that file. Use when the user asks how many asset types exist or wants the catalog without guessing ThingTemplates.",
  "input_schema" : {
    "type" : "object",
    "properties" : { }
  }
}
```

### resolve_asset_type

Size chars: `436`

```json
{
  "name" : "resolve_asset_type",
  "description" : "Map user asset type text to an asset type key, ThingTemplate/ThingShape parent, and **criticalProperties** name list. Call **resolve_thing** when the user names a specific asset instance.",
  "input_schema" : {
    "properties" : {
      "text" : {
        "type" : "string",
        "description" : "User-facing asset type phrase to resolve to a configured asset type key and ThingWorx parent."
      }
    },
    "type" : "object",
    "required" : [ "text" ]
  }
}
```

### resolve_thing

Size chars: `678`

```json
{
  "name" : "resolve_thing",
  "description" : "Resolve user text to a unique canonical Thing name using v3 identity-types.json array rules. Requires at least one loaded v3 identity rule (asset-types.json is optional unless you pass **assetTypeKey**). Optional **assetTypeKey** narrows matching rules when asset-type rows are loaded.",
  "input_schema" : {
    "properties" : {
      "text" : {
        "type" : "string",
        "description" : "User-facing identifier (display name, serial, suffix, or canonical Thing name)."
      },
      "assetTypeKey" : {
        "type" : "string",
        "description" : "Optional exact **key** from **list_asset_types** / asset-types.json to narrow identity rules to that asset class."
      }
    },
    "type" : "object",
    "required" : [ "text" ]
  }
}
```

### list_entities_by_type

Size chars: `2137`

```json
{
  "name" : "list_entities_by_type",
  "description" : "List metadata entities of a given collection type via Resource EntityServices (GetEntityList or GetEntityListByRegEx). Never pass **`entityCollectionType=Stream`**, **`DataTable`**, or **`ValueStream`** when the user asks how many/list **Things** — those are **`query_entities`** **thingTemplate**/ **thingShape** keys, not **`GetEntityList`** **`type`** strings (success would mislead toward zero). If this tool returns **`code=ENTITY_COLLECTION_TYPE_RESOLVED_AS_MODEL_KEY`**, **immediately** call **`query_entities`** again using **`repair.arguments`** **before** telling the user the count is zero. This is **not** a substitute for **list_asset_types** / **resolve_asset_type**: do not use name-mask listing here to answer “how many asset types” or to invent **AssetType** inventory when application taxonomy is available (see llm_tool_routing_guide.txt **Asset taxonomy**). For Thing instances under a template/shape use query_entities instead. Optional tags filter uses platform Model tags (empty = match all). Large results use the same session cache as invoke_service / fetch_cached_result.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "entityCollectionType" : {
        "type" : "string",
        "description" : "ThingWorx collection type string (e.g. ThingTemplate, ThingShape, Mashup, DataShape, User). NOT Thing — use query_entities or spotlight_search."
      },
      "nameMask" : {
        "type" : "string",
        "description" : "Optional name pattern. Semantics depend on useRegEx: SQL LIKE vs regex."
      },
      "useRegEx" : {
        "type" : "boolean",
        "description" : "If true, calls GetEntityListByRegEx; if false, GetEntityList (SQL LIKE). Default false."
      },
      "maxItems" : {
        "type" : "integer",
        "description" : "Max rows (default 50, max 200). Platform may stop earlier when enough matches."
      },
      "tags" : {
        "type" : "array",
        "items" : {
          "type" : "object",
          "properties" : {
            "vocabulary" : {
              "type" : "string"
            },
            "vocabularyTerm" : {
              "type" : "string"
            }
          },
          "required" : [ "vocabulary", "vocabularyTerm" ]
        },
        "description" : "Optional Model tags filter (AND). Same as invoke_service TAGS: [{\"vocabulary\":\"...\",\"vocabularyTerm\":\"...\"}]. Omit = no tag filter."
      }
    },
    "required" : [ "entityCollectionType" ]
  }
}
```

### get_property_values

Size chars: `1126`

```json
{
  "name" : "get_property_values",
  "description" : "Read current values of multiple Thing properties in one call (batch, max 40). **thingName** must be a canonical ThingWorx name; non-canonical labels return **IDENTITY_RESOLUTION_REQUIRED**; **`recoveryHint`** (to **resolve_thing**) is included only when v3 identity rules are loaded for this agent turn. propertyNames must be exact ThingWorx property names; use discover_properties first when only a business label is known. Returns per-property ok/value or error; PROPERTY_METADATA_UNRESOLVED means the name was not resolved and no value was read.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "thingName" : {
        "type" : "string",
        "description" : "Canonical ThingWorx Thing name (exact platform **Thing** name). If the user supplied a display label, serial number, suffix, or any uncertain asset identifier, call **resolve_thing** first (v3 identity taxonomy), then retry with **matches[0].name** from a **UNIQUE** result."
      },
      "propertyNames" : {
        "type" : "array",
        "maxItems" : 40,
        "items" : {
          "type" : "string"
        },
        "description" : "Property names to read (max 40)"
      }
    },
    "required" : [ "thingName", "propertyNames" ]
  }
}
```

### query_property_history

Size chars: `4152`

```json
{
  "name" : "query_property_history",
  "description" : "Query time-series history for a Thing property. **thingName** must be canonical; non-canonical labels return **IDENTITY_RESOLUTION_REQUIRED**; **`recoveryHint`** (to **resolve_thing**) is included only when v3 identity rules are loaded for this agent turn. The server picks the correct platform path after reading property metadata (numeric vs value-stream). Pass ISO-8601 **startTime**/**endTime** (UTC …Z recommended) when the user implies a bounded window so chart and query bounds align; or use **calendarPhrase** / **relativeDuration** when ISO bounds are omitted. Numeric trends support optional **actions** aggregates and Parler auto chart wire; non-numeric histories return compact **VALUE_STREAM_HISTORY_INLINE** evidence (sampleRows + **cacheId**, no points array, no auto chart).",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "thingName" : {
        "type" : "string",
        "description" : "Canonical ThingWorx Thing name (exact platform **Thing** name). If the user supplied a display label, serial number, suffix, or any uncertain asset identifier, call **resolve_thing** first (v3 identity taxonomy), then retry with **matches[0].name** from a **UNIQUE** result."
      },
      "propertyName" : {
        "type" : "string",
        "description" : "Property name. The server resolves the base type: NUMBER/INTEGER/LONG use the numeric trend path (optional aggregates + Parler auto chart when applicable); other logged types use value-stream QueryPropertyHistory with bounded compact evidence (no auto chart in 0.1.162)."
      },
      "startTime" : {
        "type" : "string",
        "description" : "Start time (ISO-8601). Omit with endTime for platform default window. Alias: start."
      },
      "endTime" : {
        "type" : "string",
        "description" : "End time (ISO-8601). Optional. Alias: end."
      },
      "calendarPhrase" : {
        "type" : "string",
        "description" : "Optional: **today** / **yesterday** / **tomorrow** (single day, user's **user_timezone** IANA). Mutually exclusive with startTime/endTime and relativeDuration."
      },
      "relativeDuration" : {
        "type" : "string",
        "description" : "Optional: duration ending **now** (e.g. **30m**, **24h**) — closed-open semantics per `docs/agent/time-interpretation.md`. Mutually exclusive with startTime/endTime and calendarPhrase."
      },
      "maxRows" : {
        "type" : "integer",
        "description" : "Max rows to read from the platform history service (default 1000, cap 5000). Alias: maxPoints. LLM-visible success bodies are always compact (sampleRows + cacheId; aggregates when requested)."
      },
      "actions" : {
        "type" : "array",
        "items" : {
          "type" : "string"
        },
        "description" : "Optional aggregate actions for **numeric** properties only (mean, min, max, sum, stddev, variance, median, count, first, last). Empty or omit for raw series. **Non-numeric:** omit actions — non-empty actions return NUMERIC_ACTIONS_UNSUPPORTED_FOR_PROPERTY_TYPE."
      },
      "y_reference_lines" : {
        "type" : "array",
        "maxItems" : 12,
        "description" : "Optional horizontal Y-axis lines (SPC limits, thresholds). Drawn on auto chart wire frame (numeric path).",
        "items" : {
          "type" : "object",
          "properties" : {
            "y" : {
              "type" : "number"
            },
            "label" : {
              "type" : "string"
            },
            "role" : {
              "enum" : [ "usl", "ucl", "lcl", "lsl", "target", "limit", "warning" ],
              "type" : "string",
              "description" : "usl/lsl = spec limits; ucl/lcl = control limits; target = center line; limit = generic; warning = advisory."
            }
          },
          "required" : [ "y" ]
        }
      },
      "kind" : {
        "enum" : [ "line", "bar", "scatter" ],
        "type" : "string",
        "description" : "Chart kind for automatic chart frame on numeric trends (default line)."
      },
      "title" : {
        "type" : "string",
        "description" : "Optional chart title override (numeric path)."
      },
      "x_label" : {
        "type" : "string",
        "description" : "Optional X-axis label (numeric path)."
      },
      "y_label" : {
        "type" : "string",
        "description" : "Optional Y-axis label (numeric path)."
      },
      "requestedTimeRange" : {
        "type" : "object",
        "properties" : {
          "start" : {
            "type" : "string",
            "description" : "ISO-8601 instant (UTC …Z recommended). Chart X-axis span when query bounds omitted."
          },
          "end" : {
            "type" : "string",
            "description" : "ISO-8601 instant (UTC …Z recommended)."
          }
        },
        "description" : "Optional chart window for the auto ChartBlock when startTime/endTime are not passed (numeric path). Prefer always setting startTime and endTime so the query and chart share the same bounds."
      }
    },
    "required" : [ "thingName", "propertyName" ]
  }
}
```

### query_stream_data

Size chars: `1522`

```json
{
  "name" : "query_stream_data",
  "description" : "Query tabular stream rows via platform **QueryStreamData** on a Stream Thing. Prefer this over invoke_service for natural-language time windows: **calendarPhrase**, **relativeDuration**, or ISO **startTime**/**endTime** (aliases start/end) — same resolver as query_property_history. Large results use the INFOTABLE_LARGE envelope (sampleRows; cacheId when server cached).",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "thingName" : {
        "type" : "string",
        "description" : "Name of the **Stream** Thing (implements Stream / RemoteStream QueryStreamData)."
      },
      "startTime" : {
        "type" : "string",
        "description" : "Start time (ISO-8601). Omit both bounds for platform-default window. Alias: start."
      },
      "endTime" : {
        "type" : "string",
        "description" : "End time (ISO-8601). Alias: end."
      },
      "calendarPhrase" : {
        "type" : "string",
        "description" : "Optional: **today** / **yesterday** / **tomorrow** (user **user_timezone** IANA). Mutually exclusive with startTime/endTime and relativeDuration."
      },
      "relativeDuration" : {
        "type" : "string",
        "description" : "Optional: duration ending **now** (e.g. **30m**, **24h**). Mutually exclusive with ISO bounds and calendarPhrase."
      },
      "maxItems" : {
        "type" : "integer",
        "description" : "Cap rows returned (default 500, max 5000). Maps to QueryStreamData maxItems."
      },
      "oldestFirst" : {
        "type" : "boolean",
        "description" : "Sort oldest-first when true (default false)."
      },
      "source" : {
        "type" : "string",
        "description" : "Optional stream entry source filter (QueryStreamData **source** parameter)."
      }
    },
    "required" : [ "thingName" ]
  }
}
```

### query_alert_summary

Size chars: `2156` (re-snapshot 2026-06-28 — multi-Thing **`thingNames[]`**)

```json
{
  "name" : "query_alert_summary",
  "description" : "Current alert **summary** for one or more Things via Resource AlertFunctions.QueryAlertSummaryForThing (in-memory snapshot, not history). **thingNames** must be canonical ThingWorx names; non-canonical labels return per-Thing **IDENTITY_RESOLUTION_REQUIRED** entries when other Things succeed. Multiple Things return **ALERT_SUMMARY_MULTI** rollups (counts + topAlerts); a single Thing returns the usual INFOTABLE envelope. **`recoveryHint`** (to **resolve_thing**) is included only when v3 identity rules are loaded for this agent turn. Optional **sort** maps to QUERY **sorts** (not combinable with **advancedQuery.sorts**). Large single-Thing tables use INFOTABLE_LARGE (sampleRows; cacheId when server cached). Prefer over invoke_service for correct parameter mapping.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "thingNames" : {
        "type" : "array",
        "minItems" : 1,
        "maxItems" : 25,
        "items" : {
          "type" : "string",
          "description" : "Canonical ThingWorx Thing name (exact platform **Thing** name). If the user supplied a display label, serial number, suffix, or any uncertain asset identifier, call **resolve_thing** first (v3 identity taxonomy), then retry with **matches[0].name** from a **UNIQUE** result."
        },
        "description" : "One or more canonical ThingWorx names. Pass a single-element array for one Thing. Non-canonical names are reported per-Thing in identityErrors[] (partial success when at least one resolves). Maximum 25 names per call."
      },
      "ackState" : {
        "enum" : [ "all", "acknowledged", "unacknowledged" ],
        "type" : "string",
        "description" : "Default all. Use unacknowledged for active-only."
      },
      "propertyName" : {
        "type" : "string",
        "description" : "Optional: filter to one source property."
      },
      "alertName" : {
        "type" : "string",
        "description" : "Optional QUERY EQ on alert name."
      },
      "alertType" : {
        "type" : "string",
        "description" : "Optional QUERY EQ on alertType."
      },
      "priorityMin" : {
        "type" : "integer"
      },
      "priorityMax" : {
        "type" : "integer"
      },
      "sort" : {
        "enum" : [ "default", "timestamp_asc", "timestamp_desc", "priority_asc", "priority_desc" ],
        "type" : "string",
        "description" : "Optional QUERY sort for summary rows. Mutually exclusive with **advancedQuery.sorts**."
      },
      "limit" : {
        "type" : "integer",
        "description" : "maxItems (default 100, max 500)."
      },
      "advancedQuery" : {
        "type" : "string",
        "description" : "Optional raw ThingWorx QUERY JSON merged AND with typed filters."
      }
    },
    "required" : [ "thingNames" ]
  }
}
```

### query_alert_history

Size chars: `2901`

```json
{
  "name" : "query_alert_history",
  "description" : "Alert **history** timeline for one Thing via AlertFunctions.QueryAlertHistory. **thingName** must be canonical; non-canonical labels return **IDENTITY_RESOLUTION_REQUIRED**; **`recoveryHint`** (to **resolve_thing**) is included only when v3 identity rules are loaded for this agent turn. Always bounded; response includes appliedStartTime/appliedEndTime, timeRangeSource, optional appliedTimePreset / implicitDefaultWindowDays, oldestFirst, and **historyQueryResource** (resource-backed path; uses platform summary-manager filtered stream per installed server). Optional **calendarPhrase** (today/yesterday/tomorrow in the user's local timezone) or **relativeDuration** (e.g. 30m, 24h) when explicit ISO bounds are omitted — mutual exclusion with startTime/endTime and timePreset.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "thingName" : {
        "type" : "string",
        "description" : "Canonical ThingWorx Thing name (exact platform **Thing** name). If the user supplied a display label, serial number, suffix, or any uncertain asset identifier, call **resolve_thing** first (v3 identity taxonomy), then retry with **matches[0].name** from a **UNIQUE** result."
      },
      "startTime" : {
        "type" : "string",
        "description" : "ISO-8601 start; optional (default window applied)."
      },
      "endTime" : {
        "type" : "string",
        "description" : "ISO-8601 end; optional (defaults to now)."
      },
      "calendarPhrase" : {
        "type" : "string",
        "description" : "Optional English phrase naming **one** local calendar day: **today**, **yesterday**, or **tomorrow** (word-boundary match). Uses host **user_timezone** (IANA). Mutually exclusive with startTime/endTime, timePreset, and relativeDuration."
      },
      "relativeDuration" : {
        "type" : "string",
        "description" : "Optional Parler duration ending **now** (e.g. **30m**, **24h**, **7d**) — closed-open window per `docs/agent/time-interpretation.md` §4.4. Mutually exclusive with startTime/endTime, timePreset, and calendarPhrase."
      },
      "timePreset" : {
        "enum" : [ "last_1h", "last_24h", "last_7d" ],
        "type" : "string",
        "description" : "Optional shortcut when **both** startTime and endTime are omitted: fixed span ending at **now** (last_1h / last_24h / last_7d). Must not be set together with startTime or endTime."
      },
      "alertName" : {
        "type" : "string"
      },
      "propertyName" : {
        "type" : "string",
        "description" : "QUERY EQ on sourceProperty."
      },
      "alertType" : {
        "type" : "string"
      },
      "priorityMin" : {
        "type" : "integer"
      },
      "priorityMax" : {
        "type" : "integer"
      },
      "order" : {
        "enum" : [ "newest_first", "oldest_first" ],
        "type" : "string",
        "description" : "Optional sort direction (newest events first vs oldest first). If set, must agree with **oldestFirst** when both are present."
      },
      "oldestFirst" : {
        "type" : "boolean",
        "description" : "Legacy boolean for platform oldestFirst (default false = newest first). Prefer **order** for LLM clarity."
      },
      "limit" : {
        "type" : "integer",
        "description" : "maxItems (default 100, max 500)."
      },
      "advancedQuery" : {
        "type" : "string"
      }
    },
    "required" : [ "thingName" ]
  }
}
```

### acknowledge_alerts

Size chars: `1723`

```json
{
  "name" : "acknowledge_alerts",
  "description" : "Acknowledge alerts on AlertFunctions. **thingName** must be a canonical ThingWorx name; non-canonical labels return **IDENTITY_RESOLUTION_REQUIRED** before any acknowledge or summary probe (**no** side effects on preflight failure). Default **specific_alerts** requires propertyName; **property_all** is explicit bulk on that property. Success JSON may include **platformAckService**, **countsAvailable** (boolean), **ackMessage** (echo of optional **message** argument), and **note**; **specific_alerts** summary probe (**QueryAlertSummaryForThing**) and ack service failures return **status** error with normalized **code** (e.g. PLATFORM_ALERT_PERMISSION). empty **specific_alerts** uses **message** for the human-readable status line.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "thingName" : {
        "type" : "string",
        "description" : "Canonical ThingWorx Thing name (exact platform **Thing** name). If the user supplied a display label, serial number, suffix, or any uncertain asset identifier, call **resolve_thing** first (v3 identity taxonomy), then retry with **matches[0].name** from a **UNIQUE** result."
      },
      "mode" : {
        "enum" : [ "specific_alerts", "property_all" ],
        "type" : "string",
        "description" : "Default specific_alerts: summary probe then AcknowledgeAlertFromSummary; when exactly one unacked row matches propertyName without alertName, may use narrow AcknowledgeAlert (same property scope only)."
      },
      "propertyName" : {
        "type" : "string",
        "description" : "Required for specific_alerts and property_all."
      },
      "alertName" : {
        "type" : "string",
        "description" : "Optional: narrow specific_alerts to one alert name."
      },
      "message" : {
        "type" : "string",
        "description" : "Optional ack message."
      }
    },
    "required" : [ "thingName" ]
  }
}
```

### set_property_value

Size chars: `921`

```json
{
  "name" : "set_property_value",
  "description" : "Request to write a Thing property. Requires human approval on Parler AlwaysOn before execution. Always set base_type when known (discover_thing_members / discover_properties / get_property_values). See docs/archived/data-operation-solution.md §1.3.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "thing_name" : {
        "type" : "string",
        "description" : "Target Thing name"
      },
      "property_name" : {
        "type" : "string",
        "description" : "Property name"
      },
      "base_type" : {
        "type" : "string",
        "description" : "ThingWorx BaseType enum (STRING, NUMBER, INTEGER, LONG, BOOLEAN, DATETIME, …). Omit only if unsure — server may infer from property metadata."
      },
      "value" : {
        "anyOf" : [ {
          "type" : "string"
        }, {
          "type" : "number"
        }, {
          "type" : "integer"
        }, {
          "type" : "boolean"
        } ],
        "description" : "Literal matching base_type (STRING→string, NUMBER→number, BOOLEAN→boolean, DATETIME→ISO-8601 string)."
      }
    },
    "required" : [ "thing_name", "property_name", "value" ]
  }
}
```

### spotlight_search

Size chars: `610`

```json
{
  "name" : "spotlight_search",
  "description" : "Fuzzy search across ThingWorx metadata via platform SearchFunctions.SpotlightSearchV2 (same as REST Spotlight; invoked in-JVM, not HTTP). For structured filters use query_entities.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "query" : {
        "type" : "string",
        "description" : "Spotlight search text (names, descriptions, etc.)"
      },
      "maxItems" : {
        "type" : "integer",
        "description" : "Max hits (default 30, max 100)"
      },
      "entityTypes" : {
        "type" : "array",
        "description" : "Optional model tags / types filter (reserved; not all platforms wired yet)",
        "items" : {
          "type" : "string"
        }
      }
    },
    "required" : [ "query" ]
  }
}
```

### search_document_chunks

Size chars: `1516`

```json
{
  "name" : "search_document_chunks",
  "description" : "Find document chunks relevant to a health issue, alarm, component, or user question. Use after live status is known when the user needs manual/troubleshooting guidance. When resolve_document_set returned a non-empty documents[], pass those ids as documentIds to scope this search to the resolved set. Returns ranked metadata and snippets; call get_document_chunk for full markdown to cite.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "query" : {
        "type" : "string",
        "description" : "Natural-language query built from the user question or normalized health issue."
      },
      "signals" : {
        "type" : "array",
        "description" : "Optional alarms, properties, symptoms, or components.",
        "items" : {
          "type" : "object",
          "properties" : {
            "kind" : {
              "type" : "string"
            },
            "name" : {
              "type" : "string"
            },
            "value" : {
              "type" : "string"
            }
          }
        }
      },
      "assetContext" : {
        "type" : "object",
        "description" : "Optional asset context such as asset model, component, or document type hints."
      },
      "documentTypes" : {
        "type" : "array",
        "description" : "Optional document type filters such as operations_manual or troubleshooting_guide.",
        "items" : {
          "type" : "string"
        }
      },
      "documentIds" : {
        "type" : "array",
        "description" : "Optional explicit document id filter. When resolve_document_set returns a non-empty documents[], pass those documents[].documentId values here to scope this search to the resolved set (selectionMode documentIds-filter).",
        "items" : {
          "type" : "string"
        }
      },
      "limit" : {
        "type" : "integer",
        "description" : "Maximum matches to return. Clamped to configured bounds."
      }
    },
    "required" : [ ]
  }
}
```

### get_document_chunk

Size chars: `527`

```json
{
  "name" : "get_document_chunk",
  "description" : "Fetch full markdown and FileRepository source links for one chunk from search results. sourceLinks[].href is the canonical clickable PDF target (includes #page= when applicable); when citing this chunk in the final answer, copy each href into a markdown link [label](href). Do not cite manual text unless it came from this tool or a search snippet.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "docId" : {
        "type" : "string"
      },
      "chunkId" : {
        "type" : "string"
      }
    },
    "required" : [ "docId", "chunkId" ]
  }
}
```

### resolve_document_set

Size chars: `759`

```json
{
  "name" : "resolve_document_set",
  "description" : "Resolve the bounded set of documents that apply to a given asset, to scope document search before search_document_chunks. Pass the asset model or Thing name as key. Returns documents[] (the in-scope document ids) and a resolverSource diagnostic. When documents is non-empty, pass documents[].documentId as the documentIds argument to search_document_chunks to scope retrieval. If documents is empty (resolverSource default-empty), no confident scope was found; proceed with a normal search_document_chunks call.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "key" : {
        "type" : "string",
        "description" : "Asset/Thing identifier to scope retrieval — typically the bound Thing's asset model or name."
      }
    },
    "required" : [ "key" ]
  }
}
```

### get_agent_skill

Size chars: `866`

```json
{
  "name" : "get_agent_skill",
  "description" : "Loads the full instructions text for a registered agent skill (Service or repository source). The chat system prompt lists skills with metadata only; call this tool to retrieve the complete body when relevant. On success the result is the skill body string; on failure a JSON object with `status`, `code`, and `message`.",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "skill_name" : {
        "type" : "string",
        "description" : "Short skill id of a **registered** agent skill. Skills are loaded from the configured `configurationRepository` FileRepository (`/skills/<id>/SKILL.md`). The per-turn catalog lists each skill’s `source` (always `repository` when present). Pass the short id only (same token as `/SkillName` in the user message), never the full Service name. Example ids: OrderWorkflow, alert_query."
      }
    },
    "required" : [ "skill_name" ]
  }
}
```

### start_playbook

Registry-driven at runtime when the playbook catalog is loaded. Example wire shape (two loaded ids `alpha`, `beta`):

```json
{
  "name" : "start_playbook",
  "description" : "Start a registered Playbook workflow with structured parameters. See the per-turn Agent playbooks catalog for titles and routing hints.\n• alpha: When alpha\n• beta: When beta",
  "input_schema" : {
    "type" : "object",
    "properties" : {
      "playbook_id" : {
        "type" : "string",
        "description" : "Registered playbook id. Loaded ids: alpha, beta."
      },
      "params" : {
        "type" : "object",
        "description" : "Inputs for the selected playbook_id per that playbook's inputSchema; invalid params fail at start_playbook.",
        "additionalProperties" : true
      }
    },
    "required" : [ "playbook_id", "params" ]
  }
}
```

Per-playbook `inputSchema` validation happens at execution time; the advertised schema uses a generic `params` object only.

## Repository Extended Tool Payloads

Canonical manifest for the **current** utilization training bundle: **`dev_data/scpa_utilization/tools/extended_tools.json`** (**4** tools, post-LLM-friendly / Day 4). Reconstruct per-tool wire objects at deploy time (sizes are not fixed in this doc).

### Current utilization tools (4 — post-LLM-friendly)

| Tool | Target service | Role |
| --- | --- | --- |
| `list_utilization_machines` | `SCPA_Utilization_helper.ListUtilizationMachines` | Machine coverage before records/summaries |
| `get_utilization_records` | `GetUtilizationRecords` | Raw event rows (optional single-machine scope) |
| `get_utilization_state_summary` | `GetUtilizationStateSummary` | Aggregate / percent-by-state questions |
| `get_utilization_overview` | `GetUtilizationOverview` | Combined coverage + state summary evidence |

Stage contracts: **`training-stage-configuration-contracts.md`**. Instructor mirror: **`ParlerGuidance/workshop/day4/tools/extended_tools.json`**.

### Historical utilization tools (7 — pre-LLM-friendly training)

**Historical evidence only** — these names powered population **(d)** / the 2026-06-28 incident census. Early workshop stages used the seven-tool surface before the LLM-friendly consolidation to the four-tool manifest above. Payload sections below are **frozen incident snapshots**, not the current **`dev_data/scpa_utilization`** SoT.

### utilization_records

Size chars: `1386`

```json
{
  "name": "utilization_records",
  "description": "Use when the user asks for raw utilization event records for all utilization-capable machines over a time range. Provide StartDate and EndDate as ISO-8601 datetimes; ShiftID is optional when the user scopes by shift. Title: Utilization records Natural-time fields **calendarPhrase** (today/yesterday/tomorrow) and **relativeDuration** (e.g. 30m, 24h) are available as alternatives to StartDate/EndDate; set at most one and do not combine with the explicit pair (same contract as query_alert_history / query_numeric_property_history).",
  "input_schema": {
    "type": "object",
    "properties": {
      "StartDate": {
        "type": "string"
      },
      "ShiftID": {
        "type": "string"
      },
      "EndDate": {
        "type": "string"
      },
      "calendarPhrase": {
        "type": "string",
        "description": "Optional natural-time alternative to StartDate/EndDate: **today** / **yesterday** / **tomorrow** (single day, host user_timezone IANA). Mutually exclusive with StartDate/EndDate and relativeDuration. Resolved to the closed-open day window per docs/agent/time-interpretation.md §4.2."
      },
      "relativeDuration": {
        "type": "string",
        "description": "Optional natural-time alternative to StartDate/EndDate: closed-open duration ending **now** (e.g. **30m**, **24h**, **7d**). Mutually exclusive with StartDate/EndDate and calendarPhrase. Same grammar as built-in tools — see docs/agent/time-interpretation.md §5."
      }
    },
    "required": [
      "ShiftID"
    ]
  }
}
```

### utilization_records_by_machine

Size chars: `1537`

```json
{
  "name": "utilization_records_by_machine",
  "description": "Use when the user asks for raw utilization event records for one specific machine over a time range. **Machine** must be the canonical ThingWorx Thing name of the machine (not a display name, short equipment label, alias, or serial). Provide StartDate and EndDate as ISO-8601 datetimes; ShiftID is optional. Title: Utilization records by machine Natural-time fields **calendarPhrase** (today/yesterday/tomorrow) and **relativeDuration** (e.g. 30m, 24h) are available as alternatives to StartDate/EndDate; set at most one and do not combine with the explicit pair (same contract as query_alert_history / query_numeric_property_history).",
  "input_schema": {
    "type": "object",
    "properties": {
      "StartDate": {
        "type": "string"
      },
      "ShiftID": {
        "type": "string"
      },
      "EndDate": {
        "type": "string"
      },
      "Machine": {
        "type": "string"
      },
      "calendarPhrase": {
        "type": "string",
        "description": "Optional natural-time alternative to StartDate/EndDate: **today** / **yesterday** / **tomorrow** (single day, host user_timezone IANA). Mutually exclusive with StartDate/EndDate and relativeDuration. Resolved to the closed-open day window per docs/agent/time-interpretation.md §4.2."
      },
      "relativeDuration": {
        "type": "string",
        "description": "Optional natural-time alternative to StartDate/EndDate: closed-open duration ending **now** (e.g. **30m**, **24h**, **7d**). Mutually exclusive with StartDate/EndDate and calendarPhrase. Same grammar as built-in tools — see docs/agent/time-interpretation.md §5."
      }
    },
    "required": [
      "ShiftID",
      "Machine"
    ]
  }
}
```

### utilization_aggregate_by_state

Size chars: `1170`

```json
{
  "name": "utilization_aggregate_by_state",
  "description": "Use after retrieving utilization records when the user asks for utilization grouped by state, total duration, counts, averages, min/max duration, or duration percentage. Title: Utilization aggregate by state",
  "input_schema": {
    "type": "object",
    "properties": {
      "UtilizationRecords": {
        "type": "array",
        "description": " Each array element is one table row. Column keys and types below.",
        "items": {
          "type": "object",
          "properties": {
            "Comment": {
              "type": "string"
            },
            "EquipmentDesc": {
              "type": "string"
            },
            "EquipmentID": {
              "type": "string"
            },
            "ShiftID": {
              "type": "string"
            },
            "Duration": {
              "type": "number"
            },
            "ProductID": {
              "type": "string"
            },
            "OperatorID": {
              "type": "string"
            },
            "ModifiedBy": {
              "type": "string"
            },
            "ReasonGroup": {
              "type": "string"
            },
            "Reason": {
              "type": "string"
            },
            "ModifiedAt": {
              "type": "string"
            },
            "DurationString": {
              "type": "string"
            },
            "UtilizationState": {
              "type": "string"
            },
            "id": {
              "type": "string"
            },
            "EventStart": {
              "type": "string"
            }
          },
          "required": [
            "Comment",
            "EquipmentDesc",
            "EquipmentID",
            "ShiftID",
            "Duration",
            "ProductID",
            "OperatorID",
            "ModifiedBy",
            "ReasonGroup",
            "Reason",
            "ModifiedAt",
            "DurationString",
            "UtilizationState",
            "id",
            "EventStart"
          ]
        }
      }
    },
    "required": [
      "UtilizationRecords"
    ]
  }
}
```

### utilization_stats_for_aggregate

Size chars: `1037`

```json
{
  "name": "utilization_stats_for_aggregate",
  "description": "Use after aggregating utilization records by state when the user asks for overall utilization percent, uptime, downtime, total time, event count, or average event duration. Title: Utilization stats for aggregate data",
  "input_schema": {
    "type": "object",
    "properties": {
      "AggregatedByUtilizationStateData": {
        "type": "array",
        "description": " Each array element is one table row. Column keys and types below.",
        "items": {
          "type": "object",
          "properties": {
            "AVERAGE_Duration": {
              "type": "number"
            },
            "Percentage": {
              "type": "number"
            },
            "COUNT_Duration": {
              "type": "integer"
            },
            "SUM_Duration_Hours": {
              "type": "number"
            },
            "UtilizationState": {
              "type": "string"
            },
            "SUM_Duration": {
              "type": "number"
            },
            "SUM_Duration_Minutes": {
              "type": "number"
            },
            "MIN_Duration": {
              "type": "number"
            },
            "MAX_Duration": {
              "type": "number"
            }
          },
          "required": [
            "AVERAGE_Duration",
            "Percentage",
            "COUNT_Duration",
            "SUM_Duration_Hours",
            "UtilizationState",
            "SUM_Duration",
            "SUM_Duration_Minutes",
            "MIN_Duration",
            "MAX_Duration"
          ]
        }
      }
    },
    "required": [
      "AggregatedByUtilizationStateData"
    ]
  }
}
```

### utilization_machine_listing

Size chars: `1082`

```json
{
  "name": "utilization_machine_listing",
  "description": "Use when the user asks which machines are available for utilization reporting or needs a machine list before querying utilization records. Pass Machines only when the user already provided a candidate machine list; UsesSelection controls whether to use that selection. Title: Utilization machine listing",
  "input_schema": {
    "type": "object",
    "properties": {
      "UsesSelection": {
        "type": "boolean"
      },
      "Machines": {
        "type": "array",
        "description": " Each array element is one table row. Column keys and types below.",
        "items": {
          "type": "object",
          "properties": {
            "isSystemObject": {
              "type": "boolean",
              "description": "Indicates if a system object or not"
            },
            "name": {
              "type": "string",
              "description": "Entity name"
            },
            "description": {
              "type": "string",
              "description": "Entity description"
            },
            "homeMashup": {
              "type": "string",
              "description": "Home mashup"
            },
            "avatar": {
              "type": "string",
              "description": "Avatar image"
            },
            "tags": {
              "type": "string",
              "description": "Tags"
            }
          },
          "required": [
            "isSystemObject",
            "name",
            "description",
            "homeMashup",
            "avatar",
            "tags"
          ]
        }
      }
    },
    "required": [
      "UsesSelection",
      "Machines"
    ]
  }
}
```

### utilization_machine_listing_with_dates

Size chars: `2087`

```json
{
  "name": "utilization_machine_listing_with_dates",
  "description": "Use when the user asks for utilization-capable machines with effective start and end dates for a specific time range. Provide StartDate and EndDate as ISO-8601 datetimes; ShiftID is optional; pass Machines only when the user already supplied a candidate machine list. Title: Utilization machine listing with dates Natural-time fields **calendarPhrase** (today/yesterday/tomorrow) and **relativeDuration** (e.g. 30m, 24h) are available as alternatives to StartDate/EndDate; set at most one and do not combine with the explicit pair (same contract as query_alert_history / query_numeric_property_history).",
  "input_schema": {
    "type": "object",
    "properties": {
      "StartDate": {
        "type": "string"
      },
      "ShiftID": {
        "type": "string"
      },
      "EndDate": {
        "type": "string"
      },
      "Machines": {
        "type": "array",
        "description": " Each array element is one table row. Column keys and types below.",
        "items": {
          "type": "object",
          "properties": {
            "isSystemObject": {
              "type": "boolean",
              "description": "Indicates if a system object or not"
            },
            "name": {
              "type": "string",
              "description": "Entity name"
            },
            "description": {
              "type": "string",
              "description": "Entity description"
            },
            "homeMashup": {
              "type": "string",
              "description": "Home mashup"
            },
            "avatar": {
              "type": "string",
              "description": "Avatar image"
            },
            "tags": {
              "type": "string",
              "description": "Tags"
            }
          },
          "required": [
            "isSystemObject",
            "name",
            "description",
            "homeMashup",
            "avatar",
            "tags"
          ]
        }
      },
      "calendarPhrase": {
        "type": "string",
        "description": "Optional natural-time alternative to StartDate/EndDate: **today** / **yesterday** / **tomorrow** (single day, host user_timezone IANA). Mutually exclusive with StartDate/EndDate and relativeDuration. Resolved to the closed-open day window per docs/agent/time-interpretation.md §4.2."
      },
      "relativeDuration": {
        "type": "string",
        "description": "Optional natural-time alternative to StartDate/EndDate: closed-open duration ending **now** (e.g. **30m**, **24h**, **7d**). Mutually exclusive with StartDate/EndDate and calendarPhrase. Same grammar as built-in tools — see docs/agent/time-interpretation.md §5."
      }
    },
    "required": [
      "ShiftID",
      "Machines"
    ]
  }
}
```

### utilization_aggregate_by_state_time_fence

Size chars: `1438`

```json
{
  "name": "utilization_aggregate_by_state_time_fence",
  "description": "Use when the user asks for an overview of utilization aggregated by state across machines over a time range without first needing raw event rows. Provide StartDate and EndDate as ISO-8601 datetimes; ShiftID is optional. Title: Utilization aggregate by state over time range Natural-time fields **calendarPhrase** (today/yesterday/tomorrow) and **relativeDuration** (e.g. 30m, 24h) are available as alternatives to StartDate/EndDate; set at most one and do not combine with the explicit pair (same contract as query_alert_history / query_numeric_property_history).",
  "input_schema": {
    "type": "object",
    "properties": {
      "StartDate": {
        "type": "string"
      },
      "ShiftID": {
        "type": "string"
      },
      "EndDate": {
        "type": "string"
      },
      "calendarPhrase": {
        "type": "string",
        "description": "Optional natural-time alternative to StartDate/EndDate: **today** / **yesterday** / **tomorrow** (single day, host user_timezone IANA). Mutually exclusive with StartDate/EndDate and relativeDuration. Resolved to the closed-open day window per docs/agent/time-interpretation.md §4.2."
      },
      "relativeDuration": {
        "type": "string",
        "description": "Optional natural-time alternative to StartDate/EndDate: closed-open duration ending **now** (e.g. **30m**, **24h**, **7d**). Mutually exclusive with StartDate/EndDate and calendarPhrase. Same grammar as built-in tools — see docs/agent/time-interpretation.md §5."
      }
    },
    "required": [
      "ShiftID"
    ]
  }
}
```

---

## Addendum — `load_tool_schemas` meta-tool (tool-schema-admission-control M3)

The `lazy` tool-admission mode adds one **meta-tool**, `load_tool_schemas(names[])`, which is
**not** part of the population **(d)** census above and is **only advertised in `lazy` mode** (registered
executor-only, so it never appears under `off`/`narrow`). In `lazy`, the first request advertises
only the core set plus this meta-tool, whose description carries a size-capped catalog
(`name: whenToUse`) of the deferred tail; the model calls it to load the full schemas it needs,
which are then advertised natively on subsequent rounds. See
`docs/operations/tool-schema-admission-control.md` §2.8.

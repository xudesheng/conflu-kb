# Chart contract — wire frame + `ChartBlock`

**Normative:** Single source for (1) the **server → client** **`type: "chart"`** wire object (including ThingWorx AlwaysOn), and (2) the nested **`chart`** payload **`ChartBlock`**.

**Related:** [`API_CONTRACT.md`](./API_CONTRACT.md), [`UI_CLIENT_PROTOCOL.md`](./UI_CLIENT_PROTOCOL.md) §2–4, [`TABLE_CONTRACT.md`](./TABLE_CONTRACT.md) (structured **`type: "table"`** wire), [`agent-alwayson.md`](../docs/architecture/agent-alwayson.md) §6 (`ReceiveMessage`), reference client [`parler-ui/lib/wireAdapter.js`](../parler-ui/lib/wireAdapter.js), renderer [`parler-ui/components/chart-draw.js`](../parler-ui/components/chart-draw.js).

**Emitter:** Production **`ChartBlock`** payloads are built on the **ThingWorx** side (e.g. **`parler-agent`** / agent tools), not in this repo.

**Extension backlog (non-normative):** [`chart-extensions-roadmap.md`](../docs/ui/chart-extensions-roadmap.md).

---

## 1. Purpose

Charts are **server-authored** structured graphics: coordinates are **not** taken from free-form model prose alone. The agent stack may call tools that return or build **`ChartBlock`** from ThingWorx data; the **client** (`<parler-ui>`) only renders validated wire JSON.

---

## 2. Wire envelope (`type: "chart"`)

Each chart is **one** JSON object on the server→client stream: one **`ReceiveMessage`** payload (default profile), or **one element** of a JSON array batch.

### 2.1 Required fields (all transports)

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | **Literal** `"chart"`. |
| `request_id` | string | Same turn id as `session.ack`, `activity`, `content.delta`, `done` for this assistant reply. |
| `chart` | object | **`ChartBlock`** — schema in **§3**. |

### 2.2 AlwaysOn / `ReceiveMessage` extension

Per **[`agent-alwayson.md`](../docs/architecture/agent-alwayson.md)** §6.2, every object sent through **`ReceiveMessage`** **MUST** also include:

| Field | Type | Description |
|-------|------|-------------|
| `conversation_id` | string | Non-empty; Conversation / thread id (snake_case). |

### 2.3 Example (AlwaysOn)

```json
{
  "type": "chart",
  "conversation_id": "agent-thread01",
  "request_id": "8fecb285-877c-4304-9d0c-0ce837830fc3",
  "chart": {
    "kind": "line",
    "chartId": "c1",
    "title": "Pressure (SteamSensor)",
    "x_label": "Time",
    "y_label": "Pressure",
    "series": [
      {
        "name": "Pressure",
        "x": ["2026-03-30T03:20:48.351Z", "2026-03-30T03:20:53.448Z"],
        "y": [18.61, 19.4]
      }
    ],
    "requested_time_range": {
      "start": "2026-03-30T03:15:00.000Z",
      "end": "2026-03-30T03:45:00.000Z"
    },
    "y_reference_lines": [{ "y": 25, "role": "ucl", "label": "UCL" }]
  }
}
```

Dedicated Parler WebSocket (non-AlwaysOn) may omit **`conversation_id`** on **`chart`** when the connection implies a single thread; see [`API_CONTRACT.md`](./API_CONTRACT.md).

### 2.4 Multiple `type: "chart"` frames per assistant turn

A single assistant reply sequence (**same `request_id`**) **MAY** include **more than one** distinct **`type: "chart"`** wire object (each its own payload in the stream). Emitters **SHOULD** assign distinct **`chart.chartId`** values (`c1`, `c2`, …) when emitting multiple charts so prose and exports can refer to them without positional wording.

### 2.5 Optional `cacheId` on tabular tool success envelopes

Qualifying **tabular** tool **success** JSON envelopes (including agent extended-tool **`INFOTABLE`** / **`INFOTABLE_LARGE`**-shaped service results, and built-in **`tabulate_cached_result`** aggregate/transform results such as **`CACHED_GROUP_METRIC_INLINE`**) **MAY** include an optional top-level string field **`cacheId`**: the opaque conversation-cache identifier under which the agent stored **that tool’s chartable table** for paging / charting (`fetch_cached_result`, `build_chart_from_tabular_result` with `source: "cache_id"`). For **`tabulate_cached_result`** transforms that produce a new derived **`InfoTable`**, **`cacheId`** refers to the **transformed** output table; the envelope **MAY** also carry **`sourceCacheId`** for the input table id. Raw **`analyze_entity_set`** success JSON also carries a **`cacheId`**, but **`build_chart_from_tabular_result`** **SHOULD** be driven from a **`tabulate_cached_result`** (or **`fetch_cached_result`**) envelope on that id so chart construction reuses the same tabular semantics as other entity-list charts (**`CONTRACTS/ENTITY_SET_TOOL.md`**). **`cacheId` is not derivable from tool call ids**; it is allocated when the table is stored. Clients and history hydrators **MUST** tolerate unknown additive top-level fields on tool-result JSON.

---

## 3. `ChartBlock` — JSON schema (logical)

The **`chart`** property **MUST** satisfy the following. Any server (ThingWorx agent tools, etc.) that emits charts on the Parler wire **MUST** use this shape wherever a nested chart object appears.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `kind` | string | yes | `line` \| `bar` \| `scatter` \| `pie` |
| `chartId` | string | no | Optional stable id within one assistant run (e.g. `c1`, `c2`). Emitters **SHOULD** assign this when emitting new charts so final prose and export can refer to a chart without positional wording. Clients **MUST** accept charts that omit `chartId` (history and older servers). |
| `title` | string | no | Short chart title |
| `x_label` | string | no | X axis caption (e.g. `Time`). Wire `x[]` / `requested_time_range` use **ISO instants** (UTC `…Z` recommended on the wire); the reference client **formats axis ticks in browser-local wall time** per [`times-solution.md`](../docs/architecture/times-solution.md). |
| `y_label` | string | no | Y axis label |
| `series` | array | yes | One or more series (many servers emit one) |
| `y_reference_lines` | array | no | Horizontal lines on the **Y** axis (limits, SPC). Cap at **12** entries. |
| `requested_time_range` | object | no | **Time `line` / `scatter` only:** ISO-8601 UTC bounds for the **X** axis domain (the query / analysis window). When present and valid, the reference client **prefers** this domain over **data extent** so sparse windows show as **blank** gaps (no fabricated points). Ignored for **`bar`** and for numeric (non-datetime) X. **Omit** when `xAxisMode` is `"elapsed"` or `"normalized"`. |
| `xAxisMode` | string | no | **`line` / `scatter` only:** `"absolute"` (default when absent), `"elapsed"` (history overlay elapsed mode), or `"normalized"` (history overlay normalized 0..1 shape comparison). When `"elapsed"`, numeric X is **elapsed seconds** from each series' window start; **`requested_time_range`** MUST be omitted. When `"normalized"`, numeric X is **fraction of window** in `[0, 1]`; **`requested_time_range`** and **`elapsedDomain`** MUST be omitted. |
| `elapsedDomain` | object | yes when `xAxisMode === "elapsed"` | `{ "start": number, "end": number }` — fixed numeric X domain in **seconds** for the full comparison window (typically `{ "start": 0, "end": <durationSec> }`). |
| `normalizedDomain` | object | yes when `xAxisMode === "normalized"` | `{ "start": number, "end": number }` — fixed numeric X domain for normalized shape comparison; **MUST** be `{ "start": 0, "end": 1 }`. |
| `source` | object | no | **Provenance** for server-built charts. Emitters **SHOULD** populate this for new charts (see **§3.0a**). Clients **MUST** tolerate absence (history / third-party). Unknown sub-fields **MAY** be ignored. |

### 3.0 Chart limits (named constants, v1 tabular chart builder)

The **reference** Parler tabular chart builder (`build_chart_from_tabular_result`) and UI validation **MUST** enforce the following limits unless a future contract revision changes them:

| Constant | Value | Meaning |
|----------|------:|---------|
| `PIE_DEFAULT_MAX_SLICES` | 8 | Default maximum **non-zero** pie slices before `top_with_other` merges tail into **`Other`**. |
| `PIE_HARD_MAX_SLICES` | 12 | Hard cap on **non-zero** slices when `pieSliceMode` is `all_nonzero` (user opt-in “show all”). |
| `BAR_MAX_SERIES` | 6 | Maximum **series** count for grouped **`bar`**. |
| `BAR_MAX_CATEGORIES` | 24 | Maximum **category** count (shared `series[0].x` length) for **`bar`**. |

### 3.0a `ChartBlock.source` (optional object)

When present, `source` **SHOULD** be a JSON object. For charts produced by **`build_chart_from_tabular_result`** in v1, the server **MUST** include `source` with at least the producer-required fields below.

| Field | Type | Required for v1 `build_chart_from_tabular_result` producer | Description |
|-------|------|:--:|-------------|
| `sourceResolved` | string | **yes** | How the tabular input was resolved (e.g. `cache_id`, `last_invoke`). |
| `sourceCacheId` | string | no | Tabular cache id when available. |
| `sourceToolCallId` | string | no | Originating tool call id when runtime exposes it. |
| `sourceResultKind` | string | no | e.g. `CACHED_TABULATE_GROUP_METRIC`. |
| `sourceColumns` | string[] | **yes** | Columns used for axes / series mapping. |
| `transformSummary` | string | no | Short deterministic transform description. |
| `rowCount` | number | **yes** | Input tabular row count used to build the chart. |
| `pointCount` | number | **yes** | Slice count or point count emitted in `series`. |
| `truncationApplied` | boolean | **yes** | `true` if top-N / **`Other`** / slice merge was applied. |
| `filledMissingCombinations` | number | no | Grouped **`bar`**: count of synthetic zero-filled `(category, series)` pairs. |
| `zeroValueCategoryCount` | number | no | **`pie`**: count of categories with numeric **zero** excluded before slice-cap / **`Other`** (audit). |
| `missingPeriods` | array | no | **History overlay elapsed mode (`build_history_overlay_chart`):** series with zero samples that were requested but not drawn; each item SHOULD include `label`, `thingName`, `propertyName`, `start`, `end`. |

### 3.0e History overlay (`build_history_overlay_chart`)

Producer: **`build_history_overlay_chart`** (`docs/agent/history-overlay-chart.md`). Replaces the retired model-facing **`build_period_over_period_chart`** and **`build_multi_series_history_chart`** tools.

**Absolute-time mode** (tool `xAxisMode`: **`absolute_time`**, or omitted when all series share the same resolved window):

| Rule | Requirement |
|------|-------------|
| `kind` | **`line`** or **`scatter`** |
| `xAxisMode` | **MUST NOT** be present (absolute ISO time X) |
| `requested_time_range` | **SHOULD** be present when all series share the same resolved window |
| `series[].x[]` | ISO-8601 UTC timestamps from platform history |
| `series[].sourceWindow` | **MUST NOT** be present |
| Missing series | Partial chart allowed; **`source.missingSeries[]`** lists zero-sample series; hard fail when all series empty |

**Elapsed-time mode** (tool `xAxisMode`: **`elapsed_time`**, or server default when windows differ):

| Rule | Requirement |
|------|-------------|
| `kind` | **`line`** or **`scatter`** |
| `xAxisMode` | **`"elapsed"`** |
| `elapsedDomain` | **Required**; `end` = **max** resolved series duration in seconds (series may differ in duration; shorter series stop earlier) |
| `requested_time_range` | **MUST NOT** be present |
| `series[].x[]` | Non-negative integer **seconds** elapsed from that series' resolved window **start** (stringified) |
| `series[].sourceWindow` | **Required** per series with data: `start`, `end` (ISO UTC), `thingName`, `propertyName`, `periodLabel`, optional `resolvedTimeZone`, `sampleCount` (emitted), optional `rawSampleCount`, optional `omittedSampleCount` |
| Mixed Things / durations | **Allowed** on one elapsed chart (cross-Thing same shifted window, unequal durations) |
| Missing series | Partial chart allowed; **`source.missingPeriods[]`** lists zero-sample elapsed series |

**Normalized-time mode** (tool `xAxisMode`: **`normalized_time`** — explicit model input only):

| Rule | Requirement |
|------|-------------|
| `kind` | **`line`** or **`scatter`** |
| `xAxisMode` | **`"normalized"`** |
| `normalizedDomain` | **Required**; **MUST** be `{ "start": 0, "end": 1 }` |
| `elapsedDomain` | **MUST NOT** be present |
| `requested_time_range` | **MUST NOT** be present |
| `series[].x[]` | Numeric **fraction of window** in **`[0, 1]`** (stringified); producer **SHOULD** clamp to `[0, 1]` |
| `series[].sourceWindow` | **Required** per series with data (same fields as elapsed mode) |
| Mixed Things / durations | **Allowed** — intended for different-duration shape comparison |
| Missing series | Partial chart allowed; **`source.missingPeriods[]`** lists zero-sample series |
| Zero-duration window | Producer **MUST NOT** emit; tool fails **`HISTORY_OVERLAY_INVALID_TIME_WINDOW`** |

**Shared (all modes):**

| Rule | Requirement |
|------|-------------|
| Series count | **2..6** series, same top-level `propertyName` |
| Point budget | **`HISTORY_OVERLAY_MAX_TOTAL_EMITTED_POINTS = 5000`**; fair per-series uniform subsample when exceeded; `source.truncationApplied=true` |
| `y_reference_lines` | **MAY** be present (cap **12**); tool input `yReferenceLines` |
| `source.sourceResolved` | **`history_overlay`** |

Reference client (`chart-draw.js`): elapsed mode uses **`elapsedDomain`** and **`m:ss`** ticks; normalized mode uses **`normalizedDomain`** and **percent** ticks; absolute mode uses ISO time formatting per [`times-solution.md`](../docs/architecture/times-solution.md).

**Historical note:** §3.0e/§3.0f in releases before the history-overlay-chart topic named separate PoP and multi-series producers; those model-facing tools are retired.

### 3.0b `kind: "pie"` — payload and REJECT rules

**Payload:** Reuses **`series[]`** with **exactly one** series: `series[0].x[]` are slice labels (categories), `series[0].y[]` are non-negative numeric **values** (same length as `x`). `series[0].name` is the legend group label (e.g. the measured dimension). `requested_time_range` **MUST** be ignored for **`pie`**.

**REJECT** (server builder **MUST NOT** emit wire `ChartBlock`; client adapter **MUST** drop invalid blocks):

| Code | Condition |
|------|-----------|
| `PIE_REQUIRES_SINGLE_SERIES` | `series.length != 1`. |
| `ROW_ALIGN_FAILED` | `len(series[0].x) != len(series[0].y)`. |
| `Y_COLUMN_NOT_NUMERIC` | Any `y` not a finite number. |
| `PIE_NEGATIVE_VALUE` | Any `y < 0`. |
| `PIE_ZERO_TOTAL` | Sum of all `y` is zero (no drawable pie). |
| `TOO_MANY_SLICES` | Non-zero slice count exceeds `PIE_HARD_MAX_SLICES` when `pieSliceMode` is `all_nonzero` (no silent merge). |
| `DUPLICATE_SLICE_LABEL` | Two or more **non-zero** rows share the same `xColumn` category string (no silent merge). |

Slice policy parameters (`pieSliceMode`, `pieMaxSlices`) are tool arguments on **`build_chart_from_tabular_result`**, not fields on **`ChartBlock`**.

**Grouped bar (long table, `seriesColumn` set):** duplicate `(xColumn, seriesColumn)` row pairs **MUST** fail with **`DUPLICATE_SERIES_CATEGORY`** even when `y` values are identical (no silent dedupe).

**Line/scatter long pivot (`seriesColumn` set, `kind` `line` or `scatter`):** pivot long-format rows into multiple `series[]` entries (one per distinct `seriesColumn` value). Series order **MUST** follow **first-seen** row order in the source table. Duplicate `(xColumn, seriesColumn)` row pairs **MUST** fail with **`DUPLICATE_SERIES_CATEGORY`**. Missing `(xColumn, seriesColumn)` combinations **MUST NOT** be zero-filled (unlike grouped bar): each series' `x[]`/`y[]` contain only rows present for that series — line/scatter pivot stays **sparse** (no bar-style zero-fill grid). The reference client connects available points per series with straight segments and point markers for **`line`**, or markers only for **`scatter`**; interior gaps are **not** broken segments — the client draws one continuous polyline (or marker path) through each series' available points without synthesizing zero-fill or visible segment breaks for missing `(xColumn, seriesColumn)` pairs. At most **`MAX_SERIES = 6`** distinct series names; excess **MUST** fail with **`TOO_MANY_SERIES`**. `source.transformSummary` **SHOULD** be `long_table_pivot(seriesColumn)`; `source.filledMissingCombinations` applies to **bar** long pivot only.

**Zero-value categories:** Before applying `pieMaxSlices` / top-N / **`Other`**, categories with **`y === 0`** **MUST NOT** consume slice slots, **MUST NOT** be merged into **`Other`**, and **MUST NOT** be drawn as arcs. The count of such categories **SHOULD** appear in `source.zeroValueCategoryCount` when `source` is present.

### 3.0c `requested_time_range` (optional)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `start` | string | yes | Inclusive interval start, **ISO 8601** instant (UTC `…Z` recommended). |
| `end` | string | yes | Inclusive interval end, same format. |

**Semantics**

- Emitters **SHOULD** set this when the underlying query used an explicit time window (e.g. **`query_property_history`** on a numeric property with `startTime`/`endTime`), so the chart X axis matches **user intent** even when samples exist only in a sub-interval.
- **Clients MUST NOT** synthesize data points to fill gaps inside this window.
- If `requested_time_range` is **absent** or **invalid**, time **`line`/`scatter`** fall back to **data extent** for X (unchanged legacy behavior).

### 3.0d `build_chart_from_tabular_result` tool success JSON (v1, agent → LLM)

This is **not** a `type: "chart"` wire frame; it is the structured tool result string. For v1 reference **`parler-agent`**, **`CHART_EMITTED`** success payloads **SHOULD** mirror **`ChartBlock.source`** so the model can cite provenance and truncation honestly:

- Nested **`source`**: same field set as **`ChartBlock.source`** (including **`sourceResolved`**, optional **`sourceCacheId`**, **`sourceColumns`**, **`rowCount`**, **`pointCount`**, **`truncationApplied`**, optional **`filledMissingCombinations`**, **`zeroValueCategoryCount`**, **`transformSummary`**).
- Top-level convenience mirrors: at minimum **`sourceResolved`**, **`sourceColumns`**, **`rowCount`**, **`pointCount`**, **`truncationApplied`**, and legacy **`truncated`** (same boolean as **`truncationApplied`**).
- **`chartBlock`** (object, optional on persisted Stream tool rows): full **`ChartBlock`** payload as emitted on the live **`type: "chart"`** wire (minus transport envelope). **`AgentMessageStreamHistoryExporter`** hydrates **`ai-parler-history-v1`** assistant **`charts[]`** from this field when present; legacy rows without **`chartBlock`** are skipped for tabular charts (numeric-history tool results remain reconstructable from **`points[]`**).

### 3.1 `y_reference_lines[]` item

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `y` | number | yes | Value on the same scale as `series[].y`. |
| `label` | string | no | Short label (e.g. `435`, `USL`). |
| `role` | string | no | `usl` \| `ucl` \| `lcl` \| `lsl` \| `target` \| `limit` \| `warning`. **Spec:** `usl` / `lsl`. **Control:** `ucl` / `lcl`. **Center:** `target`. **Generic ceiling/floor:** `limit`. **Advisory:** `warning`. Default styling: `limit`. |

Reference lines extend the Y domain when needed so limits stay visible if all samples are on one side.

### 3.2 `series[]` item

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Legend label |
| `x` | string[] | yes | X values; absolute time series: **ISO 8601 UTC** (`...Z`); elapsed history overlay: **non-negative integer seconds** (stringified); normalized history overlay: **numeric fraction in [0, 1]** (stringified). |
| `y` | number[] | yes | Y values; **same length** as `x`. |
| `sourceWindow` | object | no | **Elapsed history overlay series:** resolved window metadata (see **§3.0e**). Clients **MAY** ignore. |

### 3.3 Invariants

1. For every series, `len(x) == len(y)`.
2. `x` order matches the source (e.g. ascending time for history).
3. Only **finite numeric** scalars in **`y`** (Thingworx: `NUMBER`, `INTEGER`, `LONG`, `BOOLEAN` as 0/1). **Emitters** MUST NOT emit a **`ChartBlock`** when any mapped `y` is missing, null, non-numeric, or non-finite: fail the tool/build step with structured errors (`Y_COLUMN_NOT_NUMERIC`, `ROW_ALIGN_FAILED`, …) per [`flexible-chart-solution.md`](../docs/architecture/flexible-chart-solution.md) §§8–9 — **no** row skipping or silent dropout in the server chart builders. **Clients** (`chart-draw.js`) MUST NOT render if any `series[].y` value is non-finite after numeric coercion (defense in depth).
4. No downsampling in the **history read** path for lab-scale volumes; chart uses **all** numeric points in the window unless a future contract adds optional downsampling.
5. For **`line`** and **`scatter`**, every `x[]` value MUST be usable as **one consistent domain** for the whole chart: either all **finite numeric** (after string trim) or all **ISO-8601–parsable** instants (typical UTC `...Z`). **Categorical** X belongs in **`bar`**. Emitters MUST NOT rely on clients to coerce garbage strings to zero; [`flexible-chart-solution.md`](../docs/architecture/flexible-chart-solution.md) §9 aligns builder validation with the reference renderer.

### 3.4 Standalone `ChartBlock` example (e.g. REST block)

```json
{
  "kind": "line",
  "title": "line-1 — temperature",
  "x_label": "Time",
  "y_label": "Temperature",
  "series": [
    {
      "name": "line-1 / temperature",
      "x": ["2025-03-25T08:00:00Z", "2025-03-25T08:10:00Z"],
      "y": [41.2, 41.5]
    }
  ],
  "y_reference_lines": [{ "y": 435, "role": "limit", "label": "435" }]
}
```

---

## 4. Reference client (`parler-ui`): kinds, validation, ordering

### 4.1 Supported `kind` values

The reference UI accepts **`line`**, **`bar`**, **`scatter`**, and **`pie`** (`asChartBlock` in `wireAdapter.js`). New kinds require a **contract bump**.

| `kind` | Use | Renderer (`chart-draw.js`) |
|--------|-----|----------------------------|
| **`line`** | Time series, elapsed/normalized history overlay, or ordered numeric X | X = **time** if **every** parsed `x` across **all** `series[]` is a date, else **numeric** linear. **Elapsed history overlay:** when `xAxisMode === "elapsed"` and **`elapsedDomain`** is valid, X domain = **`elapsedDomain.start`/`end`** (seconds); ticks **`m:ss`**; **`requested_time_range`** ignored. **Normalized history overlay:** when `xAxisMode === "normalized"` and **`normalizedDomain`** is valid, X domain = **`normalizedDomain.start`/`end`**; ticks **percent** (`0%`…`100%`); **`requested_time_range`** and **`elapsedDomain`** ignored. **Absolute time X:** if **`requested_time_range`** is set and parses, its **`start`/`end`** define the **X domain**; otherwise **data extent**. **Axis tick labels** use **`d3.scaleTime`** (browser-local wall clock) for datetime X. Y domain spans **all** series' Y plus reference lines. One polyline + small circles per series; **legend** when multiple series. **Sparse** regions inside the fixed domain remain empty (no padding with fake points). |
| **`scatter`** | Same payload as line | Same scales as **`line`**; **no** line paths; larger markers per series; legend when multiple series. |
| **`bar`** | Categories / buckets | **Grouped** bars when multiple series (shared category index; labels from **`series[0].x`**). Single series: **band** on indices, heights from **`y`**. Axis ticks: **categorical** `x` strings shown as-is (very long strings ellipsized); **ISO-8601** values matching `YYYY-MM-DDThh…` use the **`HH:mm:ss`** portion for compact time-axis labels. **`requested_time_range`** is **ignored**. |
| **`pie`** | Composition / share of whole | **Exactly one** `series`; `x[]` = slice labels, `y[]` = non-negative values; D3 `pie`/`arc`; slice colors stable by label; native **`<title>`** on each slice arc (label, raw value, percent); 0-value slices draw no arc but **MAY** appear in legend when space allows. **`requested_time_range`** is **ignored**. |

**Y reference lines:** invalid entries **skipped**; valid subset kept (`asYReferenceLines`). Roles map to stroke styles in the reference theme.

### 4.2 Validation (wire → UI)

The adapter **drops** the frame (no `assistant.chart`) if:

- `chart.kind` is not `line` / `bar` / `scatter` / `pie`.
- `chart.series` is missing, not an array, or **empty**.
- Any `series[].y` value is **non-finite** after numeric coercion (`chart-draw.js`; aligns with §3.3 inv. 3).
- For **`pie`**: `series.length !== 1`, `x`/`y` length mismatch, any `y < 0`, or sum of `y` is **0** (defense in depth; see **§3.0b**).

Unknown **top-level** wire fields: **ignored**.

### 4.3 Ordering vs other frames

- **`chart`** may be sent **before**, **between**, or **after** **`content.delta`** for the same **`request_id`**.
- UI **MUST** attach charts to the **assistant row** for that **`request_id`** ([`UI_CLIENT_PROTOCOL.md`](./UI_CLIENT_PROTOCOL.md) §4).
- **`activity`** / **`content.delta`** do not replace **`chart`**; a turn may be text-only, chart-only, or both.

---

## 5. Who emits a `chart` frame? (LLM vs Agent)

| Layer | Role |
|-------|------|
| **Agent / orchestration** | **Owns** emission of **`type: "chart"`**. **SHOULD** emit when a **validated `ChartBlock`** exists (tool results, history, ThingWorx samples). **MUST NOT** use model free-text alone as the only source of coordinates. |
| **LLM** | May call tools or steer intent; may emit Markdown in **`content.delta`**. Does **not** invoke **`ReceiveMessage`** or append wire frames. |
| **Tools** | Return **structured** data; **agent code** maps to **`ChartBlock`** and sends the wire frame. |

**Recommended flow:** (1) Model selects *what* to visualize. (2) Server builds and validates **`ChartBlock`**. (3) Server emits **`{ "type": "chart", "request_id", "chart", … }`** (+ **`conversation_id`** on AlwaysOn).

**Anti-patterns:** Expecting wire JSON inside assistant text; relying on huge **`activity.message`** JSON without a **`chart`** frame (truncation risk).

---

## 6. UI rendering notes (portable)

- **Line:** connect `(x,y)` in order.
- **Bar:** one bar per index; `x` labels categorical or time strings.
- **Scatter:** points only (same payload shape as line).
- **Y reference lines:** full-width horizontal segments; style **MAY** vary by `role` (this repo: spec/limit red solid, control orange dashed, target green dashed, warning yellow dashed).

Frontends may use D3, Chart.js, ECharts, etc., if they honor this contract.

---

## 7. Reference client limitations (disclosure)

- **`drawChart`** renders **all** `series[]` entries that pass length checks (`len(x) == len(y)`). **`line` / `scatter`** use **union** Y across series; **X** for datetime charts uses **`requested_time_range`** when valid, else **union** data extent for X. Mixed date vs numeric X across series is unsupported — parsing must yield a consistent scale type.
- **Bar** with multiple series uses **grouped** bars; category count is the **minimum** length across series if lengths differ. X ticks follow the §4.1 **`bar`** row (categorical vs ISO datetime heuristic); non–ISO-like strings are not forced through a time-only slice.
- **Pie** uses **exactly one** series; arcs are drawn only for strictly positive **`y`** values (zero categories are skipped). Invalid pie payloads are **dropped** by `asChartBlock` (defense in depth; see **§4.2**).

These are **UI** behavior notes, not permission to shrink the **wire** schema.

---

## 8. Versioning

- **Every edit:** any commit that edits this normative document bumps **[`CONTRACT_VERSION.md`](./CONTRACT_VERSION.md)** in the same commit; there is no wording-only exception.
- **Additive:** new optional **`ChartBlock`** fields or new **`kind`** values also update the aligned implementation/tests (and **`API_CONTRACT.md`** if the wire `chart` frame changes) in that commit.
- **Breaking:** align the implementation/tests, **`API_CONTRACT.md`**, this file, and the bundle patch in the same commit.
- **AlwaysOn** top-level requirements on **`ReceiveMessage`** → [`agent-alwayson.md`](../docs/architecture/agent-alwayson.md).

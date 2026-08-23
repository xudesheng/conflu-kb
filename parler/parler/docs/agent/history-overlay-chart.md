# History overlay chart

**Status:** P1 + P2 **shipped** on **`main`** — extension **0.1.205**, widget **0.1.89**, contracts **0.1.139** (2026-07-03). User-run live smoke remains out-of-band hardening. Retired PoP / multi-series design docs archived @ **`docs/archived/2026-07-07T224124-consolidate-history-overlay/`**.
**Topic name:** `history-overlay-chart`
**Branch / worktree:** merged to **`main`** (2026-07-03).
**Topic kind:** tool-surface simplification + chart capability consolidation.
**Primary decision:** Replace the model-facing PoP / multi-series split with one history overlay
chart tool. The old model-facing tools are retired directly; there is no compatibility migration and
no executor-only fallback for the old names.

---

## 0. Reviewer orientation

This document is written for a reviewer or implementor who may not know the project history.

Parler is a ThingWorx-first AI agent. A user prompt arrives from a ThingWorx mashup, the Java agent
chooses LLM tools, tools query live ThingWorx Things / DataTables / ValueStreams / configuration
repositories, and the response may contain text plus structured chart/table artifacts. The UI
renders server-authored `ChartBlock` JSON.

Read these first:

| Path | Why it matters |
|---|---|
| `AGENTS.md` | Repo rules, build rules, review workflow expectations. |
| `docs/review-framework.md` | Required packet workflow when this draft is promoted to review. |
| `CONTRACTS/CHART_CONTRACT.md` | Normative chart wire shape: `series[]`, `xAxisMode`, `elapsedDomain`, `y_reference_lines`. |
| `docs/archived/2026-07-07T224124-consolidate-history-overlay/period-over-period.md` | **Historical** PoP design (retired model-facing tool). |
| `docs/archived/2026-07-07T224124-consolidate-history-overlay/multi-series.md` | **Historical** same-window multi-series design (retired model-facing tool). |
| `docs/agent/all-tools.md` | Tool-surface and context-cost background. |
| `docs/agent/time-interpretation.md` | Existing time parsing rules to reuse; do not create a new time parser. |

Key code areas:

| Area | Path |
|---|---|
| Built-in tool registration / schema | `parler-agent/src/main/java/com/thingworx/things/agent/tools/BuiltInTools.java` |
| Current PoP executor (retired) | `parler-agent/src/main/java/com/thingworx/things/agent/tools/BuildPeriodOverPeriodChartExecutor.java` |
| Current multi-series executor (retired) | `parler-agent/src/main/java/com/thingworx/things/agent/tools/BuildMultiSeriesHistoryChartExecutor.java` |
| PoP same-window redirect guard | `parler-agent/src/main/java/com/thingworx/things/agent/PeriodOverPeriodPopSupport.java` |
| History fetch helper | `parler-agent/src/main/java/com/thingworx/things/agent/HistorySeriesComposerSupport.java` |
| PoP chart builder | `parler-agent/src/main/java/com/thingworx/things/agent/PeriodOverPeriodChartBuilder.java` |
| Multi-series chart builder | `parler-agent/src/main/java/com/thingworx/things/agent/MultiSeriesHistoryChartBuilder.java` |
| Single-history tool | `parler-agent/src/main/java/com/thingworx/things/agent/tools/PropertyToolsExecutor.java` |
| Chart wire / reference-line cleaning | `parler-agent/src/main/java/com/thingworx/things/agent/ParlerChartWireSupport.java` |
| UI chart renderer | `parler-ui/components/chart-draw.js` |
| UI wire adapter validation | `parler-ui/lib/wireAdapter.js` |
| Playbook tool allowlist | `parler-agent/src/main/java/com/thingworx/things/agent/playbook/PlaybookToolAllowlist.java` |

Live evidence that triggered this topic:

- The runtime under test exposed 33 model-facing tools.
- Host context was accepted and carried the selected Thing and selected time window.
- The prompt asked for one chart with:
  - selected Thing, selected window;
  - selected Thing, same window last week;
  - another Thing, same window last week.
- Without explicit guidance, the model used three `query_property_history` calls and emitted three
  charts.
- With explicit "use period-over-period chart tool", the agent emitted one PoP chart and then one
  multi-series chart because the PoP tool rejected same-window cross-Thing periods.

The problem is not missing data. The problem is that one user concept is split across two adjacent
model-facing chart tools.

---

## 1. Problem statement

Users think in terms of one chart:

```text
Overlay these history traces so I can compare them.
```

The traces may differ by:

- Thing;
- time window;
- both Thing and time window.

Today Parler exposes two separate model-facing tools for this:

| Current tool | Current mental model | Failure |
|---|---|---|
| `build_period_over_period_chart` | Different windows, elapsed X. | Rejects same-window cross-Thing periods, which breaks mixed overlays. |
| `build_multi_series_history_chart` | Same window, multiple Things, absolute X. | Cannot naturally express shifted windows. |

That split teaches the model the wrong boundary. It then has to infer which tool owns mixed cases.
Recent live tests show that this boundary is not robust.

### 1.1 Current shipped behavior this topic replaces

The split is enforced in code, not only in routing prose:

| Mechanism | Location | Effect today |
|---|---|---|
| Same-window cross-Thing PoP redirect | `PeriodOverPeriodPopSupport.detectSameWindowCrossDevice` | Returns **`POP_MULTI_SERIES_UNAVAILABLE`** → model must call **`build_multi_series_history_chart`**. |
| PoP equal-duration guard | `PeriodOverPeriodChartBuilder.validateDurationMatch` | Fails **`POP_WINDOW_DURATION_MISMATCH`** when period durations differ. |
| PoP duplicate-window guard | `PeriodOverPeriodChartBuilder.validateNoDuplicateWindows` | Fails **`POP_DUPLICATE_PERIOD`** when two periods resolve to the same window (including same window on different Things). |
| Multi-series shared window | `MultiSeriesSharedWindowResolver` | All series share one top-level window; per-series calendar offsets are not expressible. |

The unified tool removes the redirect and the guards that reject valid mixed overlays. Internal builders may be refactored, but the model-facing surface is one tool.

---

## 2. Design decision

Introduce one model-facing tool:

```text
build_history_overlay_chart
```

Retire these two model-facing tools directly:

```text
build_period_over_period_chart
build_multi_series_history_chart
```

There is no migration layer:

- do not keep the old names as executor-only tools;
- do not keep compatibility wrappers for model calls;
- do not silently rewrite old tool names;
- existing playbooks or config that reference old names must be updated as part of this topic;
- old references should fail normal validation if any remain.

Implementation may reuse existing Java internals from PoP and multi-series. The retirement rule is
about the public/model-facing tool surface, not about whether code is physically copied or reused.

---

## 3. Core model

The unified tool reads several history series and overlays them in one chart.

The tool's central abstraction is:

```json
{
  "label": "Robot 01 - selected window",
  "thingName": "SE.CellFab.Model.Workunit.BOS-StackingRobot-01",
  "propertyName": "currentDraw",
  "startTime": "2026-07-02T04:00:39.177Z",
  "endTime": "2026-07-02T06:00:39.028Z"
}
```

All series in the first implementation use the same `propertyName`. That keeps unit validation,
axis labeling, and LLM routing simple. Multi-property overlays are a separate future topic.

The key chart choice is the X-axis mode:

| Tool `xAxisMode` | Contract output | Meaning | Replaces |
|---|---|---|---|
| `absolute_time` | absent or `"absolute"` | Keep real timestamps. Good for same-window cross-Thing comparison. | `build_multi_series_history_chart` |
| `elapsed_time` | `"elapsed"` | Convert each point to seconds from that series' window start. Good for shifted-window shape comparison. | `build_period_over_period_chart` |
| `normalized_time` | new contract value | Map each series window to the same normalized X domain. Good for different-duration shape comparison. | New |

PoP is therefore not a separate tool. It is history overlay with `xAxisMode = "elapsed_time"`.
Multi-series is not a separate tool. It is history overlay with `xAxisMode = "absolute_time"`.

---

## 4. Proposed tool schema

Tool name:

```text
build_history_overlay_chart
```

Example:

```json
{
  "propertyName": "currentDraw",
  "xAxisMode": "elapsed_time",
  "title": "Current Draw - selected window vs last week",
  "yLabel": "Current Draw",
  "yReferenceLines": [
    { "y": 8.0, "label": "Upper limit", "role": "usl" },
    { "y": 2.0, "label": "Lower limit", "role": "lsl" }
  ],
  "series": [
    {
      "label": "Robot 01 - selected window",
      "thingName": "SE.CellFab.Model.Workunit.BOS-StackingRobot-01",
      "startTime": "2026-07-02T04:00:39.177Z",
      "endTime": "2026-07-02T06:00:39.028Z"
    },
    {
      "label": "Robot 01 - last week",
      "thingName": "SE.CellFab.Model.Workunit.BOS-StackingRobot-01",
      "startTime": "2026-06-25T04:00:39.177Z",
      "endTime": "2026-06-25T06:00:39.028Z"
    },
    {
      "label": "Robot 02 - last week",
      "thingName": "SE.CellFab.Model.Workunit.BOS-StackingRobot-02",
      "startTime": "2026-06-25T04:00:39.177Z",
      "endTime": "2026-06-25T06:00:39.028Z"
    }
  ]
}
```

Fields:

| Field | Type | Required | Notes |
|---|---:|:--:|---|
| `propertyName` | string | yes | Same numeric property for every series. |
| `series` | array | yes | 2..6 series. |
| `series[].label` | string | yes | Legend label. |
| `series[].thingName` | string | yes | Canonical Thing name. Existing Thing-name preflight applies. |
| `xAxisMode` | enum | no | **`absolute_time`**, **`elapsed_time`**, or **`normalized_time`** (see §5). When omitted, server applies P1 two-branch default only (equivalent windows → absolute; else elapsed). **`normalized_time` requires explicit tool input** — no executor shape-intent inference (review-6). |
| `chart_kind` | enum | no | **`line`** (default) or **`scatter`** in P1 — parity with retired multi-series tool. |
| `title` | string | no | Chart title. |
| `yLabel` | string | no | Y-axis label. |
| `yReferenceLines` | array | no | Horizontal limit / target / warning lines. |

Each `series[]` must resolve exactly one time window. Supported shapes should reuse existing time
machinery:

```json
{ "startTime": "2026-07-02T04:00:00Z", "endTime": "2026-07-02T06:00:00Z" }
```

```json
{ "relativeDuration": "2h" }
```

```json
{ "relativeDuration": "2h", "anchorOffset": "7d" }
```

A top-level `anchorTime` may be used so relative windows in one request resolve against one clock.

---

## 5. X-axis mode selection

The model-facing description should be explicit:

| User intent | Use |
|---|---|
| "Compare these Things in the selected window" | `absolute_time` |
| "This window vs last week / yesterday / previous shift" | `elapsed_time` |
| "Compare shape/profile even though durations differ" | `normalized_time` |

If the user does not specify a mode, the executor applies deterministic defaults (**review-0 Q2
option C**): tool description teaches explicit choice; omitted field is resolved server-side.

**Default heuristic when `xAxisMode` omitted (P1 + P2 — unchanged):**

1. If all resolved series windows are equivalent within tolerance (`WINDOW_EQUALITY_TOLERANCE_SECONDS`),
   default to **`absolute_time`**.
2. Otherwise default to **`elapsed_time`**.

The executor **does not** infer shape/profile intent from omitted mode (review-6 Q6). The model MUST
set **`xAxisMode: "normalized_time"`** explicitly for shape/profile / different-duration comparisons.
Routing text and tool schema teach when to use each mode.

Do not reject merely because two series have:

- the same time window but different Things;
- different time windows but the same Thing;
- different time windows and different Things;
- different window durations.

Reject only when the chart would be invalid or misleading for concrete reasons: missing window,
non-numeric property, incompatible known units, too many series, impossible point budget, or exact
duplicate series (same `thingName`, `propertyName`, resolved `start`, resolved `end`).

**Duplicate rule (review-0 Q4 / C3):** reject identical `(thingName, propertyName, start, end)` pairs
only. **Allow** two different Things sharing the same resolved calendar window (required for
acceptance §11.1 mixed overlay).

---

## 6. Window duration and alignment

`elapsed_time` does not require equal durations. The X domain can be:

```text
0 .. max(series duration seconds)
```

Shorter series simply stop earlier. The chart should not fabricate trailing points.

`normalized_time` maps every series to the same normalized domain:

```text
normalizedX = (timestamp - windowStart) / (windowEnd - windowStart)
```

The first version does not need interpolation or resampling. Uneven sample timestamps are acceptable
for drawing. If later statistical comparison requires point-by-point alignment, that should be a
separate explicit resampling option.

If the user asks to compare by fixed unit length, such as "align by the first 30 minutes" or "align
by cycle percentage", the tool should express that through `elapsed_time` or `normalized_time`
rather than rejecting the chart.

---

## 7. Reference lines and limits

This topic must expose chart reference lines as first-class tool input.

The current chart contract and UI already support:

```json
"y_reference_lines": [
  { "y": 25, "role": "ucl", "label": "UCL" }
]
```

Supported roles:

```text
usl, ucl, lcl, lsl, target, limit, warning
```

The new tool should accept:

```json
{
  "yReferenceLines": [
    { "y": 8.0, "label": "Upper limit", "role": "usl" },
    { "y": 2.0, "label": "Lower limit", "role": "lsl" },
    { "y": 5.0, "label": "Target", "role": "target" }
  ]
}
```

Executor behavior:

- convert `yReferenceLines` to `ChartBlock.y_reference_lines`;
- cap at 12 lines, matching `CHART_CONTRACT.md`;
- **`y` must be finite numeric** — non-finite or malformed entries **fail** with
  **`HISTORY_OVERLAY_INVALID_REFERENCE_LINE`** (review-0 C4: hard error, not silent drop);
- unknown **role** normalizes to **`limit`**;
- reference lines are chart-global in this topic;
- the UI must show the lines and expand the Y domain when needed.

Acceptance must include upper-limit and lower-limit prompts, not only trend prompts.

---

## 8. Tool-surface changes

Before this topic, a live chart-capable agent may expose all three of these concepts:

```text
query_property_history
build_period_over_period_chart
build_multi_series_history_chart
```

After this topic:

```text
query_property_history
build_history_overlay_chart
```

Rules:

- `query_property_history` remains for one Thing / one property / one window.
- `query_property_history` should not be the happy path for overlay charts.
- `build_period_over_period_chart` is removed from model-facing built-ins.
- `build_multi_series_history_chart` is removed from model-facing built-ins.
- Old tool names are not kept as executor-only.
- Old tool names are not accepted by playbook validation.
- Existing playbooks, evals, docs, and dev data that reference old names must be updated in the same
  implementation topic.

The expected model-facing tool count decreases by one once old names are removed and the unified
tool is added.

---

## 9. Implementation outline

### 9.1 Internal series model

Use one internal representation:

```java
ResolvedHistoryOverlaySeries {
  String label;
  String thingName;
  String propertyName;
  Instant windowStart;
  Instant windowEnd;
  String unit;
  List<Point> points;
}
```

Implementation may reuse or refactor existing classes:

- `BuildPeriodOverPeriodChartExecutor`
- `BuildMultiSeriesHistoryChartExecutor`
- `PeriodOverPeriodChartBuilder`
- `MultiSeriesHistoryChartBuilder`
- `HistorySeriesComposerSupport`
- `PeriodOverPeriodPeriodResolver`
- `MultiSeriesSharedWindowResolver`

But the registered model-facing `ToolDefinition` should be only `build_history_overlay_chart`.

### 9.2 Builder output

For `absolute_time`:

- output normal datetime `x[]`;
- output `requested_time_range` when all windows share one range;
- omit `xAxisMode` or set contract value `"absolute"`.

For `elapsed_time`:

- output numeric elapsed seconds as `x[]`;
- output `xAxisMode = "elapsed"`;
- output `elapsedDomain`;
- omit `requested_time_range`.

For `normalized_time`:

- extend `CHART_CONTRACT.md` with a normalized X-axis mode;
- output normalized numeric `x[]`;
- use a stable domain such as `0..1` or `0..100`, chosen explicitly in contract text;
- do not require resampling in the first delivery.

For all modes:

- attach `y_reference_lines` when requested;
- include provenance in `source`;
- return a structured success payload with `code = CHART_EMITTED`;
- add exactly one pending chart block for one successful tool call.

### 9.3 Error codes

Suggested codes:

| Code | Meaning |
|---|---|
| `HISTORY_OVERLAY_TOO_FEW_SERIES` | Need at least 2 series. |
| `HISTORY_OVERLAY_TOO_MANY_SERIES` | Exceeds configured cap. |
| `HISTORY_OVERLAY_MISSING_PROPERTY` | No `propertyName`. |
| `HISTORY_OVERLAY_MISSING_THING` | A series lacks `thingName`. |
| `HISTORY_OVERLAY_INVALID_TIME_WINDOW` | A series cannot resolve a time window. |
| `HISTORY_OVERLAY_PROPERTY_NOT_NUMERIC` | Property is not chartable numeric on one Thing. |
| `HISTORY_OVERLAY_UNIT_MISMATCH` | Known non-empty units differ across Things. |
| `HISTORY_OVERLAY_INVALID_X_AXIS_MODE` | Unknown X-axis mode. |
| `HISTORY_OVERLAY_NO_DATA` | No series has chartable points. |
| `HISTORY_OVERLAY_INVALID_REFERENCE_LINE` | Non-finite or malformed reference-line input. |

Do not return `POP_MULTI_SERIES_UNAVAILABLE` from this tool. That code belongs to the retired split.

---

## 10. Contract and UI impact

Existing contract already covers:

- line charts;
- multiple `series[]`;
- `xAxisMode = "elapsed"`;
- `elapsedDomain`;
- `y_reference_lines`.

Required contract work (P1):

1. **Consolidate** `CHART_CONTRACT.md` §3.0e and §3.0f under one producer section for
   **`build_history_overlay_chart`** (absolute-mode + elapsed-mode as X-axis variants of one tool;
   review-0 C2). Do not leave two tool-named producer sections that teach the retired split.
2. Clarify elapsed charts **may** include different Things and unequal window durations;
   `elapsedDomain.end` = **max** series duration in seconds (review-0 Q3).
3. Add producer notes for `build_history_overlay_chart` success payload and scatter `chart_kind`.
4. Remove producer references that imply PoP and multi-series are separate model-facing tools.

**Contract version (review-0 C1):** P1 loosens normative elapsed semantics (mixed Things, unequal
durations). This **is** a `CONTRACT_VERSION.md` bump **candidate** even though no new wire *value*
is added. Per §7.1, the implementor **flags** the need in the P1 packet; the actual coordinate
bump requires explicit User instruction at release cut.

P2 contract work (**shipped @ 0.1.205**): `normalized_time` → contract `"normalized"` mode and widget normalized-axis rendering. (Pre-ship design note below retained as historical.)

UI work:

- verify existing elapsed and reference-line rendering still works;
- add normalized-axis rendering if normalized mode is included;
- verify one chart block with multiple series renders as one artifact;
- no widget interaction change is expected unless contract version packaging requires a widget build.

---

## 11. Acceptance criteria

> **Design record (shipped @ 0.1.205).** Bullets below describe the original implementation scope and verification plan — not an open work list.

### 11.1 Mixed shifted + cross-Thing overlay

Prompt:

```text
please show me the trend of Current Draw in the selected time window and the same in last week,
also the same property from BOS StackingRobot 02 in the same time window in last week
```

Expected:

- one `build_history_overlay_chart` call;
- one emitted chart;
- `parlerChartWireEmittedCount = 1`;
- `series.length = 3`;
- X-axis mode is `elapsed_time` / contract elapsed;
- no calls to retired chart tools;
- no repeated `query_property_history` chart assembly path.

### 11.2 Same-window cross-Thing

Prompt:

```text
compare Current Draw for BOS StackingRobot 01 and BOS StackingRobot 02 in the selected time window
```

Expected:

- one emitted chart;
- `series.length = 2`;
- X-axis mode defaults to `absolute_time`;
- chart uses real timestamps.

### 11.3 Same Thing shifted windows

Prompt:

```text
show Current Draw for BOS StackingRobot 01 in the selected window and the same window last week
```

Expected:

- one emitted chart;
- `series.length = 2`;
- X-axis mode defaults to `elapsed_time`.

### 11.4 Limits

Prompt:

```text
show Current Draw for BOS StackingRobot 01 and BOS StackingRobot 02 in the selected window,
with upper limit 8 amps and lower limit 2 amps
```

Expected:

- one emitted chart;
- two data series;
- `y_reference_lines` includes an upper line at 8 and a lower line at 2;
- reference lines render and remain visible even if data is outside the band.

### 11.5 Different-duration shape comparison

Prompt:

```text
compare the startup shape of BOS StackingRobot 01 over the first 30 minutes today
with BOS StackingRobot 02 over the first 45 minutes yesterday
```

Expected:

- no rejection solely because windows have different durations;
- **preferred (1.0):** explicit **`normalized_time`** → normalized X on wire;
- **acceptable fallback (0.75):** **`elapsed_time`** + prose that one series has a longer span;
- eval suites MUST NOT reject the elapsed fallback when normalized is preferred (review-6 A1).

### 11.6 Removal of old tools

Expected:

- runtime snapshot no longer lists `build_period_over_period_chart`;
- runtime snapshot no longer lists `build_multi_series_history_chart`;
- playbook allowlist no longer lists old names;
- **`BuiltInTools.java` routing prose** no longer references old tool names or
  **`POP_MULTI_SERIES_UNAVAILABLE`** / **`ROUTE_MULTI_SERIES`** (review-0 C5);
- **`llm_tool_routing_guide.txt`**, **`docs/agent/LLM_CONTEXT.md`**, playbooks, docs, evals, and dev
  data no longer instruct callers to use old names or the PoP redirect;
- **`docs/archived/2026-07-07T224124-consolidate-history-overlay/period-over-period-live-smoke.md`** and
  **`docs/archived/2026-07-07T224124-consolidate-history-overlay/multi-series-live-smoke.md`**
  archived as historical smoke checklists (no active old-tool instructions);
- docs/evals/dev data no longer instruct new users to call old names;
- any stale old-name playbook fails validation until updated.

**Scrub counter-guardrail (review-1):** do **not** rewrite audit history — archived review packets,
`CONTRACT_VERSION.md` history rows, release notes, and planning archives stay as-is. Scrub only
**current** model-facing / user-instruction surfaces.

---

## 12. Documentation updates when implemented

Implementation should update:

- `CONTRACTS/CHART_CONTRACT.md`
- `CONTRACTS/CONTRACT_VERSION.md` if normalized mode or other normative wire semantics change
- `docs/archived/2026-07-07T224124-consolidate-history-overlay/period-over-period.md`
- `docs/archived/2026-07-07T224124-consolidate-history-overlay/multi-series.md`
- `docs/agent/all-tools.md`
- `docs/agent/chart-intent.md`
- `docs/agent/LLM_CONTEXT.md` — runtime LLM routing; **required** scrub (review-1 L1)
- `docs/archived/2026-07-07T224124-consolidate-history-overlay/period-over-period-live-smoke.md` — archived
- `docs/archived/2026-07-07T224124-consolidate-history-overlay/multi-series-live-smoke.md` — archived
- `docs/agent/README.md` if needed
- `docs/agent/evals/history_overlay_v1.yaml` (new primary overlay eval)
- `docs/agent/evals/period_over_period_v1.yaml` — thin-wrap to new tool **or** retire
- `docs/agent/evals/multi_series_v1.yaml` — thin-wrap to new tool **or** retire
- playbook docs and allowlist docs that mention chart-capable tools
- ParlerGuidance built-in-tool and chart-training chapters after implementation lands

Follow `docs/review-framework.md` for review packets during implementation.

---

## 13. Implementation phases (proposed)

Reviewers should confirm phase boundaries before any Java/UI work.

| Phase | Scope | Retires / adds | Contract / UI |
|---|---|---|---|
| **P1 — Core overlay (v1 first slice)** | `build_history_overlay_chart` with `absolute_time` + `elapsed_time`; per-series windows in `series[]`; **`chart_kind` line/scatter**; `yReferenceLines`; executor two-branch default (§5); remove model-facing PoP + multi-series tools and **`POP_MULTI_SERIES_UNAVAILABLE`** path; consolidate contract §3.0e/§3.0f; update routing guide, playbooks, evals, docs. | Replaces both old tools for §11.1–§11.4 and §11.6. | Reuse `"absolute"` / `"elapsed"` wire modes; **normative elapsed semantics change** (contract-version candidate). **No** `normalized_time` in P1 tool enum. |
| **P2 — Normalized axis** | **`normalized_time`** tool input → contract `"normalized"` + UI tick/domain rendering. | Enables §11.5 shape comparison without misleading elapsed stretch. | **Shipped @ 0.1.205** (extension) / widget **0.1.89**; `CHART_CONTRACT.md` + `CONTRACT_VERSION.md` aligned. |
| **P3 — Hardening** | Live smoke, eval refresh, guidance chapters, dev_data / playbook sweep. | — | — |

**Recommendation:** land **P1** as the first implementation milestone; treat **P2** as a separate
review slice unless reviewers insist normalized mode is required for v1 acceptance §11.5.

Internal reuse: P1 may delegate to refactored `PeriodOverPeriodChartBuilder` / `MultiSeriesHistoryChartBuilder` logic behind one executor; physical deletion of old executors is not required in P1 if they are unreachable from `BuiltInTools`.

---

## 14. Design decisions (resolved — review-0, 2026-07-03)

Both reviewers **continue**. Decisions below are binding for P1 unless a **`## User Ruling`**
overrides them.

| Id | Decision |
|---|---|
| **Q1 / IQ1** | **Defer `normalized_time` to P2.** P1 ships `absolute_time` + `elapsed_time` only. §11.5 satisfied by elapsed X + prose that one series has a longer span. |
| **Q2 / IQ2** | **Option C:** tool description teaches explicit `xAxisMode`; executor applies §5 two-branch default when omitted. |
| **Q3** | `elapsedDomain.end` = **max(durationSeconds)** across emitted series. |
| **Q4 / IQ3 / C3** | Reject exact duplicate `(thingName, propertyName, start, end)` only; allow same window on different Things. Drop `POP_DUPLICATE_PERIOD` / redirect guards from unified path. |
| **Q5** | `series[].sourceWindow` **required** on elapsed output only; absolute mode uses `requested_time_range` when windows share one range (multi-series wire style). |
| **Q6** | **`chart_kind` line (default) + scatter in P1** — required parity with retired multi-series; not a new UI mode. |
| **Q7** | Single constant **`HISTORY_OVERLAY_MAX_TOTAL_EMITTED_POINTS = 5000`**; fair per-series subsample. |
| **Q8** | One new overlay eval file covering same-window, shifted, mixed, limits, duplicate rejection, old-tool removal; legacy eval filenames may remain as **thin wrappers that call only `build_history_overlay_chart`** — they **must not** retain old tool names (review-1 L2). |
| **IQ4** | Reuse old Java classes internally OK if old names unreachable from `BuiltInTools`, playbook validation, and tool discovery. |
| **IQ5 / C5** | Keep `query_property_history` for single-series; routing guide **discourages** repeated history calls for overlay assembly (steering only). Scrub all redirect prose. |
| **C1** | Flag `CONTRACT_VERSION.md` bump candidate in P1 packet; do not bump coordinates without User cut order. |
| **C2** | Consolidate contract §3.0e/§3.0f under one `build_history_overlay_chart` producer section. |
| **C4** | Reference lines: hard error on non-finite/malformed `y`; unknown role → `limit`. |

---

## 15. Schema delta vs retired tools

| Aspect | `build_period_over_period_chart` | `build_multi_series_history_chart` | `build_history_overlay_chart` (proposed) |
|---|---|---|---|
| Series input | `periods[]` with per-period window fields | `series[]` (Thing only) + **one shared** top-level window | `series[]` with **per-series** window resolution |
| X axis | Always elapsed | Always absolute | **`xAxisMode`** selects mode |
| Cross-Thing same window | Redirect / error | Supported | Supported (`absolute_time`) |
| Cross-Thing shifted windows | Rejected / redirect | Not expressible | Supported (`elapsed_time`) |
| Reference lines | Not in tool schema today | Not in tool schema today | **`yReferenceLines`** first-class input (§7) |
| `chart_kind` | Line only | Line + scatter | Line + scatter (P1) |
| Mixed duration elapsed | Rejected (`POP_WINDOW_DURATION_MISMATCH`) | N/A | Allowed; domain = max duration (§6) |

---

## 16. P1 implementation checklist (review-1 + review-2)

Binding P1 work list. Review-1 reviewers **continue**; review-2 folds checklist refinements L1–L4.

### Agent (`parler-agent/`)

1. Register **`build_history_overlay_chart`** in `BuiltInTools.java` (schema per §4).
2. New executor + builder; remove old tools from `BuiltInTools`, `ToolBuckets`, `PlaybookToolAllowlist`.
3. Drop **`POP_MULTI_SERIES_UNAVAILABLE`** and these PoP guards from the unified path (do not bypass):
   **`PeriodOverPeriodPopSupport.detectSameWindowCrossDevice`**, **`PeriodOverPeriodChartBuilder.validateDurationMatch`**
   (`POP_WINDOW_DURATION_MISMATCH`), **`PeriodOverPeriodChartBuilder.validateNoDuplicateWindows`**
   (`POP_DUPLICATE_PERIOD` across different Things).
4. §5 two-branch executor default when `xAxisMode` omitted.
5. Duplicate-series reject (exact tuple); unit validation; **`HISTORY_OVERLAY_MAX_TOTAL_EMITTED_POINTS = 5000`** fair subsample.
6. Reference lines per §7 / C4.
7. JUnit: §11.1–§11.5, duplicate reject, default mode, registry absence of old tools, **reference-line hard error + unknown-role normalize** (review-1 L3).

### Contract (`CONTRACTS/`)

8. Consolidate `CHART_CONTRACT.md` §3.0e + §3.0f → one **`build_history_overlay_chart`** producer section.
9. Update elapsed normative text (mixed Things, unequal durations, `elapsedDomain.end = max(duration)`).
10. Flag `CONTRACT_VERSION.md` bump candidate in implementation notes; **no coordinate bump** without User release cut (C1).

### Routing / eval / docs (steering scrub — review-1 L1)

11. Rewrite **`llm_tool_routing_guide.txt`** overlay section.
12. Scrub **`BuiltInTools.java`** tool descriptions.
13. Scrub **`docs/agent/LLM_CONTEXT.md`** — required; highest-authority runtime routing text.
14. Archived: **`period-over-period-live-smoke.md`**, **`multi-series-live-smoke.md`** (see consolidate stamp).
15. New **`docs/agent/evals/history_overlay_v1.yaml`**; legacy evals thin-wrap **only** with new tool name (L2).
16. Archived **`period-over-period.md`**, **`multi-series.md`**; **`chart-intent.md`** points to overlay. **`all-tools.md`** inventory refreshed @ **2026-07-08** (Pilot 3 / consolidate topic).
17. Update tests: **`LlmToolRoutingGuideChartWordingTest`**, registry/allowlist tests.

### Non-targets (do not expand P1 scope)

- **`PropertyToolsExecutor.java`** javadoc mentioning old PoP tool — update when file is touched; not a routing-scrub gate (L4).
- Archived packets, `CONTRACT_VERSION.md` history rows, release notes — not scrub targets.

### Out of P1 scope (historical — P1/P2/widget shipped @ 0.1.205)

- `normalized_time` (P2) — **shipped** @ extension **0.1.205** (historical P1 deferral).
- Widget build for overlay release — **shipped** @ widget **0.1.89** (historical "unless User orders release cut" deferral).

### User-run live smoke (deferred — out-of-band hardening)

- User-run live smoke remains out-of-band per doc header; not a convention-closure gate.

---

## 17. P2 implementation checklist (normalized axis — agreed review-6)

**Gate:** review-6 **continue** with Q6 amendment (explicit `normalized_time` only). Implementation in review-7 slice.

### Binding decisions (review-6)

| Id | Decision |
|---|---|
| **P2-Q1** | Add **`normalized_time`** to tool enum + routing guide. |
| **P2-Q2** | Wire **`xAxisMode: "normalized"`**. |
| **P2-Q3** | Required **`normalizedDomain: { start: 0, end: 1 }`**. |
| **P2-Q4** | `normalizedX = (t - start) / duration`; **clamp to [0, 1]** in builder (A2); zero-duration → **`HISTORY_OVERLAY_INVALID_TIME_WINDOW`**. |
| **P2-Q5** | Omit **`requested_time_range`** / **`elapsedDomain`**; keep per-series **`sourceWindow`**. |
| **P2-Q6** | **Amended:** omitted mode keeps P1 two-branch default only — **no** executor shape-intent inference. |
| **P2-Q7** | UI percent ticks **`0%`…`100%`**; update **`wireAdapter.js`**, **`chart-draw.js`**, **`types.js`** JSDoc (A3). |
| **P2-Q8** | Extend **`CHART_CONTRACT.md`** §3.0e; **`CONTRACT_VERSION.md`** patch at User release cut (fold P1 elapsed candidate). |
| **P2-Q9** | JUnit §11.5, wire validation tests, eval with normalized preferred + elapsed fallback (A1). |
| **P2-Q10** | Widget patch bump when UI ships. |

### P2 non-targets

- Point-by-point resampling / interpolation across series.
- Multi-property overlay.
- New chart kinds beyond line/scatter.

# Multi-Thing Alert Query

**Status:** **M1 shipped** @ extension **0.1.202** — **`query_alert_summary`** accepts **`thingNames[]`** (1–25); N≥2 **`ALERT_SUMMARY_MULTI`**; **`query_alert_history`** / **`acknowledge_alerts`** remain scalar **`thingName`**. Design history: **`docs/archived/2026-07-03T164342-multi-thing-alert-query/`**.

> **Read this first if you are new to the repo.** This document is written to be
> **self-contained**: the implementor (Cursor) and the reviewers may have **no carried-over
> context** from the discussion that produced it. §0 gives background — what `parler`
> is, how the alert tools work, where the result-size cap bites, why this topic
> existed, and the M1 migration surface. §1 records the pre-ship problem; §2+ documents the
> **shipped** wire and rollup semantics. Everything *normative* lives in the cited
> contracts/docs — this file points at them, it does not restate them as law.
>
> **Shipped design.** M1 landed @ **0.1.202**; normative contract:
> **`CONTRACTS/TAXONOMY_RESOLVER.md`** §7.2. §0 retains pre-M1 orientation where noted.
>
> **Live-test ownership:** every step that requires a **live run against the dev server is owned
> by the User**, not the implementor. The implementor writes the offline gates and *specifies*
> the live measurement (what to run, what to record, the pass bar); the implementor does **not**
> execute live runs. See §4.

---

## 0. Background (orientation for a fresh agent)

### 0.1 What `parler` is

`parler` is a monorepo for an AI agent that runs **inside PTC ThingWorx** (an industrial IoT
platform). The agent answers operator questions about industrial assets by calling tools against
the live ThingWorx model and an indexed corpus of equipment manuals. The piece relevant here is
the **Java extension** under `parler-agent/` (a ThingWorx extension = a JAR + entity XML imported
into the platform). It is **Java 11** — **no records**; use plain final classes with accessors.

Key facts a fresh agent needs:

- **Source of truth for the repo:** `AGENTS.md` (root). Read it before changing code.
- **AI operating contract:** `docs/review-framework.md` for design-review workflow on active topics.
- **Build the extension (offline tests):**
  `cd parler-agent && ./gradlew test -PuseLocalTwxLib=true --console=plain`. The
  `-PuseLocalTwxLib=true` flag is **mandatory** when `parler-agent/twx-lib/all` is populated
  (`.cursor/rules/parler-agent-gradle-local-lib.mdc`); without it the build times out on remote
  resolution.
- **Version bumps:** patch segment only — `parler-agent/build.gradle` `revisionVersion` and
  `parler-agent/metadata.xml` `packageVersion`, in lockstep. **Never bump without an explicit
  User instruction** (`.cursor/rules/thingworx-extension-version.mdc`). M1 wire change shipped @
  **0.1.202**; current **`revisionVersion`** lives in **`parler-agent/build.gradle`**.
- **Contract coupling:** any change to a tool's **wire shape** (the JSON the model sends/receives)
  must update the relevant `CONTRACTS/*.md` and bump `CONTRACTS/CONTRACT_VERSION.md` **in the same
  commit** (`CLAUDE.md` guardrails). The alert tools are pinned by `CONTRACTS/TAXONOMY_RESOLVER.md`
  §7.2 — see §0.5.
- **No backward-compat shims.** Per the project norm (the repo is pre-production; see
  `.claude` memory *"No migration / backward-compat during dev"*): when a wire contract changes,
  **rewrite cleanly and migrate every caller in the same change** — do not keep a deprecated
  alias path alongside the new one. This materially simplifies §2.
- **Live diagnostics (User-run):** `uv run parler-collect-live --window 30m -o logs` pulls
  ApplicationLog + AgentMessageStream + AgentThing status from the dev server (needs `.env` with
  `DEV_SERVER` + `DEV_KEY`). See `docs/agent/live-diagnostics.md`.

### 0.2 The alert tools (shipped shape)

There are **three** built-in alert tools, all backed by the ThingWorx resource
`Resources["AlertFunctions"]`. The design of record is `docs/operations/alert-solution.md`
(read §5 and §8). **`query_alert_summary`** accepts **`thingNames[]`** (1–25 canonical names per
call; N=1 flat **`INFOTABLE`** / **`INFOTABLE_LARGE`**, N≥2 **`ALERT_SUMMARY_MULTI`**). History and
acknowledge remain **single-Thing** — scalar **`thingName`**.

| Tool | Platform service | Required arg | Returns |
|------|------------------|--------------|---------|
| `query_alert_summary` | `AlertFunctions.QueryAlertSummaryForThing` (per-Thing fan-out inside one dispatch) | **`thingNames[]`** (1–25) | N=1: current alert **summary** rows for **one** Thing; N≥2: per-Thing rollups in **`ALERT_SUMMARY_MULTI`** |
| `query_alert_history` | `AlertFunctions.QueryAlertHistory` | `thingName` (scalar) | time-bounded alert **history** rows for **one** Thing |
| `acknowledge_alerts` | `AlertFunctions.AcknowledgeAlert*` | `thingName` (scalar) | acknowledges alerts on **one** Thing (mutating) |

**Where each piece lives** (shipped @ 0.1.202 — confirm line numbers before editing):

- **Wire schemas** are built in Java, not JSON resources:
  `parler-agent/src/main/java/com/thingworx/things/agent/tools/BuiltInTools.java` —
  **`queryAlertSummaryDef()`** (~L924) sets `required = {"thingNames"}` and an array **`thingNames`**
  prop (`minItems: 1`, `maxItems: 25`); **`queryAlertHistoryDef()`** (~L968) and
  **`acknowledgeAlertsDef()`** (~L1016) keep scalar **`thingName`** with `required = {"thingName"}`.
- **Registration / dispatch:** `BuiltInTools.java` ~L138–140 wires each def to its executor.
- **Summary executor:** `AlertToolsExecutor.doQueryAlertSummary` parses **`thingNames`**, enforces
  the 25-name cap, runs **`ScalarThingnamePreflight.gateApplicationThings`**, fans out to
  **`QueryAlertSummaryForThing`** per resolved name, and builds N=1 inline tables or N≥2
  **`AlertSummaryMultiRollup`** JSON.
- **History / acknowledge executors:** `AlertToolsExecutor` binds a **single** Thing via
  **`ScalarThingnamePreflight.gateApplicationThing`** and calls the per-Thing platform services
  (history ~`doQueryAlertHistory`; acknowledge paths unchanged).
- **Identity gate:** summary uses **list-gate** (`gateApplicationThings`); history/ack use **scalar**
  **`gateApplicationThing`**. Non-canonical labels return **`IDENTITY_RESOLUTION_REQUIRED`** (summary
  can return partial success with **`identityErrors[]`** when other names resolve).
- **Result formatting + the size cap:** N=1 summary paths and history still route InfoTables through
  `InvokeServiceExecutor.formatBuiltinInfotableResult` → INLINE / **`INFOTABLE_LARGE`** at
  **`ToolResultEgressGateway.ARRAY_SAMPLE_LIMIT` (20)** rows. N≥2 summary returns compact per-Thing
  rollups (**`topAlerts`** capped at 3) — sidestepping naive row concatenation under the global cap
  (see §2.4).

### 0.3 How tools reach the model, and the two budget pressures

Every agent turn builds an LLM request whose `tools` array carries one wire object per
model-facing tool (`{name, description, input_schema}`), assembled in
`AgentThing.getMergedToolDefinitions()`. The request is sized by `ContextBudgetPlanner`
(`parler-agent/.../compaction/`), which logs `LLM_CONTEXT_PLAN` / `LLM_CONTEXT_PLAN_FAIL`.

Two pressures matter for this topic:

1. **Tool-schema overhead** — the `tools` array is fixed per-round overhead (`toolSchemaChars`).
   A *sibling* topic (`tool-schema-admission-control`, design at
   `docs/operations/tool-schema-admission-control.md`) addresses this by admission control. **This
   topic does not change which tools are advertised.**
2. **Conversation-history growth** — every tool call appends a `tool_use` + `tool_result` message
   pair to the transcript, growing `stableChars`/`messages`. When the agent calls a single-Thing
   tool **once per Thing** across N Things, it adds **N round-trips and N result pairs** to history.
   This is the pressure **this topic** attacks: collapsing N single-Thing alert calls into **one**
   multi-Thing call cuts round-trips and history growth at the source.

### 0.4 Why this topic exists (the evidence)

The `tool-schema-admission-control` topic ran a live off-vs-lazy measurement (recorded in
`docs/operations/tool-schema-admission-control.md` §4.3, dev-bench 2026-06-28). The decisive
scenario, **Q2**, was the prompt *"compare the health status of assets between USA and Germany"*
on a Mashup card host-context. In that single turn the model called **`query_alert_summary` 14
times** — once per Thing — and:

- in `off` mode the turn hit **`OVERHEAD_EXCEEDS_CAP`** and produced **no answer**; the accumulated
  history (`stableChars=31638`) from the fan-out, plus the fixed tool block, overshot the request
  cap;
- the same prompt is the textbook "compare alerts across many assets" use case — which **pre-M1 (before 0.1.202)** the alert layer could only serve by single-Thing fan-out.

`docs/operations/alert-solution.md` §8 ("Cross-cutting verdict") deliberately **declined** a fleet
alert tool: *"The three-tool split is sufficient … A fourth 'fleet alert' tool is justified only
with explicit platform backing and proof that fan-out is insufficient."* It also notes a historical
fleet macro was **removed in extension 0.1.161**. The §4.3 Q2 incident is precisely that proof:
fan-out is not merely more expensive, it can **break the turn**. This topic does **not** add a
fourth tool — it makes the **existing** summary (and optionally history) tool accept **multiple
Things in one call**, which is the smallest change that removes the fan-out.

### 0.5 The migration surface (the full blast radius — read before scoping)

A multi-Thing schema is a **wire-contract change**. The implementor MUST migrate every caller in
the same change (no compat alias — §0.1). The complete surface, by category:

- **Schemas / registration / executor:** `BuiltInTools.java` (defs + registration),
  `AlertToolsExecutor.java` (executor), `ScalarThingnamePreflight.java` (needs a list-gate variant).
- **Result-shape consumers:**
  `parler-agent/.../playbook/PlaybookAlertSummaryRows.java` (parses `rows`/`sampleRows`/`alertRows`
  from `query_alert_summary` output — per-Thing attribution changes here),
  `parler-agent/.../playbook/PlaybookEvidenceFormatter.java` (alert history envelope fields),
  `parler-agent/.../ParlerTableWirePresentationTitles.java`,
  `parler-agent/.../compaction/LlmToolResultCohortMerger.java`,
  `parler-agent/.../tools/TabularChartRoundHooks.java`.
- **Routing guidance (model-facing prose to rewrite):**
  `parler-agent/src/main/resources/com/thingworx/things/agent/llm_tool_routing_guide.txt` — the
  "Alert comparison" section explicitly tells the model *"`query_alert_summary` requires `thingName`
  per Thing … you cannot treat it as hierarchy-scoped"* and prescribes the per-Thing fan-out loop.
  Also `AlertPromptDefaults.java` ("Current active alerts on **one** Thing").
- **Skills (config-repo sample data under `dev_data/`):**
  `dev_data/scpa_utilization/skills/region_health/SKILL.md` (step 4: *"For each returned Thing, call
  `query_alert_summary`…"*),
  `dev_data/scpa_utilization/skills/asset_pair_health/SKILL.md` (same loop pattern). These are
  user-curated but **editable** (`.cursor/rules/dev-data-user-owned.mdc` / `.claude` memory).
- **Playbooks (`dev_data/` JSON + test fixtures):**
  `dev_data/playbooks/cross_region_health/playbook.json`,
  `dev_data/playbooks/cross_asset_pair_health/playbook.json`, the `dev_data/scpa_utilization/…`
  copies, and the JUnit fixtures under `parler-agent/src/test/resources/playbook-*/…` and
  `docs/agent/playbook-converter/examples/…`. `PlaybookToolAllowlist.java` lists the tool names.
- **Contracts:** `CONTRACTS/TAXONOMY_RESOLVER.md` §7.2 (the scalar-`thingName` alert gate semantics,
  version-history entry `2.0.10`) and `CONTRACTS/CONTRACT_VERSION.md` (entry tying
  `AlertToolsExecutor` + `ScalarThingnamePreflight` to the gate). Both must be updated + version
  bumped with the code.
- **Docs to re-snapshot / update:** `docs/operations/alert-solution.md` (the three-tool design of
  record, esp. §5 and §8's now-superseded verdict), `docs/agent/all-tools.md` (the regenerated
  per-tool schema dump + token table — the alert schemas change), `docs/agent/query-spec.md`.
- **Tests (golden + behavior):** `tools/AlertToolsExecutorThingnamePreflightTest.java`,
  `tools/BuiltInToolsLlmSchemaRegistryTest.java` (golden schema registry — will fail until updated),
  `ParlerToolTableWireUtilAlertWireTest.java`, `ParlerInvokeServiceInfotableTableWireTest.java`,
  the `playbook/Playbook*Test.java` family that exercises alert steps/rows, `tools/ToolBucketsTest.java`.

`acknowledge_alerts` is **mutating** (it acknowledges alerts). It is **out of scope** for
multi-Thing in this topic (§5) — batch acknowledgment across Things is a safety-sensitive change
that deserves its own topic. Keep it single-Thing.

### 0.6 Shipped verification snapshot (@ 0.1.202)

Cross-checked against shipped Java after M1:

| Claim | Verified |
|-------|----------|
| `query_alert_summary` schema is **`thingNames[]`**, `required = {"thingNames"}` | `BuiltInTools.queryAlertSummaryDef()` ~L924–954 |
| Summary executor fans out per name, N≥2 → **`ALERT_SUMMARY_MULTI`** | `AlertToolsExecutor.doQueryAlertSummary`; `AlertSummaryMultiRollup.RESULT_KIND` |
| History/ack remain scalar **`thingName`** | `BuiltInTools.queryAlertHistoryDef()` / `acknowledgeAlertsDef()` |
| N=1 summary uses `formatBuiltinInfotableResult` → INLINE / `INFOTABLE_LARGE` at 20 rows | `InvokeServiceExecutor.formatBuiltinInfotableResult`; `ToolResultEgressGateway.ARRAY_SAMPLE_LIMIT = 20` |
| Routing guide prescribes single-call **`thingNames[]`** comparison | `AlertPromptDefaults` + `llm_tool_routing_guide.txt` |
| `PlaybookAlertSummaryRows` handles **`ALERT_SUMMARY_MULTI`** and flat INFOTABLE | `PlaybookAlertSummaryRows.fromToolOutput` |
| M1 wire change | **0.1.202** (`parler-agent/CHANGELOG.md`) |
| Native multi-Thing `AlertFunctions` service in-repo | **Not found** — server-side per-Thing loop inside one tool dispatch |

---

## 1. Problem statement

The "compare alerts across many assets" intent — a first-class operator question on Mashup cards
(*"compare health between USA and Germany"*) — is served today only by calling a **single-Thing**
alert tool **once per Thing**. That fan-out:

1. **adds N round-trips** (latency: each is an LLM round + a tool dispatch), and
2. **grows conversation history by N tool-result pairs**, which — as measured in
   `tool-schema-admission-control` §4.3 Q2 — can push a turn over `effectiveRequestCapChars` and
   **fail it outright** (`OVERHEAD_EXCEEDS_CAP`, no answer).

The fix is to let one alert call cover **multiple Things**, so the common comparison reduces to a
**single** dispatch and a **single** result pair. The open question this topic must answer
carefully (because it determines whether the change is net-positive) is **result shape under the
~20-row inline cap** (§0.2): naïvely concatenating rows from N Things overflows the 20-row
`ARRAY_SAMPLE_LIMIT`, so the model would see ~20 rows total with **no per-Thing fairness** — which
could make multi-Thing *worse* than fan-out for answer quality. §2.4 is the heart of the design.

---

## 2. Design

### 2.1 Scope recommendation: `query_alert_summary` first; `query_alert_history` deferred

These two tools have **different payload natures**, and the 20-row cap treats them differently:

- **`query_alert_summary`** is naturally **aggregatable per Thing** (counts of active/unacked
  alerts, by priority/property, the top few). A multi-Thing summary can return **one compact
  rollup object per Thing** — N objects, not a flat row list — which **sidesteps the 20-row global
  cap entirely** (you cap *rows per Thing*, not the comparison). This is a clean, high-value win and
  maps exactly onto the Q2 use case. **Recommended: in scope, this topic's M1/M2.**
- **`query_alert_history`** is a **sample-bearing timeline**; its value is the individual events.
  Multi-Thing history genuinely runs into the cap (N Things × per-Thing samples), the demand is
  lower (operators rarely want raw multi-asset timelines at once — when they want comparison, that's
  summary's job), and the per-Thing-sub-cap design is murkier. **Recommended: defer** — keep
  single-Thing for now; revisit only if a concrete need appears. (Flagged as a review ask, §6.)

`acknowledge_alerts` stays single-Thing (§0.5, §5).

> This is a **recommendation**, not a mandate. If reviewers prefer to also do history (with a
> per-Thing sub-cap), that is a legitimate scope decision — but it should be a deliberate one.

### 2.2 Schema: `thingNames` array replaces `thingName`

Clean break (no compat alias — §0.1). For `query_alert_summary`:

- Remove the scalar `thingName`; add **`thingNames`**: `{ "type": "array", "items": {"type":
  "string"}, "minItems": 1, "description": "One or more canonical ThingWorx names …" }`, and set
  `required = {"thingNames"}`.
- All existing filter args (`ackState`, `propertyName`, `alertName`, `alertType`, `priorityMin/Max`,
  `sort`, `limit`, `advancedQuery`) keep their meaning and apply to **every** Thing in the call.
- The description must state: each name must be canonical; non-canonical names are reported
  per-Thing (not a whole-call failure — §2.3); the result is a **per-Thing rollup** (§2.4).
- **Missing / empty array:** absent `thingNames`, null, or `[]` → whole-call
  **`THINGNAME_VALUE_REQUIRED`** with **`parameterName: thingNames`** (same envelope family as §7.2
  scalar blank). Do **not** treat as partial success.
- **Upper bound:** **`maxThingNamesPerCall = 25`** (executor-enforced). Over limit → whole-call structured
  error (`THING_NAMES_LIMIT_EXCEEDED`, caller mistake — split the fleet or scope tighter). The bound is
  justified after the egress contract in §2.8: worst-case rollups for 25 Things must pass
  `ToolResultEgressGateway.compactForLlmAppend` without **`byThing[]` sampling** (M1 offline test). Q2
  evidence requires **14** Things visible — that fleet size is the design proof point, not the max-N
  ceiling.

The single-Thing case is **`thingNames` length 1** — the model always passes an array. **Output shape
branches on length** (§2.4): N=1 preserves today's flat INFOTABLE path; N≥2 uses the rollup envelope.

### 2.3 Executor: bounded fan-out with per-Thing identity gating

In `AlertToolsExecutor`:

- Parse `thingNames` (array). Enforce **`maxThingNamesPerCall = 25`** (§2.2) before any platform work.
- Gate **each** name through a **list variant** of `ScalarThingnamePreflight` (new
  `gateApplicationThings(...)` returning per-name outcomes). Canonical names proceed; non-canonical
  names are collected into **`identityErrors[]`** (each entry reuses the scalar
  **`IDENTITY_RESOLUTION_REQUIRED`** envelope fields + optional **`recoveryHint`**) **without failing
  the whole call when at least one name resolves**.
- **Zero resolved Things:** if **every** supplied name fails identity preflight, return a **whole-call
  error** — **`status: error`**, **`code: IDENTITY_RESOLUTION_REQUIRED`**, **`parameterName:
  thingNames`**, plus **`identityErrors[]`** listing each failure. Do **not** return vacuous
  `status: success` with an empty `byThing[]` (review-0 B4 / C4).
- For each canonical name, invoke `QueryAlertSummaryForThing` inside a **per-Thing try/catch**. A
  platform failure for one Thing must **not** sink the comparison: record a **`byThing[]` error entry**
  (`status: error`, `code`, `message` — normalized like today's executor errors) and continue with the
  remaining Things (review-0 B3 / C4).
- **All resolved, all sub-calls fail:** if at least one name passed identity but **every**
  `QueryAlertSummaryForThing` sub-call errors, return whole-call **`status: error`** with
  **`code: QUERY_ALERT_SUMMARY_ERROR`** (or the dominant platform code) and per-Thing error detail in
  **`byThing[]`** / summary fields — same class of outcome as today's single-Thing executor failure.
- Prefer a native multi-Thing platform service **iff** `AlertFunctions` is verified to expose one;
  otherwise the server-side per-Thing loop is the contract (§0.6: none found in-repo).
- Preserve the existing per-call filter semantics (`ackState`, sort, filters) for each sub-query.

**Branch on array length after gating:**

| `thingNames.length` | Path |
|---------------------|------|
| **1** (and identity passes) | **Existing** `formatBuiltinInfotableResult` → `INFOTABLE` / `INFOTABLE_LARGE` (preserves table wire, cohort merge input, presentation titles — review-0 B1 / C2). |
| **≥ 2** | New rollup builder → `resultKind: ALERT_SUMMARY_MULTI` (§2.4). |

### 2.4 Result shape — N=1 flat vs N≥2 per-Thing rollup

#### N=1 — preserve today's INFOTABLE envelope

When exactly one canonical Thing resolves, the executor uses the **unchanged** single-Thing code path:
`QueryAlertSummaryForThing` → `formatBuiltinInfotableResult`. The model still **calls** with
`thingNames: ["…"]` but **receives** the same `INFOTABLE` / `INFOTABLE_LARGE` JSON as today (`rows` /
`sampleRows`, `thingName` extra, table/chart wire unchanged).

`PlaybookAlertSummaryRows` continues to parse the flat shape for N=1 playbook steps.

#### N≥2 — `ALERT_SUMMARY_MULTI` rollup envelope

Return a **per-Thing aggregate array**, not a flat concatenation of rows:

```json
{
  "status": "success",
  "resultKind": "ALERT_SUMMARY_MULTI",
  "completeness": "complete",
  "thingsRequested": 14,
  "thingsSucceeded": 13,
  "thingsFailedIdentity": 1,
  "thingsFailedService": 0,
  "byThing": [
    {
      "thingName": "SE.CellFab…StackingRobot-02",
      "status": "success",
      "totalAlerts": 7,
      "unackedCount": 3,
      "byPriority": { "high": 2, "medium": 4, "low": 1 },
      "topAlerts": [
        { "alertName": "…", "sourceProperty": "…", "priority": 900, "timestamp": "…" }
      ],
      "rowCount": 7,
      "cacheId": "…"
    },
    {
      "thingName": "germany",
      "status": "error",
      "code": "IDENTITY_RESOLUTION_REQUIRED",
      "message": "…",
      "recoveryHint": { "tool": "resolve_thing", "argument": "text" }
    }
  ],
  "identityErrors": [
    { "thingName": "germany", "code": "IDENTITY_RESOLUTION_REQUIRED", "recoveryHint": "…" }
  ],
  "extras": { "ackState": "all", "limitApplied": 100 }
}
```

**Field rules (settled per review-0):**

| Field | Rule |
|-------|------|
| **`completeness`** | `"complete"` when `thingsFailedIdentity + thingsFailedService == 0`; `"partial"` otherwise. Downstream routing/skills MUST treat `"partial"` as an incomplete comparison until gaps are resolved. |
| **`byPriority`** | Canonical breakdown (high / medium / low or platform priority bands). No separate **`byProperty`** map in M1 — use **`topAlerts[].sourceProperty`**. |
| **`topAlerts`** | Cap **≤ 3** entries per Thing (headroom under §2.8 egress char budget; reviewers accepted ≤5 — executor uses **3** for worst-case sizing). **Order:** priority **desc**, then timestamp **desc**, then **`alertName`** asc (stable tie-break). |
| **`cacheId`** | Present on a success entry only when that Thing's raw row count exceeds the per-Thing inline sample threshold (reuse conversation-cache machinery for full drill-down). |
| **`identityErrors[]`** | Parallel list for model-facing identity failures (duplicate of identity-failure entries also present in `byThing[]` is acceptable for clarity). |
| **`byThing[]` error entries** | Platform sub-call failures use `status: error` + normalized `code` / `message` on the entry; identity failures may appear here **and** in `identityErrors[]`. |

**Tabular / chart wire (N≥2):** `ALERT_SUMMARY_MULTI` does **not** auto-qualify for
`TabularChartRoundHooks` in M1. Frame as **deferred UX**, not foreclosed: `byThing[]` is already
tabular (one row per Thing: name, totals, unacked, high-priority count) and is shaped for a future
synthetic comparison-table projection on Mashup cards (review-0 ask 7).

**Why this addresses the 20-row INFOTABLE cap:** the N≥2 path never concatenates alert rows into one
flat `rows[]` list. Counts plus ≤3 `topAlerts` per Thing keep inline evidence bounded; full rows stay
behind per-Thing `cacheId` when needed.

### 2.5 Migration of model-facing prose, skills, playbooks

- **Routing guide** (`llm_tool_routing_guide.txt`, "Alert comparison" section): rewrite to say
  `query_alert_summary` accepts **multiple** `thingNames` in one call and returns a per-Thing
  rollup — remove the "requires `thingName` per Thing / fan-out loop" guidance. Update
  `AlertPromptDefaults.java` similarly.
- **Skills** (`region_health`, `asset_pair_health` SKILL.md): replace the "for each returned Thing,
  call `query_alert_summary`" loop with a **single** call passing the resolved Thing set, and adjust
  the grouping step to consume `byThing[]`.
- **Playbooks** (`cross_region_health`, `cross_asset_pair_health` + fixtures): the alert step now
  passes the Thing list once; update the step + any downstream row-consuming steps + JUnit fixtures.

### 2.6 Contracts

Update `CONTRACTS/TAXONOMY_RESOLVER.md` §7.2 for **`query_alert_summary` only**:

- Input: **`thingNames`** array (required); blank / missing / `[]` → **`THINGNAME_VALUE_REQUIRED`**.
- Identity: list gate with partial success; **`identityErrors[]`**; zero resolves → whole-call
  **`IDENTITY_RESOLUTION_REQUIRED`** (§2.3).
- Output: document N=1 INFOTABLE path vs N≥2 **`ALERT_SUMMARY_MULTI`** envelope, **`completeness`**,
  per-Thing service errors, and **`topAlerts`** ordering (§2.4).

`query_alert_history` and `acknowledge_alerts` **remain scalar `thingName`** in §7.2.

Add a version-history entry and bump **`CONTRACTS/CONTRACT_VERSION.md` in the same commit as M1 code**
(review-0 B2 / C3 — contract coupling is not deferrable to M2).

### 2.7 Downstream consumers of the new result shape

| Consumer | N=1 (flat INFOTABLE) | N≥2 (`ALERT_SUMMARY_MULTI`) |
|----------|----------------------|----------------------------|
| **`PlaybookAlertSummaryRows`** | Unchanged (`rows` / `sampleRows`) | **M1:** parse `byThing[]` success entries (`topAlerts` or drill via `cacheId`) |
| **`TabularChartRoundHooks`** | Unchanged | No auto table/chart wire in M1 (deferred projection — §2.4) |
| **`LlmToolResultCohortMerger`** | Unchanged safety net for residual fan-out | No merge on rollup bodies; unchanged class-level logic |
| **`ParlerTableWirePresentationTitles`** | Unchanged | No table wire for rollup (deferred UX) |
| **Wire / golden tests** | Existing assertions stand | **M1:** add rollup + egress tests |

**Rollup derivation:** each sub-call returns an InfoTable from `QueryAlertSummaryForThing`. The executor
computes counts and **`topAlerts`** (§2.4 ordering) from those rows, then optionally caches the full
InfoTable behind per-Thing **`cacheId`**.

### 2.8 Egress-safe contract (review-0 C1 — mandatory in M1)

`ToolResultEgressGateway.compactForLlmAppend` samples most JSON arrays at **`ARRAY_SAMPLE_LIMIT`
(20)** and, when serialized size exceeds **`LLM_EVIDENCE_CHARS_SOFT_CAP` (8192)**, may sample again at
**5** entries. Only `columns`, `dataShape`, and `fieldDefinitions` are exempt today. A naïve
`byThing[]` rollup **would be truncated** at 20 Things — contradicting §2.4.

**M1 egress fix (settled):** when **`resultKind` is `ALERT_SUMMARY_MULTI`**, treat **`byThing`** and
**`identityErrors`** as **non-samplable evidence arrays** (same class as schema arrays — extend
`isSchemaOrMetadataArrayField` or an equivalent result-kind-aware branch). Nested **`topAlerts`**
remain executor-capped at **3** (under the generic sample limit). Add **`completeness`**,
**`thingsRequested`**, **`thingsSucceeded`**, and related counters to egress **`PRIORITY_FIELDS`**.

**Offline proof (M1 gate):** a JUnit test builds a worst-case **14-Thing** rollup fixture (Q2 fleet
size, max-length plausible `thingName`s, 3 `topAlerts` each, partial `identityErrors`) and asserts
`compactForLlmAppend("query_alert_summary", …)` returns **all 14** success `thingName` values in
`byThing[]` with **`completeness`** preserved — no `_egress.reducedFields.byThing` sampling marker.

**Char budget note:** with **`byThing` / `identityErrors` preserved** and **`topAlerts` capped at 3**,
estimated worst-case JSON for 25 Things stays under the 8192 soft cap; the 14-Thing Q2 proof test is
the authoritative check. If the test fails during M1 implementation, reduce **`maxThingNamesPerCall`**
or tighten rollup fields — do not ship with silent egress truncation.

---

## 3. Milestones (sequenced; each is a reviewable increment)

Redrawing per review-0 B2 / C3 (**M1 = green offline gate + contracts; M2 = prose/sample data only**):

- **M1 — wire change + engine + contracts + test-coupled consumers.** `thingNames[]` schema;
  `gateApplicationThings`; executor branches (N=1 flat, N≥2 rollup, §2.3 failure semantics);
  **`ToolResultEgressGateway`** preservation for `ALERT_SUMMARY_MULTI`; **`CONTRACTS/TAXONOMY_RESOLVER.md`**
  §7.2 + **`CONTRACTS/CONTRACT_VERSION.md`** bump **in the same commit**; **`PlaybookAlertSummaryRows`**
  dual-shape support; golden schema registry; alert executor / egress / wire JUnit; **`./gradlew test
  -PuseLocalTwxLib=true` green**. M1 is independently mergeable on the offline gate.
- **M2 — model-facing prose and sample data.** Routing guide + `AlertPromptDefaults`; `region_health` /
  `asset_pair_health` skills; cross-region / cross-asset playbooks + fixtures; `docs/agent/all-tools.md`
  re-snapshot; `docs/operations/alert-solution.md` §5 / §8 supersession note. No further wire changes.
- **M3 (deferred) — multi-Thing `query_alert_history`** unless explicitly pulled in later.

Do not bump **extension** `revisionVersion`/`packageVersion` until the User orders a release cut.

---

## 4. Acceptance

### 4.1 Offline JUnit (the gate — implementor-owned)

`cd parler-agent && ./gradlew test -PuseLocalTwxLib=true --console=plain` must be green, including
new tests for: `thingNames` parse + `minItems`/upper-bound enforcement; per-Thing identity gating
with partial success (`identityErrors[]` while other Things succeed); the per-Thing rollup shape and
its per-Thing sample cap; the single-Thing (N=1) path; the updated golden schema registry; and the
migrated playbook fixtures. This gate is what blocks merge.

### 4.2 Live measurement (**User-owned — left to the User**)

> The implementor does **not** run these. Specify them here; the User executes and provides results.

Re-run the **Q2 scenario** from `tool-schema-admission-control` §4.3 — host-context
`…AssetMonitoring.ContainedCardsAndMapParler_MU`, prompt *"compare the health status of assets
between USA and Germany"* — in `off` admission mode (so the only variable is the alert tool), and
record from `parler-collect-live` telemetry, **before vs after** this change:

- **alert tool-call count** in the turn (target: from ~14 single-Thing calls → **1**),
- `agentIterations` / `toolCallCount` / `turnWallMs`,
- whether `OVERHEAD_EXCEEDS_CAP` occurs (target: the off-mode failure no longer reproduces, because
  history no longer grows by ~14 result pairs),
- final-answer **parity**: the per-region health comparison is at least as complete and correct as
  the fan-out answer, with every Thing represented (verify the rollup didn't starve any asset).
- **Partial-failure turns** (review-0 secondary): one unresolved Thing name **and** one platform
  sub-call error in the same multi-Thing call — confirm **`completeness: partial`**, surviving
  **`byThing[]`** rollups are usable, and the model does not answer as if the comparison were complete.

Target reading (evidence, not a merge gate): one multi-Thing call replaces the fan-out, history
growth drops, and the Q2 turn that failed under fan-out now completes — independently of the
`lazy`/`narrow` tool-schema levers.

---

## 5. Out of scope

- **Multi-Thing `acknowledge_alerts`** — mutating/safety-sensitive; its own topic.
- **`query_alert_history` multi-Thing** unless reviewers pull it in (§2.1, M3).
- **Tool-schema admission** (which tools are advertised) — that is
  `tool-schema-admission-control`; this topic does not touch it.
- **A new fourth "fleet" tool** — this topic extends the existing summary tool, it does not add a
  tool (per the `alert-solution.md` §8 constraint, now satisfied by §0.4's evidence).
- **Backward-compat alias** keeping scalar `thingName` alongside `thingNames` — explicitly not done
  (§0.1).

---

## 6. Decisions log (review-0 → review-1)

| # | Topic | Decision (review-1) |
|---|-------|---------------------|
| 1 | Scope | **Summary-only.** History (M3) and `acknowledge_alerts` out of scope. |
| 2 | Rollup fields | **`totalAlerts`**, **`unackedCount`**, **`byPriority`**, **`rowCount`**, **`topAlerts[]`** (≤**3**, ordered §2.4). No **`byProperty`** map in M1. |
| 3 | N=1 shape | **Preserve flat INFOTABLE / INFOTABLE_LARGE.** Rollup only for **N≥2**. Input always `thingNames[]`. |
| 4 | Upper bound | **`maxThingNamesPerCall = 25`**; over-limit → whole-call **`THING_NAMES_LIMIT_EXCEEDED`**. Egress proof at **14** Things (Q2). |
| 5 | Partial identity | **Partial success** + **`identityErrors[]`** when ≥1 resolves; **whole-call error** when **zero** resolve (§2.3). |
| 6 | Native service | **Server-side per-Thing loop** (none in-repo). |
| 7 | Table wire (N≥2) | **Deferred UX**; shape `byThing[]` for future projection. |
| 8 | Cohort merge | **Unchanged**; N=1 flat path keeps safety net. |
| — | Service errors (B3) | Per-Thing **`byThing[]` error entries**; whole-call error only when all sub-calls fail (§2.3). |
| — | Egress (C1) | **`byThing` / `identityErrors` non-samplable** for `ALERT_SUMMARY_MULTI` + offline 14-Thing test (§2.8). |
| — | M1 / M2 split (B2) | **Contracts + test-coupled code in M1**; prose/skills/playbooks in **M2** (§3). |

**Remaining for review-1:** confirm the above resolves B1–B4 / C1–C4 and record **continue
(implementation authorized)** or redirect. No open design forks unless reviewers identify a gap.

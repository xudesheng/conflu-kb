# Tabular insight — agent tool JSON (`tabulate_cached_result` / `summarize_cached_result`)

**Contract bundle:** [`CONTRACT_VERSION.md`](./CONTRACT_VERSION.md).  
**Scope:** Normative **common root and error shell + `insightEnvelope`** for the **Parler ThingWorx Java agent** built-in tools **`tabulate_cached_result`** and **`summarize_cached_result`** (shared success-path fields, error payload fields, `insightEnvelope` schema revision **1**, and LARGE-branch alignment principle). **Tool-specific** argument schemas, `resultKind` enumerations, per-tool success-only fields, and full error **code** catalogs remain in [`docs/agent/cached_tabular_tools.md`](../docs/agent/cached_tabular_tools.md) as implementation-side reference. This contract describes **agent ↔ LLM tool-call surfaces**, not the **`parler-ui`** streaming reducer wire (see [`UI_CLIENT_PROTOCOL.md`](./UI_CLIENT_PROTOCOL.md) / [`API_CONTRACT.md`](./API_CONTRACT.md)). UI **`insightEnvelope`** field semantics and the D7 gate live in [`UI_CLIENT_PROTOCOL.md`](./UI_CLIENT_PROTOCOL.md) **§View — `insightEnvelopeLoose` (D7)**.

**Implementation-side reference (outside contract bundle):** [`docs/agent/cached_tabular_tools.md`](../docs/agent/cached_tabular_tools.md), [`docs/agent/tabular_insight_envelope.md`](../docs/agent/tabular_insight_envelope.md). **Golden / harness fixtures:** [`docs/agent/cached_tabular_golden.md`](../docs/agent/cached_tabular_golden.md).

All JSON is UTF-8.

---

## 1. Error payload (both tools)

When the tool returns an error string to the LLM pipeline, it MUST be a JSON object with:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `status` | string | yes | Literal **`error`**. |
| `code` | string | yes | Machine code (e.g. `INVALID_PARAMETERS`, `CACHE_MISS`, `LIMIT_OUT_OF_RANGE`, `TABLE_TOO_LARGE_FOR_TRANSFORM`, `PROTECTED_TABULAR_COLUMN_BLOCKED`, …). Full sets are tool-specific; see **`docs/agent/cached_tabular_tools.md`**. |
| `message` | string | yes | Human-readable detail (may be empty string). |

**P2 mirror lifecycle (normative, both tools):** When **`code`** is **`CACHE_MISS`**, if the conversation-scoped P2 **`TOKEN`** mirror (same keying described under **`sourceCacheId`** in **§2**) currently equals the **`cacheId`** that **`lookupCachedInfotable`** attempted (after **`TOKEN`** resolution when applicable), implementations MUST remove that mirror entry.

---

## 2. Success — common root fields

On success, the root object MUST include:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `status` | string | yes | Literal **`success`**. |
| `sourceCacheId` | string | yes | The **effective** cache id used for `lookupCachedInfotable` (after P2 sentinel resolution when the request used **`__PARLER_LAST_QUALIFYING_TABULAR_CACHE__`** — **per-turn** state first, then **conversation-scoped** last qualifying `cacheId` in the agent JVM when the turn snapshot is empty). The conversation-scoped mirror uses the wire **`conversation_id`** for persistent threads; for **`__single_turn__`** paths with an AlwaysOn **`request_id`**, implementations use a per-request composite key so adhoc turns do not share one map entry. On **`summarize_cached_result`** success, implementations MUST set this mirror to the root **`sourceCacheId`** without updating per-turn **`TabularChartRoundState`** last qualifying **`cacheId`** used for **`build_chart_from_tabular_result`** **`last_invoke`**. MUST be present even when `insightEnvelope` is omitted in a future revision. |
| `resultKind` | string | yes | One of the tool-specific literals documented in **`cached_tabular_tools.md`** (e.g. `CACHED_TABULATE_INLINE`, `CACHED_SUMMARY_EMPTY`, …). |

---

## 3. `insightEnvelope` (current P1 behavior)

On **every** success today for the `resultKind` values listed in **`tabular_insight_envelope.md`**, the root object MUST include **`insightEnvelope`** as an object with:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `schemaVersion` | string | yes | Literal **`1`** for this revision. |
| `sourceCacheId` | string | yes | Same value as root `sourceCacheId`. |
| `rowEstimate` | number | yes | Row count semantics per tool / mode (see **`tabular_insight_envelope.md`**). |
| `columns` | array | yes | Array of `{ "name", "baseType" }` objects; column set semantics per tool (see **`tabular_insight_envelope.md`**). |

Clients SHOULD tolerate a future server omitting `insightEnvelope` while still requiring root `sourceCacheId`.

---

## 4. LARGE branch alignment

When `resultKind` ends with **`_LARGE`**, the payload MUST follow the same structural conventions as other tabular LARGE tool results in this extension (e.g. `cacheId`, `sampleRows`, `hint` / row estimate) as documented in **`docs/agent/AGENT-TAXONOMY.md`** §5.2.1 and **`cached_tabular_tools.md`**.

---

## Changelog (this file)

| Revision | Notes |
|----------|--------|
| 1.0.7 | **§1** error **`code`** union includes **`PROTECTED_TABULAR_COLUMN_BLOCKED`** (PASSWORD-typed columns — **`tabulate_cached_result`** / **`summarize_cached_result`** stage 4; **`docs/agent/protection.md` §4.7). Bundle **`0.1.43`**. |
| 1.0.6 | **§1** P2 mirror: **`CACHE_MISS`** MUST prune mirror when it still pointed at the missed id; **§2** **`summarize_cached_result`** success MUST update mirror from **`sourceCacheId`** without advancing per-turn **`last_invoke`** snapshot (Further Insight **#36**). |
| 1.0.5 | **§2** `sourceCacheId`: document **`__single_turn__` + `request_id`** mirror key vs persistent **`conversation_id`**; **`ClearConversation`** clears mirror (Further Insight **#35**). |
| 1.0.4 | **§2** `sourceCacheId`: P2 **`TOKEN`** resolution order — per-turn then conversation-scoped snapshot (Further Insight **#34**). |
| 1.0.3 | **§2** `sourceCacheId`: clarify **effective** cache id after P2 sentinel resolution (Further Insight **#33**). |
| 1.0.2 | L7 link line: **"Non-normative …"** → **"Implementation-side reference (outside contract bundle)"** — aligns with **§Scope** L4 wording (Claude **#15** P3 polish). |
| 1.0.1 | Clarify **§Scope**: common root / error / `insightEnvelope` / LARGE principle only; tool-specific arguments and code catalogs stay in **`cached_tabular_tools.md`** (Codex / Claude **#14** 附录). |
| 1.0.0 | Initial normative registration (Further Insight P1 surface); bundle **`0.1.3`**. |

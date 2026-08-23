# What goes into the LLM (and how it differs from AGENT-CONTEXT.md)

| Audience | Medium | Sent to the model at runtime? |
|------|------|----------------------|
| **People / Cursor / contributors** | **`./AGENT-CONTEXT.md`** and design docs in the same directory | **No** — repository and IDE reference; not automatically bundled into the prompt |
| **LLM** | **①** **`AgentSettings.systemPrompt`** (and per-call **`Chat` / `ChatAsync`** override of the **first** segment only)<br>**②** Optional **`appendBuiltInToolRoutingGuide`** appendix (`LlmRoutingGuide`)<br>**③** Cached stable suffix — optional repository **`/taxonomies/type-taxonomy.md`** Markdown (when configured and non-empty), **GenericThing** ThingTemplate name catalog, **`GetAlertPrompt`** — from the in-memory **prompt-context snapshot** (populated **lazily on the first LLM submit** via **`buildLlmTurnContext`** when the snapshot is still empty, and on **`RefreshPromptContextCache`**). Structured taxonomy rows are **not** duplicated into this suffix; they come from loaded **`identity-types.json`** for resolver tools and in-process **`TaxonomyRow`** projection. The snapshot also stores **`skillRegistry`** (Service + optional repository metadata). Folded into the **leading** system row per **`docs/agent/system-prompt-cache.md`**. **④** Per-turn system rows: skill catalog Markdown (from **`skillRegistry`**), **`/Skill`** bodies, **`ParlerTimeAnchor`**, host-scope meta; then the user message. | **Yes** — assembled in **`AgentThing.buildLlmTurnContext`** (not a single call to **`resolveConversation`** alone). |
| **LLM** (each provider API call) | **⑤** Ephemeral **UTC current time** snippet | **Yes** — **`AgentLoop`** / **`LlmUtcClockInjector`**: inserted immediately **before** the current user message on the first model call of a turn, or **after** trailing tool-result rows on follow-up rounds; removed in **`finally`** after each **`LlmClient.chat`**. For tools that expose **`calendarPhrase`** / **`relativeDuration`**, pass those fields (server resolves with **`user_timezone`**); for other tools, map relative windows to **`startTime`/`endTime`** (or ISO) when needed. |

## Built-in “tool routing” appendix

- **Resource file (shipped with JAR):** `src/main/resources/com/thingworx/things/agent/llm_tool_routing_guide.txt`  
- **Load logic:** `com.thingworx.things.agent.llm.LlmRoutingGuide`  
- **Concatenation:** `your systemPrompt` + `\n\n---\n` + full appendix (appendix is English bullet points to control tokens).  
- **Disable:** On the **`AgentSettings`** Thing, set **`appendBuiltInToolRoutingGuide`** to **false**.  
- **Maintenance:** Long planning notes stay in **`docs/agent/*.md`** (e.g. **`./entity_search_balance.md`**); to change model behavior, sync **actionable conclusions** into **`llm_tool_routing_guide.txt`** (or into **`systemPrompt`**). **Model tags** conventions are in **`./tags_in_agent_tools.md`**.

### #1 — Cached-tabular routing (generic)

When a **cached tabular** result already contains the columns required for the user’s requested **grouping**, **display labels**, and **metric calculation**, prefer deterministic cached-table tools (**`tabulate_cached_result`**, **`summarize_cached_result`**) on that **`cacheId`** directly. **Do not** call listing or discovery tools merely to re-fetch identifiers or names **already present** in the cached rows.

Use listing or discovery tools when the prompt actually requires the **candidate asset universe**, **assets with no matching records**, **identifier or name resolution**, or **metadata not present** in the cached table.

This rule is **not** prompt-specific: do not hard-code bans such as “never call tool X after tool Y” for a single scenario; apply the principle to any workflow where the cache already carries the fields needed for the analysis.

**Parler charts / numeric history:** The appendix **“Charts and numeric history”** section defines **`query_property_history`** **`startTime`/`endTime`** (preferred, aligned with the query), optional **`calendarPhrase`** / **`relativeDuration`** when the schema supports them, and optional **`requestedTimeRange`** (only when the query uses the platform default window but must still cover the user-described interval). The same natural-time envelope applies to the non-numeric history branch of the same tool. **`query_alert_history`** uses the same natural-time fields where documented. **History overlay:** the same section’s **`build_history_overlay_chart`** bullet defines unified overlay routing for same-window and shifted-window traces — see **`docs/agent/history-overlay-chart.md`**. Review and versioned pair builds: Parler **`../archived/times-impl-review-3.md`** and **[`../../scripts/build-twx-release-pair.mjs`](../../scripts/build-twx-release-pair.mjs)**.

## Tool definitions themselves

**Azure OpenAI**, **OpenAI Chat Completions** (`CHATGPT` provider), and **Anthropic Messages** (`ANTHROPIC`) all receive **`ToolDefinition`** name / description and parameter schemas (built-ins as JSON Schema; Anthropic maps to **`input_schema`**). Division of labor for **`invoke_service`** vs **`list_entities_by_type`** is mainly in **description** and **`llm_tool_routing_guide.txt`**; cross-tool choice still relies on the appendix + **`systemPrompt`**.

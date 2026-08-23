# Host Context Turn State 与系统 ID 直用

状态：设计已达成一致（review-2，Codex + Claude **continue**）；实现进行中。

本文是 `docs/architecture/host-context.md` 的下一阶段 topic。上一阶段已经完成了 Host Context 的注册与渲染机制：

```text
HostScopeJson
  -> key + context
  -> Agent 查找注册模板
  -> 按模板渲染 prompt fragment
  -> 作为当前 turn 的 ephemeral system message 注入 LLM
```

这个机制解决了“宿主 Mashup 如何把当前页面上下文告诉 Agent”的第一步问题。现在 workshop 中出现了第二步问题：当 widget 嵌入到真实 Mashup 中，并且 conversationId 保持稳定时，每个回答必须能够说明它是在什么 Host Context 下产生的；同时，当 Host Context 变化后，LLM 不应该继续复用旧页面状态下的历史回答。

## 1. 背景

`parler-ui-widget` 会被嵌入到不同的 ThingWorx Mashup 中，例如：

- Asset Monitoring 页面；
- Asset Detail 页面；
- Dashboard 页面；
- 客户自定义的业务页面。

这些页面本来就有自己的状态，例如：

- 当前选择了哪些 asset type；
- 当前选择了哪个 hierarchy node；
- 当前有哪些 filter；
- 当前 detail 页面对应哪个 Thing；
- 当前 tab、time window、可见结果集是什么。

用户在这些页面中会自然地问：

```text
How many assets are here?
How about now?
Show the selected asset's alert history.
What is the current status under this node?
Summarize this page.
```

上一阶段的 Host Context 设计刻意保持简单：Host Context 是当前 turn 的 sideband，不是用户手工输入，不是权限机制，也不是 server-side 自动 scope inject。Agent 只是验证它、渲染它、把它放进当前 LLM turn。

这个方向仍然正确。但在稳定 conversationId 的场景下，还缺少三个能力：

1. **每个回答要能追溯当时的 Host Context。**  
   同一个 conversation 中，用户可能不断改变页面选择。历史中必须能看出每个 user prompt 对应的 Host Context。

2. **Host Context 变化后，要提醒 LLM 不要复用旧答案。**  
   例如用户先问 `how many assets are here?`，然后改变页面筛选，再问 `how about now?`。LLM 应该基于当前 Host Context 重新查证，而不是复述上一轮数字。

3. **Host Context 中的系统 ID 应该被直接使用。**  
   Mashup 传给 Agent 的值通常已经是 ThingWorx 系统中的真实 ID，例如 ThingName、ThingShape 全名、ThingTemplate 全名、hierarchy node id。它们不是用户说出来的自然语言短语，不应该再走自然语言 resolver。

## 2. 目标

本 topic 的目标：

- 每个 user turn 持久化 Host Context snapshot metadata；
- history replay / collection tool 能看到 Host Context key、hash、raw JSON、变化状态；
- LLM 每个 turn 都能知道当前 Host Context 是否相对上一轮发生变化；
- 对 `here`、`now`、`current view`、`selected`、`this page` 这类问题，优先使用当前 Host Context；
- 建立通用原则：Host Context 中已经是系统 ID 的值，工具应该能直接消费，不要再次 resolve；
- 首先补齐 hierarchy node id 的 direct path；
- UI 中用折叠方式显示每个 user prompt 对应的 Host Context，不占用主对话空间；
- user prompt 与展开的 Host Context raw JSON 各提供一个紧凑 copy icon（见 §4.2；**User 裁决纳入本 milestone**，见 §8.3）。

非目标：

- 不恢复旧 v1 的 server-side Host Scope inject；
- 不把 Host Context 做成权限机制；
- 不定义跨所有 Mashup 的统一业务 schema；
- 不要求使用随机 conversationId；
- 不实现复杂 JSON diff、semantic diff、`hashFields`、volatile-field filtering；
- 不在这一 topic 中解决所有 final answer 表述质量问题；
- 不实现更广泛的 copy-toolbar、assistant-row copy、或与本 topic 无关的 UI polish。

## 3. 核心设计

### 3.1 每个 user turn 存 Host Context snapshot metadata

每个 user message row 都应该记录当时的 Host Context snapshot metadata。

建议 Stream/DataShape 中采用一个主 JSON 字段：

```text
hostContextSnapshotJson
```

内容示例：

```json
{
  "schema": "parler-host-context-snapshot-v1",
  "accepted": true,
  "outcome": "ACCEPTED",
  "key": "asset_monitoring.query_scope",
  "hash": "sha256:1f4f...",
  "utf8Bytes": 335,
  "changedFromPreviousUserTurn": true,
  "rawJsonStored": true,
  "rawJson": "{\"key\":\"asset_monitoring.query_scope\",\"context\":{...}}"
}
```

如果未来需要方便查询，也可以额外增加两个轻量字段：

```text
hostContextKey
hostContextHash
```

但 canonical source of truth 仍然是 `hostContextSnapshotJson`。

这样做的原因：

- Host Context 输入已经有 16 KB 上限；
- workshop 和 live-debug 阶段，需要能回溯 raw JSON；
- 但 raw JSON 不必在每个 user row 上重复保存；
- 每个 user row 保存 key/hash/bytes/outcome/changed 等 metadata；
- 只有 `changedFromPreviousUserTurn=true` 的 accepted Host Context row 保存 raw JSON；
- `changedFromPreviousUserTurn=false` 的 row 可省略 raw JSON，只通过相同 hash 指向前一个 raw snapshot；
- 如果 history replay 从中间截断，导致第一条可见 row 是 unchanged 且找不到前一个 raw snapshot，UI 可以显示 metadata，但 raw JSON 展开内容标记为 unavailable。这是可接受的显示缺口。

rejected Host Context 不保存 raw JSON。既然 Host Context 已被 reject，它不会影响 final response；持久化 reject outcome 与 reject detail 足够。

**每个 user row 都写入 snapshot（含 absent / rejected）：** 便于 rehydrate 时 bounded backward lookup 区分「上一条 user row 无 accepted Host Context」与「根本没有上一条 user row」。`changedFromPreviousUserTurn` 仅对 **accepted** 与 **absent** 按 §3.3 表计算；**rejected** row 写入 snapshot 但不驱动 freshness block。

**Persisted shape（`parler-host-context-snapshot-v1`）：**

| Outcome | `accepted` | 字段 |
| --- | --- | --- |
| **ACCEPTED** | `true` | `outcome`, `key`, `hash`, `utf8Bytes`, `changedFromPreviousUserTurn`, `rawJsonStored`, `rawJson`（仅 anchor row） |
| **UNREGISTERED_GENERIC_FALLBACK** | `true` | 同上，加 `genericFallback: true`, `templateFound: false`（可选）, `renderTruncated`（若 fallback 截断） |
| **ABSENT** | `false` | `outcome: "ABSENT"`, `changedFromPreviousUserTurn`（按 §3.3 absent 行） |
| **REJECTED**（`OVERSIZE` … `RENDER_FAILED`） | `false` | `outcome`, `utf8Bytes`（若有 uplink bytes）, `rejectCode`, `rejectDetail`, `changedFromPreviousUserTurn`（metadata；不用于 freshness） |

示例 — **UNREGISTERED_GENERIC_FALLBACK**（parseable key，无注册 template，generic prompt 已插入）：

```json
{
  "schema": "parler-host-context-snapshot-v1",
  "accepted": true,
  "genericFallback": true,
  "templateFound": false,
  "outcome": "UNREGISTERED_GENERIC_FALLBACK",
  "key": "asset_monitoring.query_scope",
  "hash": "sha256:…",
  "utf8Bytes": 184,
  "changedFromPreviousUserTurn": true,
  "rawJsonStored": true,
  "rawJson": "{…}"
}
```

`accepted: true` 表示 generic fallback prompt fragment 已插入；**不得**据此触发 registered-template 侧效应（`requiredTools` / document-scope 等）。判别 registered vs generic 用 `outcome` / `genericFallback`，不是 `accepted`  alone。

示例 — **ABSENT**（上一条也为 absent → unchanged）：

```json
{
  "schema": "parler-host-context-snapshot-v1",
  "accepted": false,
  "outcome": "ABSENT",
  "changedFromPreviousUserTurn": false
}
```

示例 — **REJECTED**（invalid / oversize / schema）：

```json
{
  "schema": "parler-host-context-snapshot-v1",
  "accepted": false,
  "outcome": "MISSING_KEY",
  "utf8Bytes": 42,
  "rejectCode": "missing_or_empty",
  "rejectDetail": "key: missing_or_empty",
  "changedFromPreviousUserTurn": false
}
```

**Collection / live-debug 读取路径（normative intent）：** anchor raw JSON 的存储简化，前提是 **collection 与 live-debug 始终从 AgentMessageStream（`hostContextSnapshotJson`）读取 raw snapshot**，**不得**依赖 UI 已加载、可能被 `historyClearedAt` 或分页截断的 history hydrate。UI history 上的 “raw unavailable” 缺口只影响 UI 展开区；debug 路径必须能恢复 Stream 中的 anchor row。

### 3.2 raw string hash

Host Context hash 直接使用 Agent 收到的 raw string：

```text
sha256(UTF-8 raw Host Context string as received)
```

不做 canonical JSON normalization，不做字段排序，不做 `hashFields`，不做 volatile-field filtering。

原因很简单：Host Context 不是用户手写 JSON。正常路径是 App Developer 在 Mashup 中根据页面状态构建 JavaScript object，然后调用 `JSON.stringify`。字段顺序由 App Developer 的表达式决定，不是随机的。

因此 raw string hash 是最容易解释、最容易实现、最容易 debug 的方案：

- 它代表 Agent 实际收到的值；
- collection 保存的也是这个值；
- 用户或开发者可以直接比较两次 raw JSON；
- 不需要引入额外的 canonicalization 规则。

如果某个 Mashup 把 timestamp、nonce、scroll offset 这类不应该影响语义的字段放进 HostScopeJson，导致每轮都 `changed=true`，那是该 Mashup 的 Host Context 设计问题。解决方式应该是从 HostScopeJson 中移除这些字段，或使用不同的 template key，而不是让 Agent 负责复杂 diff。

**可诊断性增强（非 blocker，建议纳入实现）：** 当连续多个 user turn 的 `key` 相同但 hash 每轮都变化，或 hash 变化而 parsed `context` 的 key 集合不变时，`ValidateHostContext` 日志与 collection 输出 **SHOULD** 带上可观测信号（例如 `hostContextHashChangedRepeatedly` 或 “hash changed, context key set unchanged”），便于 workshop 发现 Mashup 误注入 volatile 字段，而无需读 Java 源码。

### 3.3 changed 计算

Agent 在处理当前 user turn 时：

1. 验证 Host Context；
2. 若接受，计算 raw string hash；
3. 取得当前有效历史范围内 **上一条 user row** 的 Host Context hash（见下方 bounded read）；
4. 比较 hash；
5. 得到 `changedFromPreviousUserTurn`。

**Bounded read（hot path MUST NOT 随会话长度线性扫描）：**

- **Live JVM 会话（首选）：** 在 conversation 内存状态中携带 `lastAcceptedHostContextHashOrNull`（及可选 `lastAcceptedHostContextKeyOrNull`），每 append accepted user row 后更新；下一 turn O(1) 比较。
- **Rehydrate / 冷启动：** 从 Stream 做 **有界** 查询——在 `historyClearedAt` 之后、当前 row 之前，取 **最近一条** 带 `hostContextSnapshotJson` 的 user row（单条 backward lookup，利用 Stream 时间序 + conversation filter，**不得** full-history scan）。与 `AgentMessageStreamHistoryExporter` 的 clear 边界语义一致。
- Rehydrate 后 MUST 用读到的 hash 回填 JVM carry，后续 turn 恢复 O(1) 路径。

规则：

| 当前 Host Context | 上一条 user row | changed |
| --- | --- | --- |
| accepted | 无 accepted Host Context | `true` |
| accepted | hash 相同 | `false` |
| accepted | hash 不同 | `true` |
| absent | 上一条也 absent | `false` |
| absent | 上一条 accepted | `true` |
| rejected | 记录 outcome/reject detail；不保存 raw JSON；不应用于 freshness 强指令 |

比较范围应该尊重 `historyClearedAt`。也就是说，只比较当前有效历史范围内的上一条 user row。

### 3.4 Freshness prompt

当 Host Context 被接受时，Agent 在当前 turn 中注入固定 freshness block。这个 block 应该位于 Host Context rendered fragment 前面，位置固定，便于后续排查。

示例：

```text
Host page context freshness:
- The host page context below is the current page state for this user turn.
- For "here", "now", "current view", "selected", "this page", and similar prompts, prefer this current host page context over prior assistant answers.
- If the host page context changed since the previous user turn, re-query evidence before answering counts, lists, summaries, or comparisons.
- Explicit user text still wins over host page context.
- Host page context changed since the previous user turn.
```

如果未变化，最后一行改为：

```text
- Host page context is unchanged since the previous user turn.
```

注意：这仍然是 LLM steering，不是硬约束。它不能 100% 保证模型一定重新查，但它能显著降低 `how about now?` 直接复用旧答案的概率。

**Prompt-cache / prefix 稳定性：** freshness block 的前四行固定文案 **MUST** 保持不变且排在 variable 行之前；仅最后一行在 changed / unchanged 之间切换。后续编辑不得把 variable 行插入 stable prefix 中间，以便 provider prefix cache 命中 stable 前缀。

后续如果需要更强约束，可以考虑在 rehydrated history 中给旧 assistant rows 增加标记，例如“this answer was computed under a previous host context”。但这不是第一阶段必须项。

## 4. History 与 UI 显示

### 4.1 History wire shape

history export / UI hydrate 时，user row 应包含 nested `hostContext` 字段：

```json
{
  "role": "user",
  "content": "how about now?",
  "hostContext": {
    "schema": "parler-host-context-snapshot-v1",
    "accepted": true,
    "outcome": "ACCEPTED",
    "key": "asset_monitoring.query_scope",
    "hash": "sha256:1f4f...",
    "utf8Bytes": 335,
    "changedFromPreviousUserTurn": true,
    "rawJsonStored": true,
    "rawJson": "{...}"
  }
}
```

Stream 内部可以存 `hostContextSnapshotJson`，history wire 再展开成 nested `hostContext`。不要在不同层面发明多套字段名。

当 `changedFromPreviousUserTurn=false` 时，history row 可以省略 `rawJson`：

```json
{
  "role": "user",
  "content": "how about now?",
  "hostContext": {
    "schema": "parler-host-context-snapshot-v1",
    "accepted": true,
    "outcome": "ACCEPTED",
    "key": "asset_monitoring.query_scope",
    "hash": "sha256:1f4f...",
    "utf8Bytes": 335,
    "changedFromPreviousUserTurn": false,
    "rawJsonStored": false
  }
}
```

UI 可以向前查找同一 conversation 中最近一个相同 hash 且 `rawJsonStored=true` 的 row 来展开 raw JSON。如果历史从中间开始，找不到 anchor row，则显示 `raw JSON unavailable in loaded history`。

### 4.2 UI row-level 折叠显示

UI 端倾向简单实现：只要某个 user row 有 Host Context snapshot，就在该 user prompt 顶部显示一个折叠条。

默认折叠：

```text
Host context: asset_monitoring.query_scope · changed · 335 bytes
```

或者：

```text
Host context: asset_monitoring.query_scope · unchanged · 335 bytes
```

点击展开后显示 raw JSON，并在左侧提供一个极小 copy icon：

```json
{
  "key": "asset_monitoring.query_scope",
  "context": {
    "page": "Asset Monitoring",
    "queryParameters": {
      "selectedEntityTypes": [
        {
          "EntityType": "ThingShape",
          "EntityName": "PTCTDD.CellfabDataset.Contacting_TS"
        }
      ]
    }
  }
}
```

设计原则：

- Host Context 属于 user turn，不属于 assistant final response；
- 不要把 Host Context JSON 插进 assistant 自然语言回答的开头；
- 默认折叠，避免浪费对话空间；
- 每个 user prompt 都显示折叠入口，简单、一致、可 debug；
- user prompt 文本左侧提供极小 copy icon（复制当前 user message 纯文本）；
- Host Context raw JSON 展开区左侧提供极小 copy icon（复制展开区显示的 raw JSON 字符串；当 UI 显示 `raw unavailable` 时 **不显示** copy icon）；
- copy icon 不占用额外纵向空间，样式可参考 Codex / Claude 的紧凑复制按钮；
- print/export 时可以展开或保留折叠状态，第一阶段不强求。

**Copy icon 范围（User 裁决，2026-06-23）：** 仅 user prompt 与 expanded raw Host Context JSON；**不**扩展至 assistant 回答、tool activity、或全局 copy 工具栏。此范围 supersedes review-0 的 defer 建议及 review-1 中的 open question（`host-context-turn-state-review-1.md` **`## User Ruling`**）。

## 5. Host Context 中的系统 ID

### 5.1 通用原则

Host Context 来自 Mashup 页面，不是用户自然语言输入。Mashup 传来的很多值已经是 ThingWorx 系统中的真实 ID：

```json
{
  "selectedEntityTypes": [
    {
      "EntityType": "ThingShape",
      "EntityName": "PTCTDD.CellfabDataset.Contacting_TS"
    }
  ],
  "networkName": "PTCTDD.Cellfab.AssociationNetwork_NW",
  "selectedNetworkNode": "SE.CellFab.Model.Region.Germany",
  "thingName": "SE.CellFab.Model.Workunit.ORD-Contacting-01"
}
```

这些值应该直接使用。

通用规则：

```text
Host Context system id -> use directly
User text / display phrase -> resolve
```

例如：

| 场景 | 用户自然语言路径 | Host Context 系统 ID 路径 |
| --- | --- | --- |
| Thing | 用户说 `ORD Contacting 01` -> `resolve_thing` | Host Context 有 canonical `thingName` -> 直接传 `thingName` |
| Asset type | 用户说 `Contacting` -> `resolve_asset_type` | Host Context 有 `EntityType` / `EntityName` -> 直接用 |
| Hierarchy node | 用户说 `Germany` -> `hierarchyNodeName` -> `ResolveNetworkID` | Host Context 有 node id -> 直接用 `hierarchyNodeId` |
| Mashup service 参数 | LLM 从用户文本构造参数 | Host Context 直接提供 fenced JSON 参数块 |

这不是权限绕过。所有工具调用仍然运行在当前用户的 ThingWorx visibility / permission / policy / HITL 约束下。这里只是避免把已经 resolved 的系统 ID 再拿去做自然语言解析。

### 5.2 不做万能 `id` 抽象

不要设计一个通用字段，例如：

```json
{
  "id": "..."
}
```

原因是不同系统 ID 的消费路径不同：

- ThingName 传给 `thingName`；
- ThingShape / ThingTemplate 传给 `EntityType` / `EntityName` 或已有 parent 参数；
- hierarchy node id 用于 `GetAssetList`；
- service 参数块可能直接作为 `invoke_service.parameters`。

因此更好的通用方案不是“一个万能 id 字段”，而是：

```text
Host Context 保留真实系统 ID
Template 明确告诉 LLM 哪个字段应该传给哪个工具参数
Built-in tools 补齐必要的 direct-id 参数
```

### 5.3 Asset type direct path 已经存在

对于 Asset Monitoring 中的 `selectedEntityTypes`：

```json
{
  "EntityType": "ThingShape",
  "EntityName": "PTCTDD.CellfabDataset.Contacting_TS"
}
```

这已经可以直接映射到 `query_entities_by_taxonomy` 的 `EntityType` / `EntityName`，不需要新增 `thingShape` / `thingTemplate` 平行参数。

### 5.4 当前最紧迫缺口：hierarchy node id

当前 `query_entities` / `query_entities_by_taxonomy` 支持：

```json
{
  "hierarchyNodeName": "Germany"
}
```

内部语义是：

```text
ResolveNetworkID("Germany") -> GetAssetList(resolvedId) -> intersect with entity query result
```

但 Asset Monitoring 页面自然提供的是：

```json
{
  "selectedNetworkNode": "SE.CellFab.Model.Region.Germany"
}
```

这已经是 node id，不应该再传给 `hierarchyNodeName` 去做 `ResolveNetworkID`。

因此需要给 `query_entities` / `query_entities_by_taxonomy` 增加：

```json
{
  "hierarchyNodeId": "SE.CellFab.Model.Region.Germany"
}
```

语义：

1. 如果 `intersectThingNames` 非空，直接使用；
2. 否则如果 `hierarchyNodeId` 非空，直接调用 `GetAssetList(hierarchyNodeId)`，得到 node 下的 ThingName 集合；
3. 将该集合与 entity query 结果求交；
4. 否则如果 `hierarchyNodeName` 非空，走原有 `ResolveNetworkID -> GetAssetList -> intersect`；
5. 否则不做 hierarchy scope。

优先级：

```text
intersectThingNames > hierarchyNodeId > hierarchyNodeName > unscoped
```

如果同时提供 `hierarchyNodeId` 和 `hierarchyNodeName`，使用 `hierarchyNodeId`。

**Failure / empty 语义（与 `API_CONTRACT.md` intersect augment 族一致，MUST）：**

direct-id 路径 **不** 调用 `ResolveNetworkID`，因此 **不** 产生 `HIERARCHY_RESOLVE_*`。其余 outcome **MUST** 复用现有 envelope，且 **MUST NOT** 在失败或空结果时 silent fallback 到 `hierarchyNodeName` 或未 scoped 的全局 listing（与 §460 “no hidden fallback” 一致）：

| 条件 | Tool `status` | `code` |
| --- | --- | --- |
| `GetAssetList(hierarchyNodeId)` 抛错 / 服务失败 | `error` | `HIERARCHY_ASSET_LIST_FAILED` |
| `GetAssetList` 成功但无可用 Thing `name` 行 | `error` | `HIERARCHY_SCOPED_EMPTY` |
| 展开后 name 数超过 intersect cap | `error` | `INTERSECT_LIST_TOO_LARGE` |

当 `hierarchyNodeId` 非空且 augment 已选中该路径时，即使同时提供了 `hierarchyNodeName`，失败 **也 MUST NOT** 回退到 name-resolve 路径。

**`query_entities_by_taxonomy`  parity：** `hierarchyNodeId` 与 `hierarchyNodeName` 一样，在 **taxonomy / identity 解析完成之后** 再对结果集做 ∩（与今日 `hierarchyNodeName` augment 顺序相同）；两工具 MUST 保持同一 intersect 阶段语义。

工具结果中应该标记使用了 direct-id 路径：

```json
{
  "hierarchyScope": {
    "source": "hierarchyNodeId",
    "id": "SE.CellFab.Model.Region.Germany",
    "assetCount": 12
  }
}
```

这是新的 built-in tool 参数和 precedence 变化，因此需要更新 `CONTRACTS/API_CONTRACT.md`、`CONTRACTS/CONTRACT_VERSION.md`，并随 agent 版本发布。

第一阶段不验证 `networkName`。设计假设 Host Context 中的 node id 在当前 ThingWorx hierarchy/network 使用语境内唯一。暂不支持跨 network disambiguation。如果未来客户环境证明 node id 不能唯一定位，再引入带 `networkName` 的 direct path。

### 5.5 Template 更新

`dev_data/scpa_utilization/host-contexts/asset_monitoring.query_scope.json` 应该明确告诉 LLM：

```text
- If the fenced JSON includes selectedNetworkNode, it is a hierarchy node id from the page.
- For query_entities or query_entities_by_taxonomy, pass selectedNetworkNode as hierarchyNodeId.
- Do not pass selectedNetworkNode to hierarchyNodeName.
```

如果 fenced JSON 中包含 `selectedEntityTypes`，模板也可以提示：

```text
- If selectedEntityTypes contains EntityType and EntityName, use those exact values for query_entities_by_taxonomy.
- Do not call resolve_asset_type for selectedEntityTypes from host context.
```

## 6. 涉及改动范围

### 6.1 Agent

可能涉及：

- `HostContextUplink.Decision`
  - 暴露 key、outcome、raw utf8 bytes、reject detail；
  - 支持构建 snapshot。
- `ParlerStreamToRemoteThing` / `Chat` / `ChatAsync`
  - 计算 Host Context snapshot；
  - 查找上一条 user row 的 Host Context hash；
  - 注入 freshness block；
  - append user row 时带上 snapshot。
- `AgentMessageStreamAppender`
  - user row 写入 `hostContextSnapshotJson`。
- `AgentMessageStreamReader` / history export
  - 输出 nested `hostContext`。
- `HierarchyQueryEntitiesIntersectAugment`
  - 支持 `hierarchyNodeId` direct path。
- `BuiltInTools`
  - 为 `query_entities` / `query_entities_by_taxonomy` 增加 `hierarchyNodeId` schema 与描述。

### 6.2 UI

可能涉及：

- history hydrate 读取 user row 的 `hostContext`；
- live streaming 当前 user row 保存 Host Context metadata；
- user bubble 顶部显示折叠条；
- 展开后显示 raw JSON，或在当前 loaded history 找不到 anchor row 时显示 raw unavailable；
- user prompt 与 raw JSON 展开区各一个紧凑 copy icon（§4.2）；
- 样式应紧凑，不与正常 prompt 文本竞争空间。

### 6.3 Contracts / docs

需要同步（**history `hostContext` 是 wire contract，不再 hedging**）：

- `CONTRACTS/API_CONTRACT.md`
  - `query_entities` / `query_entities_by_taxonomy` 增加 `hierarchyNodeId`；
  - 说明 precedence（`intersectThingNames > hierarchyNodeId > hierarchyNodeName > unscoped`）；
  - 说明 direct-id 路径的 `HIERARCHY_ASSET_LIST_FAILED` / `HIERARCHY_SCOPED_EMPTY` / `INTERSECT_LIST_TOO_LARGE` 与 **no fallback** 规则；
  - **history export / `GetConversationHistoryJson` user row** 增加 nested `hostContext`（`parler-host-context-snapshot-v1` schema）；Stream 字段 `hostContextSnapshotJson` 与 wire `hostContext` 的映射关系。
- `CONTRACTS/UI_CLIENT_PROTOCOL.md`
  - `rows[]` 中 **user** row 可选 nested `hostContext`（与 API history 同形）；UI hydrate / live turn 消费规则；raw JSON anchor 向前查找与 “raw unavailable” 显示语义。
- `CONTRACTS/CONTRACT_VERSION.md` — 上述 wire 变更随 agent + widget 版本一并 bump。
- `docs/architecture/host-context.md`
  - 增加指向本文的短链接；
  - 不复制本文全部内容。
- `docs/architecture/hierarchy-network-services.md`
  - 说明 direct node id path 与原 `ResolveNetworkID` path 的区别。
- `docs/agent/collection-tool.md`
  - collection 输出 Host Context snapshot。
- `docs/agent/live-diagnostics.md`
  - 增加 Host Context debug checklist。
- `dev_data/scpa_utilization/host-contexts/asset_monitoring.query_scope.json`
  - 更新 template guidance。
- ParlerGuidance
  - 实施合并后再由熟悉培训材料的人同步章节和截图位置。

## 7. 验证

### 7.1 单元测试

Host Context snapshot：

- accepted context 生成 key/hash/bytes/outcome；
- changed/anchor accepted context 保存 raw JSON；
- unchanged accepted context 可省略 raw JSON；
- absent context after accepted context 标记 changed；
- 相同 raw JSON 标记 unchanged；
- 不同 raw JSON 标记 changed；
- rejected context 记录 outcome/reject detail。

LLM context：

- accepted Host Context 注入 freshness block；
- changed 注入 `changed since previous user turn`；
- unchanged 注入 `unchanged since previous user turn`；
- freshness block 位置固定。

Stream/history：

- user row 写入 `hostContextSnapshotJson` metadata；
- changed/anchor row 写入 raw JSON；
- unchanged row 可省略 raw JSON；
- rejected row 不写 raw JSON；
- assistant/tool row 不重复写 raw Host Context；
- history export 输出 nested `hostContext`。

Hierarchy：

- `hierarchyNodeId` 直接调用 `GetAssetList`；
- `hierarchyNodeId` 与 entity query 结果求交；
- `hierarchyNodeId` 优先于 `hierarchyNodeName`；
- 原有 `hierarchyNodeName` 路径保持不变；
- `GetAssetList` 失败 → `HIERARCHY_ASSET_LIST_FAILED`；
- 零 asset → `HIERARCHY_SCOPED_EMPTY`；
- `hierarchyNodeId` 失败时 **不** fallback 到 `hierarchyNodeName`；
- `query_entities_by_taxonomy`：taxonomy 解析 **之后** 再 ∩，与 name 路径一致。

Collection / debug：

- collection 从 Stream 读取 anchor raw JSON，不依赖 UI truncated history。

UI：

- 每个带 Host Context 的 user row 顶部显示折叠条；
- 默认折叠；
- 展开显示 raw JSON，或在当前 loaded history 无 anchor row 时显示 raw unavailable；
- user prompt 左侧 copy icon 复制 message 文本；
- raw JSON 展开区 copy icon 复制可见 raw JSON；raw unavailable 时不显示 copy icon；
- hydrate 后仍能显示历史 Host Context；
- 当前 turn live 发送后也能显示 Host Context。

### 7.2 Live test

Prompt 1：

```text
how many assets are here?
```

期望：

- user row 显示 Host Context 折叠条；
- tool 基于 Host Context 查询；
- collection 中能看到 raw Host Context。

改变页面筛选后 Prompt 2：

```text
how about now?
```

期望：

- Host Context hash 变化；
- freshness block 表示 changed；
- LLM 重新查询，不直接复用上一轮数字。

选择 hierarchy node 后 Prompt 3：

```text
how many assets are there under the selected node?
```

期望：

- LLM 使用 `hierarchyNodeId`；
- 工具走 direct `GetAssetList(hierarchyNodeId)`；
- 不调用 `ResolveNetworkID("SE.CellFab.Model.Region.Germany")`。

## 8. 已裁决的问题

1. **rejected Host Context 不持久化 raw JSON。**  
   rejected context 不会影响 final response，因此记录 outcome/reject detail 即可。

2. **第一阶段不验证 `networkName`。**  
   `hierarchyNodeId` 假定在当前使用语境中唯一。暂不支持跨 network disambiguation。

3. **copy icon 纳入第一阶段（User 裁决，2026-06-23）。**  
   user prompt 与 expanded raw Host Context JSON 各一个紧凑 copy icon；**不**扩展至 assistant / tool / 全局 toolbar。Supersedes review-0 defer 建议及 review-1 open question（见 review-1 **`## User Ruling (recorded by Codex, 2026-06-23)`**）。

4. **不增加关闭 raw Host Context persistence 的配置项。**  
   即便 Host Context 中包含多个或十多个 ThingName，空间占用仍可接受。第一阶段不做 metadata-only 配置。

5. **raw JSON 只在 changed/anchor row 保存。**  
   每个 user row 保存 metadata；`changedFromPreviousUserTurn=true` 的 accepted row 保存 raw JSON；unchanged row 可省略 raw JSON。history replay 如果从中间截断，导致找不到 raw anchor，UI 显示 raw unavailable，这个缺口可接受；collection **必须**仍从 Stream 读取 anchor。

6. **Collection / live-debug 读 Stream，不读 UI history。**  
   anchor-only 存储模型的 debug 价值，依赖 Stream 为 raw JSON 的 authoritative 来源。

7. **`history` user row `hostContext` 是 normative wire。**  
   实施时同步 `API_CONTRACT.md` + `UI_CLIENT_PROTOCOL.md` + `CONTRACT_VERSION.md`，不再 conditional wording。

8. **`hierarchyNodeId` 失败语义与 no-fallback。**  
   复用 `HIERARCHY_ASSET_LIST_FAILED` / `HIERARCHY_SCOPED_EMPTY`；不得 silent fallback 到 `hierarchyNodeName` 或未 scoped listing。

9. **`changed` 比较 MUST bounded。**  
   JVM carry + rehydrate 时有界 backward lookup；hot path 不得 full-history scan。

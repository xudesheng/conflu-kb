# Start here — working with Parler through Conflu

## What Parler is

Parler is a ThingWorx extension: `parler-agent` (a Java extension providing
the `AgentThing` template, which runs an LLM-backed agent loop inside
ThingWorx) and `parler-ui-widget` (the Mashup chat/visualization widget,
AlwaysOn WebSocket). The agent answers a Mashup user's questions about their
data by selecting tools, gathering platform evidence, and rendering results.
Two of its design principles matter to you as a configurer: **trust by
construction** (chart data never passes through the LLM) and
**human-in-the-loop** (high-impact actions need explicit approval in the UI).

An App Developer customizes Parler **without touching Java**, entirely
through files in a ThingWorx FileRepository the AgentThing points at (its
`configurationRepository`):

```text
<ConfigRepositoryThing>/
├── taxonomies/      asset-types.json, identity-types.json
├── tools/           extended_tools.json          (wrap services as LLM tools)
├── skills/          <id>/SKILL.md                (reusable LLM guidance per task)
├── playbooks/       <id>/playbook.json           (deterministic DAGs run by the engine)
├── policies/        invoke_service.json          (fail-closed allow-lists, HITL gates)
└── host-contexts/   <mashup_key>.json            (render Mashup page state into the prompt)
```

A second FileRepository may hold **document knowledge** (converted manuals),
configured in the AgentThing's `AgentSettings` configuration table
(`documentKnowledgeRepository`, `documentKnowledgeRootPath`, default
`/document-knowledge`).

The AgentThing loads these files at (re)start. `RestartThing` reloads them.

## How to work: the live system is the truth

This digest is advisory. Before acting on anything version- or
state-dependent, look at the target with Conflu:

| Question | Conflu call |
|---|---|
| Which Parler is installed? | `conflu twx extensions list` → `packageVersion` of `parler-agent` and `parler-ui-widget` |
| Which ThingWorx? | `twx.env.describe` |
| Which repository does the AgentThing use, what did it load, does it drift? | `twx.service.invoke` on `Things/<AgentThing>/Services/GetAgentRuntimeSnapshot` (`configurationRepository.thingName`, loaded file hashes, drift checks) |
| AgentSettings (document knowledge etc.)? | `twx.service.invoke` on `GetConfigurationTable` with `{"tableName":"AgentSettings"}` |
| What is in the repository right now? | `twx.filerepo.download` of `<repo>/<path>` (e.g. `taxonomies/asset-types.json`) |
| Does the repository validate? | `twx.service.invoke` on `ValidateAgentConfigurationRepository` |
| What happened on the last turn? | `twx.log.query` with `log_name: "ApplicationLog"` (playbook runner, agent loop, tool router, taxonomy resolver all log there) |
| Does a host-context template load? | `twx.service.invoke` on `ValidateHostContext` |
| Run a turn from code | `twx.service.invoke` on `Chat` (synchronous; returns the result) — `ChatAsync` + `AgentResponseEvent` for asynchronous use |

Three rules:

1. **Dev profile only for anything that writes or runs a turn.** `Chat` has no
   "mutating" prefix, but a turn persists conversation state and may invoke
   HITL-gated tools; `RestartThing` interrupts live conversations. Use a
   Conflu `dev` profile and say what you are about to do.
2. **Show before you upload; confirm before you restart.** Download the
   current file, diff, show the developer, then `twx.filerepo.upload`, then
   `RestartThing`, then re-validate and test one `Chat` turn. Never write to a
   repository the developer has not named.
3. **Say version deviations once.** If the installed version lacks a
   capability, say so in one line and proceed with what exists (or
   recommend the upgrade). Do not repeat it and do not refuse.

## Where answers come from (inside this digest)

| Topic | Read |
|---|---|
| Architecture, agent loop, services, contracts | `parler/docs/agent/AGENT-CONTEXT.md`, `parler/docs/agent/LLM_CONTEXT.md`, `parler/CONTRACTS/` |
| Taxonomies (`asset-types.json`, `identity-types.json`) | `parler/docs/agent/AGENT-TAXONOMY.md`, `parler/CONTRACTS/TAXONOMY_RESOLVER.md`, book ch. 7–8, app. H; golden example `parler/dev_data/scpa_utilization/taxonomies/` |
| Extended tools, wrapping services | `parler/docs/agent/CUSTOMIZED-TOOLS.md`, `parler/docs/agent/configuration-repository.md`, book ch. 13 + 16, app. I; golden example `…/tools/extended_tools.json` |
| Skills | `parler/docs/agent/skill-management.md`, `CUSTOMIZED-SKILLS.md`, book ch. 11 + 15, app. J; golden `…/skills/` |
| Playbooks | `parler/docs/agent/playbook-engine.md`, `playbook-directory-packaging.md`, book ch. 12, app. J + N; golden `…/playbooks/` |
| Policies and HITL | `parler/docs/agent/configuration-repository.md`, book ch. 14, app. I |
| Host context (embed in a Mashup) | `parler/docs/architecture/host-context*.md`, book ch. 19, `API_CONTRACT.md` §hostContext, `UI_CLIENT_PROTOCOL.md` |
| Document knowledge (PDF manuals) | `parler/docs/agent/document-chunk-tools.md`, `parler/docs/operations/pdf-conversion-agent-playbook.md` (no book chapter) |
| Alerts, history charts, all built-in tools | `parler/docs/operations/multi-thing-alert-query.md`, `parler/docs/agent/history-overlay-chart.md`, `parler/docs/agent/all-tools.md`, book ch. 10, app. E |
| Diagnosis with live evidence | `guide/diagnose.md`, `parler/docs/agent/live-diagnostics.md`, `collection-tool.md` |
| Worked, known-good configurations per workshop day | `ParlerGuidance/workshop/day1..day4/`, `parler-workshop/workshop/day1..day4/` |

Concrete rules that bite are collected in `guide/configuration-rules.md`.
The course book is `ParlerGuidance/src/SUMMARY.md` onward.

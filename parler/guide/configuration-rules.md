# Configuration rules that bite

Condensed from Parler's documents and four workshop days of student
failures. Each rule names the file it applies to; the authoritative text is
in `parler/docs/` and `parler/CONTRACTS/`.

## Repository and loading

- The AgentThing **loads** repository files at start; the repository may have
  **changed since**. The most common failure in the workshop: "what ran is not
  what is in the repo now". `GetAgentRuntimeSnapshot` reports loaded-vs-current
  drift per file; after any upload, `RestartThing`, then check again.
- `ValidateAgentConfigurationRepository` returns structured validation
  errors — run it after every deploy, before testing.
- Naming collision to watch: the course book uses the name `AIDocRepository`
  for the *export* repository (ch. 3), while Parler reuses the same name in
  examples for the *document-knowledge* repository. Point
  `documentKnowledgeRepository` at the document-knowledge FileRepository, not
  the export one.

## `tools/extended_tools.json`

- Tool names match `[A-Za-z][A-Za-z0-9_]{0,63}` — no hyphens.
- Each entry: a bounded name, a `whenToUse` the model can act on, the target
  service. Give the model **bounded names, bounded operations, trustworthy
  evidence**; do not make it guess the application.
- **The INFOTABLE-input trap.** Services taking `INFOTABLE` / DataShape
  inputs are fragile for an LLM: a flat shape may be exposed but is
  error-prone to construct; a complex or unsupported shape may be dropped at
  discovery or rejected at invoke time. Prefer a thin **scalar wrapper
  service** on the ThingWorx side and wrap that.
- `hitl` marks tools that need approval; `playbookSafe` marks tools a
  playbook `tool_call` node may use. Keep both honest.
- Utilization reference: the final model-facing catalog is
  `parler/dev_data/scpa_utilization/tools/extended_tools.json` — four tools;
  earlier workshop days teach a seven-underlying-service framing on purpose.
  Read the live file; do not paste a remembered manifest.

## `policies/invoke_service.json`

Fail-closed, allow-only. A service not listed cannot be invoked by the
agent. HITL gates belong here, not in prose.

## `taxonomies/asset-types.json`, `identity-types.json`

Every `entityName` / `entityType` must resolve to a real entity on the
target — check with `twx.entity.inspect` or `twx.model.resolve` before
writing. Tool, playbook and skill names referenced across files must agree.

## `skills/<id>/SKILL.md`

Front-matter plus: when to use, workflow, evidence rules, answer shape,
guardrails. Keep prose short. Label a generated skill a **first-pass draft**
until it has been tested with a real turn.

## `playbooks/<id>/playbook.json`

A skill is LLM guidance; a playbook is a deterministic DAG run by the engine.
Hard guards the engine enforces:

- exactly one `llm_summary` node and it must be the final node;
- no `provider`; `fan_out.maxConcurrency` must be 1; no orphan nodes;
- `tool_call` tools must be `playbookSafe`;
- `derive` operations come from a **fixed allowlist** — on parler-agent
  0.1.190+ it includes the service-orchestration ops listed in the README's
  Versions table; a transform that is not shipped cannot be expressed (needs
  a new Java built-in or a restructure);
- V1 out of scope: dynamic planning, arbitrary expressions, fan-out
  concurrency, cross-turn memory; `continueOnError` is V1b+ (V1a is
  fail-fast).

Package as `/playbooks/<id>/playbook.json`. Do not promise it runs until it
has been run.

## `host-contexts/<mashup_key>.json`

- `schema` is exactly `"parler-host-context-template"`; `key` equals the
  Mashup's `mashup_key` **and** the filename.
- `requiredContextFields`: the top-level `context` keys the template relies
  on — a turn missing one is rejected (`SCHEMA_REJECT`).
- `maxRenderedChars` default 4000; the whole host-context JSON is capped at
  16384 bytes on the wire.
- `promptTemplate` lines render fields with formatters: `jsonFence` (whole
  line, kebab-case block name, unique), `list`, `typedList`, `filters`,
  `timeWindow`, `hierarchy`, `kv`. An unknown formatter or wrong arity fails
  at load.
- A page-supplied node **id** routes to `hierarchyNodeId` and is used
  directly; a typed **label** uses `hierarchyNodeName` and is resolved. Page
  scope is advisory — the template must tell the model to use it.
- Dry-run with `ValidateHostContext`, then reload the AgentThing.
- Needs parler-agent 0.1.192+ (templates), 0.1.193+ and widget 0.1.84+
  (per-turn snapshots, `hierarchyNodeId`), 0.1.206+ (generic fallback for
  unregistered keys).

## Document knowledge

Gated by `AgentSettings.documentKnowledgeBuiltinsEnabled = true` plus a
configured repository. One folder per document under
`documentKnowledgeRootPath`, keyed by `docId` (lowercase, hyphens, no
extension): `manifest.json`, `source/original.pdf`, `markdown/manual.md`,
`chunks/chunks.jsonl` (page chunks always; section chunks with a real TOC;
≤ 30 keyword-signal chunks), optional `pages/*.png`. Every
`sourceLinks[0].href` is a FileRepository PDF link with `#page=N`. Deploy as
one archive, extract in place with the FileRepository's `ExtractZipArchive`,
then `RestartThing` (or wait `documentKnowledgeIndexTtlSeconds`, default
300). Converter and validator: `parler/docs/operations/pdf-conversion-agent-playbook.md`.

## Alert summary (0.1.202+)

`query_alert_summary` takes `thingNames: ["<canonical Thing name>", …]`
(1–25); a scalar `thingName` argument in a skill or playbook is the 0.1.202
migration bug. `query_alert_history` stays scalar.

## Returning files to a developer

Return the full file in a fenced block with the path it belongs at. Fence
with **tildes** (`~~~~`), because `SKILL.md` files contain backtick fences.

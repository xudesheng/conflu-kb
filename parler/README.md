_Digested 2026-08-23 from parler@ea41f8c5 (parler-agent 0.1.210, parler-ui-widget 0.1.89), ParlerGuidance@f5112da, parler-workshop@ce7ae5b._

# Parler Kit — KB digest

Advisory knowledge for a coding agent that configures, deploys, diagnoses,
and tests **Parler**, the ThingWorx extension that embeds an AI data-insight
agent (`AgentThing`). Start with `guide/start-here.md`. Anything
version-sensitive is checked against the live system, not trusted from here.

## What is here

| Path | Content | Origin |
|---|---|---|
| `guide/` | three authored pages: orientation, configuration rules, diagnosis recipe | written for this digest |
| `skills/` | `parler-taxonomy`, `parler-wrap-tools`, `parler-deploy` — procedures through Conflu; copy a skill directory into your agent's skill folder to activate it | written for this digest |
| `parler/docs/…`, `parler/CONTRACTS/…` | a selection of Parler's design documents and its normative contracts, unchanged | parler repository, at original paths |
| `parler/dev_data/scpa_utilization/` | a complete, working configuration repository (taxonomies, tools, skills, playbooks, policies, host contexts) — the golden example | parler repository |
| `ParlerGuidance/src/` | the four-day workshop course book, chapters 1–19 and appendices A–O (figures omitted) | ParlerGuidance repository |
| `ParlerGuidance/workshop/`, `parler-workshop/workshop/` | the instructor's and the students' per-day configuration files, exercises and evals | the two workshop repositories |

Not included: Parler source code, the remaining design documents, the
contract changelog, the course book's images, slides, and anything binary.
Links from the copied documents to files outside this selection will not
resolve; the text still stands on its own.

## Versions

This digest describes **parler-agent 0.1.210** and **parler-ui-widget
0.1.89**. Read the installed versions with `conflu twx extensions list`
(column `packageVersion`) before relying on anything below; if the system is
older, the capability is simply absent — say so once and work with what is
there, or recommend the upgrade.

Capabilities that appeared at a specific version (older systems lack them):

| Needs | Capability |
|---|---|
| parler-agent 0.1.190 | service-orchestration playbook primitives (`normalize_resolved_things`, `extract_from_tool_output`, `build_nested_object`, `json_stringify`, `resolve_time_window_for_playbook`, `empty_rows_if_skipped`, `add_computed_fields`, `collect_values`, `join_values`) and `$infotable` binding to extended tools |
| parler-agent 0.1.191 | skill-to-playbook converter knowledge pack, structured validation reports, `playbookRuntime`, last-run collection |
| parler-agent 0.1.192 | host-context `key + context` templates (`/host-contexts/<mashup_key>.json`) |
| parler-agent 0.1.193 + widget 0.1.84 | per-turn host-context snapshots, collapsed UI disclosure, `hierarchyNodeId` direct node scope |
| parler-agent 0.1.202 | `query_alert_summary` takes `thingNames[]` (1–25) instead of scalar `thingName`; `query_alert_history` stays scalar |
| parler-agent 0.1.205 + widget 0.1.89 | unified `build_history_overlay_chart` (replaces `build_period_over_period_chart` / `build_multi_series_history_chart`) with normalized axes |
| parler-agent 0.1.206 | generic fallback for unregistered host-context keys (no built-in classpath templates) |
| parler-agent 0.1.210 | the Day-4 utilization playbook's continuing `$ref` through `toolOutput.result` (JSON-string results parsed before later segments) |

The course book was taught on parler-agent 0.1.209/0.1.210; where a chapter
and a Parler document disagree, the Parler document is newer.

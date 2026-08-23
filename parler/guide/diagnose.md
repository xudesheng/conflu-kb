# Diagnose a Parler turn with live evidence

Use this when a question is about *what actually happened* on a system —
"I asked X, expected Y, got Z" — or when a skill/playbook ran but behaved
wrongly. Explanations and authoring questions do not need this.

## 1. Collect (read-only, dev profile)

| Evidence | Conflu call | Look for |
|---|---|---|
| Runtime snapshot | `twx.service.invoke` → `Things/<AgentThing>/Services/GetAgentRuntimeSnapshot` | `extensionVersion`; `configurationRepository.thingName`; per-file loaded hash vs current hash, `driftChecks[]`; registered playbooks (id, path, node count, required inputs) |
| Settings | `GetConfigurationTable` `{"tableName":"AgentSettings"}` | document-knowledge fields; provider settings (never echo secrets) |
| Validation | `ValidateAgentConfigurationRepository` | structured errors by file |
| Application log | `twx.log.query` `{"log_name":"ApplicationLog", …}` over the turn's window | `LLM_PLAYBOOK_RUN status=… failureCode=…`, `evidence_too_large`, `needs_clarification`; tool router and taxonomy resolver lines; which tools were selected and what they returned |
| Repository files | `twx.filerepo.download` of the files involved (`taxonomies/…`, `tools/extended_tools.json`, `skills/<id>/SKILL.md`, `playbooks/<id>/playbook.json`, `policies/invoke_service.json`, `host-contexts/<key>.json`) | the bytes currently in the repository — compare with the snapshot's loaded hash before calling them "what ran" |

Parler's own collection tool (`parler/docs/agent/collection-tool.md`) bundles
the same evidence for humans; through Conflu you take the same REST calls
one at a time.

## 2. Read

1. **Version first.** Several capabilities are version-gated (README
   "Versions"). If the snapshot's version is below what the configuration
   assumes, that is the finding — report it before reading JSON.
2. **Drift second.** If a file's loaded hash differs from its current bytes,
   what ran is not what is in the repository. Say so plainly; the fix is
   `RestartThing` (after confirming with the developer), then re-test.
3. **Then the turn.** From the log: which skill or playbook was selected,
   which tools were called with which arguments, what evidence came back,
   where a node failed (its failure code) or evidence was missing, and where
   the model went wrong. The live per-node progress frames are in-memory
   only and never in the log — reconstruct the sequence from the log lines.
4. **Name the lever.** Root causes in the workshop were almost always one
   of: a missing taxonomy entry, a tool description the model could not act
   on, a policy denying the service, an evidence rule in a skill, a playbook
   guard violation, or an `INFOTABLE` input the model could not construct.

## 3. Report

Lead with the root cause and the concrete fix (which file, what change).
Show the evidence lines that support it. State what you could not verify.
If you produce a corrected file, return it whole (tilde fence) with its
path, and offer the deploy choreography in `skills/parler-deploy`.

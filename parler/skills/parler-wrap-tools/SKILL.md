---
name: parler-wrap-tools
description: Wrap a developer's ThingWorx services as LLM-callable Parler tools in tools/extended_tools.json, choosing scalar wrapper services where a raw service is hostile to an LLM, and hand the result to parler-deploy.
---

# parler-wrap-tools

Use when a developer wants their Parler agent to call their own services —
"expose GetShiftUtilization to the agent", "make my APIs AI-friendly".

## Preflight (always)

1. Installed versions: `conflu twx extensions list` (note once).
2. AgentThing and repository: `GetAgentRuntimeSnapshot` via `twx.service.invoke`.
3. Current `tools/extended_tools.json` and `policies/invoke_service.json`
   via `twx.filerepo.download` (either may not exist yet).
4. KB: `guide/configuration-rules.md` (tools and policies sections),
   `parler/docs/agent/CUSTOMIZED-TOOLS.md`, book ch. 13 and 16, golden
   example `parler/dev_data/scpa_utilization/tools/extended_tools.json`.

## Procedure

1. **Inspect the candidate services** on the target with `twx.entity.inspect`
   (service definitions: inputs, base types, result shape). Classify each
   input: scalar (fine), `INFOTABLE`/DataShape (the trap — see rules).
2. **Decide the surface by user intent, not by the Mashup.** One tool per
   bounded question the user would ask; a bounded name
   (`[A-Za-z][A-Za-z0-9_]{0,63}`), a `whenToUse` the model can act on, a
   result the model can trust as evidence.
3. **Where a raw service is LLM-hostile**, sketch a thin scalar wrapper
   service (inputs the model can fill from conversation; bounded output) and
   offer to author it on the dev system with `twx.entity.edit` +
   `twx.entity.push` — only with the developer's go-ahead, dev profile.
4. **Write the entries**: name, `whenToUse`, target service, `hitl` for
   anything that changes state, `playbookSafe` only for tools a playbook may
   call. Add the services to `policies/invoke_service.json` — it is
   fail-closed; an unlisted service cannot be invoked.
5. **Self-check**: names match the grammar and are unique; every target
   service exists on the target; policy and tools agree.
6. **Hand over**: both files whole in tilde fences with paths, first-pass
   drafts; offer `parler-deploy`.

## Guardrails

- Read-only until asked to deploy; wrapper services are a write to the dev
  system and need explicit agreement.
- Never wrap a service that mutates state without `hitl`.
- Say version deviations once; do not refuse.

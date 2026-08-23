---
name: parler-taxonomy
description: Author or extend a Parler agent's asset-types.json / identity-types.json from the live ThingWorx model, grounded in entities that actually exist, and hand the result to parler-deploy.
---

# parler-taxonomy

Use when a developer asks to teach their Parler agent what their
application's things and identities mean — "add my HVAC units", "the agent
does not know what a Line is", "build asset-types.json for project X".

## Preflight (always)

1. Read the installed versions: `conflu twx extensions list` → `packageVersion`
   of `parler-agent` and `parler-ui-widget`. Note them once.
2. Find the AgentThing and its repository: `twx.service.invoke`
   `GetAgentRuntimeSnapshot` → `configurationRepository.thingName`. If there is
   more than one AgentThing, ask which.
3. Read the current files: `twx.filerepo.download` of
   `taxonomies/asset-types.json` and `taxonomies/identity-types.json` (they may
   not exist yet — that is fine).
4. Read the KB: `guide/configuration-rules.md` (taxonomy section),
   `parler/docs/agent/AGENT-TAXONOMY.md`, and the golden example
   `parler/dev_data/scpa_utilization/taxonomies/`.

## Procedure

1. **Ground in the live model.** For each kind of thing the developer names,
   resolve it on the target: `twx.model.resolve` / `twx.entity.inspect` for the
   ThingTemplate, ThingShape, or Things involved, their key properties, and the
   services that expose them. Never write an `entityName` / `entityType` you
   have not seen on the target.
2. **Interview for meaning.** Ask only what the model cannot tell you: what
   the business calls this kind of thing, which properties matter for
   questions, how identities (sites, lines, machines, users) relate. Keep it
   short; prefer concrete examples from the developer.
3. **Write the entries** following the golden example's shape and
   `AGENT-TAXONOMY.md`. Consistency across files matters more than
   completeness: names used here must match what `tools/`, `skills/`, and
   `playbooks/` use.
4. **Self-check** against `guide/configuration-rules.md`: every referenced
   entity resolves; no duplicate ids; JSON parses.
5. **Hand over.** Return both files whole in tilde fences with their
   repository paths, labelled first-pass drafts, and offer `parler-deploy`
   (diff → upload → restart → validate → one `Chat` test).

## Guardrails

- Read-only until the developer asks to deploy; deploying is `parler-deploy`'s
  job and needs a dev profile.
- If the installed version lacks something you would rely on, say so once
  and continue with what exists.
- Do not invent entities, property names, or service names.

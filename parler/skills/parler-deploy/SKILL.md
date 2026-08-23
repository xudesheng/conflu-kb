---
name: parler-deploy
description: Deploy changed Parler configuration files to the AgentThing's repository safely — diff, show, confirm, upload, restart, validate, test one turn — using Conflu's generic tools and a dev profile.
---

# parler-deploy

Use after `parler-taxonomy`, `parler-wrap-tools`, or any other change to
files under the AgentThing's configuration repository (`taxonomies/`,
`tools/`, `skills/`, `playbooks/`, `policies/`, `host-contexts/`).

## Preconditions

- A Conflu **dev** profile for the target (`-e <env>`); this skill writes to
  a FileRepository and restarts a Thing.
- The AgentThing name and its repository name from `GetAgentRuntimeSnapshot`
  (`configurationRepository.thingName`). Never deploy to a repository the
  developer has not named.

## Choreography (in this order, every time)

1. **Diff.** `twx.filerepo.download` the current remote file (if it exists)
   and diff against the new content. Show the developer the diff, not just
   the new file.
2. **Confirm.** Ask explicitly: upload these N files to `<repo>`? Proceed
   only on a clear yes.
3. **Upload.** `twx.filerepo.upload` each file to its repository path
   (`content_text` for JSON/Markdown). Keep the downloaded originals so the
   change can be reverted by uploading them back.
4. **Restart.** The AgentThing loads files at (re)start. Say that a restart
   interrupts live conversations, confirm, then `twx.service.invoke`
   `Things/<AgentThing>/Services/RestartThing`.
5. **Validate.** `twx.service.invoke` `ValidateAgentConfigurationRepository`;
   then `GetAgentRuntimeSnapshot` and check the uploaded files show no
   loaded-vs-current drift. Report errors by file.
6. **Test one turn.** `twx.service.invoke` `Chat` with a prompt that should
   exercise the change (synchronous; the result returns from the call), then
   `twx.log.query` `ApplicationLog` for the turn: was the new taxonomy
   entry / tool / skill used? Report what the evidence shows.
7. **Report.** What was uploaded (paths), restart done, validation result,
   the test prompt and outcome, and how to revert.

## Guardrails

- Never skip the diff or the confirmations; never bundle an unrelated file.
- If validation fails, offer the revert (upload the originals, restart)
  before anything else.
- Host-context templates: `ValidateHostContext` dry-run before step 3.
- Document-knowledge packages are deployed as one archive extracted in
  place with the FileRepository's `ExtractZipArchive`, then restart — see
  `guide/configuration-rules.md`.

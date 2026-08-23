# Outlook

## Near-term team work

- Expand **`extended_tools.json`** coverage until **all** high-value utilization questions map to **one or two** tool calls.
- Grow **`/skills/`** for recurring operator questions; add **eval YAML** so regressions are caught before extension import.
- Promote only **mature** skills to **playbooks** where the DAG is stable and evidence shape is known.
- Move workshop issue support into the future `parler-workshop` repository and local-runner flow.

## Platform and contract discipline

Any change to **wire JSON**, **history hydration**, or **tool envelopes** must ride with **`CONTRACTS/*`** updates and **`CONTRACTS/CONTRACT_VERSION.md`**. Train contributors to treat **`CONTRACTS/API_CONTRACT.md`** as the negotiation surface between **`parler-ui`** and **`parler-agent`**. These files live in the **parler** monorepo, not in the workshop bundle; this chapter uses them as maintainer pointers.

## Operations

- Centralize **reload** and **validate** procedures (`ValidateAgentConfigurationRepository`, `GetAgentRuntimeSnapshot`—see **`docs/agent/configuration-repository.md`**).
- Build a **version matrix**: ThingWorx platform, **`parler-agent`** extension version, **`parler-ui-widget`** version.

## Research directions (optional talking points)

- Richer **hierarchy** pagination (`afterId`, `maxChildNodes`) alignment with **`docs/architecture/entity-hierarchy.md`** §9.1.
- More **customer playbooks** once Provider services return **`PlaybookJson`** (see **`docs/agent/playbook-engine.md`** §2.2).

## TODO for authors

- Drop links to your **internal roadmap** tickets or quarterly goals.

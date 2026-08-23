# Playbook Regression Eval Commands

This is an operator run list, not a suite-composition file. The current `agent-eval` runner does not implement `includes:` or multi-suite manifests.

Run the existing focused Playbook suites directly:

```bash
uv run agent-eval --suite docs/agent/evals/cross_region_health_v1a.yaml --agent-matrix env --agent-filter gpt_5_4
uv run agent-eval --suite docs/agent/evals/cross_asset_pair_health_v1b.yaml --agent-matrix env --agent-filter gpt_5_4
```

Use explicit filters for Sonnet reference runs:

```bash
uv run agent-eval --suite docs/agent/evals/cross_region_health_v1a.yaml --agent-matrix env --agent-filter sonnet_4_6
uv run agent-eval --suite docs/agent/evals/cross_asset_pair_health_v1b.yaml --agent-matrix env --agent-filter sonnet_4_6
```

Future no-alert fixture cases should be added to `cross_asset_pair_health_v1b.yaml`, not to a duplicate Playbook suite.

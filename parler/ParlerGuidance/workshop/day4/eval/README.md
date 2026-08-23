# Day 4 eval pack

This folder is a self-contained workshop copy of the Parler live Agent eval runner and sample suites.

Run commands from this directory:

```bash
cd workshop/day4/eval
cp .env.example .env
# edit .env with your DEV_SERVER, DEV_KEY, and AgentThing names

uv run agent-eval --suite customer-evals/smoke.yaml --agent-matrix env
```

The runner calls `AgentThing.Chat(...)` through ThingWorx REST, then reads `AgentMessageStream` to evaluate final
answers, tool calls, tool arguments, and tool results. It is not a UI test.

## Layout

```text
pyproject.toml
.env.example
test_scripts/agent_eval.py
docs/agent/agent-evaluation-harness.md
docs/agent/evals/
customer-evals/
```

`docs/agent/evals/` contains the current Parler sample suites copied for the workshop. `customer-evals/` contains the
recommended App-specific pack shape described in Appendix L.

## Useful commands

```bash
uv run agent-eval --suite customer-evals/smoke.yaml --agent-matrix env
uv run agent-eval --suite customer-evals/workflows.yaml --agent-matrix env --agent-filter gpt_5_4
uv run agent-eval --suite customer-evals/workflows.yaml --agent-matrix env --case asset_current_status
uv run agent-eval --suite customer-evals/workflows.yaml --agent-matrix env --reset-mode stable_clear
uv run agent-eval --suite customer-evals/workflows.yaml --agent-matrix env --out-dir customer-evals/reports
```

Reports are written to `tmp/agent-eval/<timestamp>/` by default, or under the `--out-dir` base when supplied.

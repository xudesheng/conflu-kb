# Taxonomy v2 — operator checklist (M2)

Structured taxonomy is **`/taxonomies/identity-types.json`** **`version: 2`** only. Legacy **`asset-types.json`** is ignored (warning **`TAXONOMY_LEGACY_FILE_PRESENT`**).

## Preconditions

1. Copy **`docs/agent/evals/fixtures/identity-types-minimal.json`** → configuration repository **`/taxonomies/identity-types.json`**.
2. **`RefreshTaxonomyCache`** on each matrix AgentThing.
3. **`export AGENT_EVAL_TAXONOMY_V2=1`** before running **`docs/agent/evals/taxonomy_v2.yaml`**.
4. Optional identifier env vars (same pattern as prior gates): **`AGENT_EVAL_TAXONOMY_JET_DRYER_EXACT_NAME`**, **`_DISPLAY`**, **`_SERIAL`**.
5. Unavailable tri-state: **`taxonomy_v2_unavailable.yaml`** with **`--agent-matrix env`** and **`AGENT_EVAL_AGENT_GPT_5_4`** (see suite header).
6. Playbook regression: run **`cross_region_health_v1a.yaml`** separately under **`full_table`** (Playbook harness naming; not taxonomy V1a).

## R2 `queryParent` coverage

Suite **`taxonomy_v2.yaml`** includes **`resolve_asset_type_workunit_query_parent`** and **`list_asset_types_includes_workunit_query_parent`**: after deploying **`identity-types-minimal.json`**, both cases assert the tool JSON echoes **`queryParent.entityName`** **`PTC.MfgModel.DefaultWorkunit_TT`** for type **Workunit Jet Dryer**.

## Commands

```bash
uv run agent-eval --suite docs/agent/evals/taxonomy_v2.yaml --agent-matrix yaml --agent-filter gpt_5_4
```

Attach `tmp/agent-eval/<timestamp>/report.json` when closing an operator gate.

## Non-goals

- PASSWORD-only properties (unit tests).
- `totalCount > 5000` truncation probes unless the environment reproduces them.

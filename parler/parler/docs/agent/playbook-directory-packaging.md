# Playbook Directory Packaging

**Status:** Implemented
**Topic:** `playbook-directory-packaging`
**Scope:** `parler-agent` configuration repository playbook discovery, validation, diagnostics, docs, and fixtures

**Relationship to shipped docs:** The repository layout described here is the implemented playbook packaging model. `docs/agent/playbook-engine.md` keeps historical notes for pre-directory-packaging branches, but the runtime loader now uses directory discovery.

## 1. Background

The current playbook packaging model uses a central catalog plus one or more playbook body files:

```text
/playbooks/playbooks.json
/playbooks/<id>.playbook.json
```

This was acceptable while playbooks were a small engine demo, but it is already becoming friction in workshop and app-developer workflows. Every new playbook requires two coordinated edits:

1. Add or update the playbook JSON file.
2. Add or update the corresponding row in `playbooks.json`.

That double-entry model is fragile. A user can upload a valid playbook file and still fail to load it because the central catalog was not updated, or update the catalog with the wrong `playbookPath`, id, title, or input schema. This is especially painful during training, where the developer is already writing JSON-heavy DAG definitions and should not also maintain a second JSON registry by hand.

Skills already have a more natural authoring model: each unit can be reasoned about as its own file or directory artifact. Playbooks should move in the same direction. A playbook should be an independently uploadable package, not a row in a global catalog.

## 2. Design Goal

Replace the central `playbooks.json` catalog with directory-based discovery:

```text
/playbooks/<playbook-id>/playbook.json
```

Each immediate child directory under `/playbooks` represents one candidate playbook package. The only effective runtime file in that directory is exactly:

```text
playbook.json
```

Examples:

```text
/playbooks/cross_asset_pair_health/playbook.json        # effective
/playbooks/cross_asset_pair_health/README.md            # ignored
/playbooks/cross_asset_pair_health/playbook_v1.json      # ignored
/playbooks/cross_asset_pair_health/playbook_v2.json      # ignored
/playbooks/cross_asset_pair_health/fixtures/sample.json  # ignored
/playbooks/playbooks.json                                # ignored
/playbooks/cross_asset_pair_health.playbook.json         # ignored
/playbooks/.scratch/playbook.json                        # ignored (dot-prefixed package dir)
```

The goal is a clean mental model:

- To add a playbook, add one directory containing `playbook.json`.
- To remove a playbook, remove that directory or remove its `playbook.json`.
- To draft alternate versions, keep them outside the effective filename.
- To activate a different draft, rename or copy it to `playbook.json`.

No runtime version suffix is part of this design. If developers want `playbook_v1.json`, `playbook_v2.json`, or branch-specific variants during authoring, that is a source-control or local-file concern. The runtime should only load one active file per playbook directory.

## 3. Non-Goals

- No support for the old central `/playbooks/playbooks.json` format.
- No migration layer that reads both old and new formats.
- No warning or error when legacy files exist. They are simply not part of discovery.
- No support for multiple active versions of the same playbook id.
- No recursive playbook discovery below nested subdirectories.
- No broad change to the playbook DAG schema or engine semantics.
- No change to the **`whenToUse` wire type**: it remains a **single JSON string**, as in shipped `playbooks.json` and `PlaybookRegistryBuilder` catalog parsing (`optString`). Promoting an array-of-strings authoring shape is a separate topic (parser, validator, snapshot rows, tool copy, fixtures).

The clean break is intentional. In-repo development fixtures and agent docs can be converted once as part of this topic. External training repositories should be converted separately after the implementation is accepted and merged. Carrying both formats would preserve the very confusion this topic is meant to remove.

## 4. Discovery Contract

The repository loader discovers playbooks using this exact pattern:

```text
/playbooks/*/playbook.json
```

Discovery rules:

1. Enumerate the immediate child directories of `/playbooks` (same repository listing seam as skill discovery: `RepositoryReader.getFileListing` over `/playbooks`, as `RepositorySkillScanner` does for `/skills`).
2. **Exclude** any child whose directory name is empty or whose **first code unit is `.` (U+002E)**. Dot-prefixed package dirs (for example `/playbooks/.scratch/`) are authoring-only sandboxes: **no** `playbook.json` load attempt and **no** loader diagnostics (same silence class as legacy flat files).
3. For each remaining child directory, attempt to load `<child>/playbook.json`.
4. Ignore any other files or nested directories under those packages.
5. Sort candidate directory names lexicographically (**Unicode code-point / `String::compareTo`**) before loading attempts, so runtime snapshots and diagnostics are deterministic.
6. Validate each discovered `playbook.json` independently.
7. Load all valid playbooks; report diagnostics for discovered but invalid `playbook.json` files only for **non–dot-prefixed** package directories that passed step 3.

Files and directories outside this pattern **do not produce playbook-loader diagnostics**. This includes old catalog files, draft playbook files, README files, fixtures, and any authoring scratch files.

**Deterministic ordering:** sort discovered package directory names lexicographically by Unicode code-point order of the directory segment (simple `String::compareTo` style), independent of host default locale, so snapshots and logs match across JVMs.

## 5. Id And Path Rules

The directory name and playbook document id should be the same:

```text
/playbooks/cross_asset_pair_health/playbook.json
```

```json
{
  "id": "cross_asset_pair_health"
}
```

This should be a hard validation rule for discovered `playbook.json` files. If the directory is `cross_asset_pair_health` but the document says `"id": "other_id"`, the playbook is invalid and skipped with a diagnostic that includes the exact path.

Reasoning:

- It gives operators a fast way to inspect the repository.
- It prevents invisible aliasing.
- It avoids needing a separate `playbookPath` field.
- It makes duplicate ids structurally difficult.

Recommended id character set remains conservative: lower-case letters, digits, and underscores. If the current validator already has a stricter or equivalent id rule, keep it.

## 6. Single-File Metadata

Because the catalog disappears, metadata currently held in `playbooks.json` must move into `playbook.json`.

The effective playbook document should contain enough metadata for both runtime admission and LLM-facing tool descriptions:

```json
{
  "schema": "parler-playbook-v1",
  "id": "cross_asset_pair_health",
  "title": "Cross Asset Pair Health",
  "description": "Compare two assets using alerts, current summaries, and recent property trends.",
  "whenToUse": "Use when the user asks to compare the operational health of two assets, or which of two assets is more urgent or degraded.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "assetA": { "type": "string" },
      "assetB": { "type": "string" },
      "window": { "type": "string", "default": "24h" }
    },
    "required": ["assetA", "assetB"]
  },
  "execution": {
    "mode": "sync"
  },
  "budgets": {
    "maxToolCalls": 20
  },
  "nodes": [],
  "finalNode": "final"
}
```

The exact field names should stay aligned with the existing playbook document model where possible. The key design point is that there is no separate catalog row and no authored `playbookPath`.

Implementation may keep an internal `PlaybookCatalogEntry`-like object if that is convenient, but it must be derived from `playbook.json` and the discovered path.

### 6.1 Merged root JSON — fields and validation delta

Catalog-era playbooks split metadata (`playbooks.json` row) from the DAG document (`*.playbook.json`). This topic **merges** them into one `playbook.json` root object. **`PlaybookDocument.parse`** (or an equivalent single-file parse step) MUST read **both** the existing DAG fields and the former catalog fields from that root:

| Area | JSON fields | Notes |
|------|-------------|--------|
| Schema / identity | `schema`, `id` | `schema` MUST equal shipped constant `PlaybookIds.SCHEMA_V1` (`parler-playbook-v1`). `id` MUST equal the parent directory name (§5). |
| Former catalog metadata | `title`, `description`, `whenToUse`, `inputSchema`, `execution` | `whenToUse` is a **scalar string** (same as today’s `optString` catalog parse). `inputSchema` and `execution` are objects; use empty `{}` when absent, matching catalog defaults. |
| DAG body | `budgets`, `nodes`, `finalNode` | Unchanged semantics; existing `PlaybookDocument` graph parsing applies. |

**Admission and diagnostics (normative):**

- **`PlaybookValidator.validateDocument`** (and/or a dedicated admission pass invoked for each discovered file) MUST enforce metadata that the runtime previously received only via `validateCatalog` + catalog row + document split. At minimum, spell out in implementation (and cover in tests):
  - **Invalid package (diagnostic, skipped):** JSON parse failure; `schema` ≠ `parler-playbook-v1`; `id` missing / grammar / **directory mismatch**; DAG validation failures; unsafe tool/operation references; missing **`finalNode`** / graph rules already enforced today.
  - **Metadata for LLM / tool admission:** Require parity with what operators expect when registering a playbook today: non-blank **`title`** for snapshot and routing surfaces; **`inputSchema`** present as a JSON object (may be minimal empty object only if that remains compatible with `start_playbook` parameter validation — if not, tighten in code and mirror here); **`description`** and **`whenToUse`** MAY be empty strings (catalog used `optString` with default `""`).
- Removing **`validateCatalog`** and **`playbookPath`** checks MUST NOT drop guarantees: any constraint that lived only in catalog validation MUST be **re-homed** onto the merged document path.
- **Collection cap (re-homed from `validateCatalog`):** at most **`PlaybookIds.MAX_PACKAGED_PLAYBOOKS` (32)** packages may load successfully. When more than 32 valid directories exist after sorting, the registry **loads the first 32** in lexicographic directory-name order and emits a **diagnostic** for the cap (operators see truncation; no silent drop of already-loaded rows).
- **`inputSchema`:** absent or non-object JSON values parse as **`{}`** (via `JSONObject.optJSONObject` semantics, matching catalog defaults). Stricter JSON-Schema validation of parameters remains future work; **`start_playbook`** continues to receive caller JSON as today.

### 6.2 `PlaybookRegistrySnapshot` flags (partial success)

Today the builder is all-or-nothing: the first catalog or document failure yields `PlaybookRegistrySnapshot.empty(...)` with `isLoaded() == false`. Directory discovery with **per-package** validation requires:

- **`isLoaded()`** is **true** iff **at least one** playbook was loaded successfully after discovery (success count ≥ 1). It MUST **not** become true merely because `/playbooks` was readable if every package failed validation.
- **`reservedSlashIds()`** returns exactly the set of **successfully loaded** playbook ids (the same keys as the in-memory success maps). **Invalid** discovered package ids appear **only** in `diagnostics()`; they MUST **not** reserve slash names and MUST **not** suppress repository skills with the same short id.

**Consumer checklist (re-verify under partial success when implementing):**

- `AgentThing.getMergedToolDefinitions` — register `start_playbook` only when `isLoaded()` is true (implies ≥ 1 valid playbook).
- `AgentThing.buildLlmTurnContext` — skill short-id suppression uses `reservedSlashIds()` (must not include failed packages).
- `AgentThing.tryExecutePlaybookSlashTurn` — returns early when `!isLoaded()`; playbook slash routing uses `reservedSlashIds()` only.
- `AgentThing` `GetAgentRuntimeSnapshot` playbooks section — `loaded`, `catalogIds`, `documentIds`, `reservedSlashIds`, and per-row paths reflect **successful** loads; diagnostics list failed packages.
- `SkillRegistryBuilder.build(..., playbookRegistry.reservedSlashIds())` — receives only successful playbook ids.
- `ConfigurationRepositoryLoadedFileCaptures` — union paths for captures reflect discovered effective `playbook.json` files (not legacy catalog pair).

**Normative empty catalog:** If zero playbooks load successfully, `isLoaded()` MUST be **false**. Then there MUST be **no** model-facing `start_playbook` tool registration and **no** playbook slash ids reserved — even when `diagnostics()` is non-empty due to failed sibling packages.

## 7. Validation Semantics

Validation should distinguish between ignored files and discovered invalid playbooks:

Ignored without diagnostics:

- `/playbooks/playbooks.json`
- `/playbooks/*.playbook.json`
- `/playbooks/<id>/README.md`
- `/playbooks/<id>/playbook_v1.json`
- `/playbooks/<id>/fixtures/*`
- any file not named exactly `/playbooks/<id>/playbook.json`

Reported as diagnostics:

- `/playbooks/<id>/playbook.json` cannot be parsed as JSON.
- document id is missing or invalid.
- document id does not match `<id>`.
- required metadata for LLM/tool admission is missing.
- input schema is invalid.
- DAG nodes fail existing playbook validation.
- referenced tools or operations fail existing playbook safety/admission checks.

The loader should prefer partial success. If one discovered playbook is invalid and two are valid, the valid playbooks should still load. The invalid playbook should appear in diagnostics with its path and error summary.

If `/playbooks` exists but **zero** packages yield a successfully loaded playbook (including “no candidate dirs after dot-prefix filter” and “every discovered `playbook.json` invalid”), the runtime MUST treat the registry as **not loaded** for tool and slash purposes: **`isLoaded() == false`**, no `start_playbook` in the merged tool list, and **`reservedSlashIds()` empty**. Diagnostics MAY still list per-path failures from invalid siblings. This matches the test plan expectation that an empty effective catalog produces **no** model-facing playbook tool advertisement.

## 8. Runtime Snapshot And Diagnostics

`GetAgentRuntimeSnapshot` should expose enough information to prove what was loaded without requiring a user to inspect repository files manually.

Recommended additive fields under the existing playbook snapshot area:

```json
{
  "playbooks": {
    "loaded": true,
    "discoveryPattern": "/playbooks/*/playbook.json",
    "catalogIds": ["cross_asset_pair_health"],
    "documentIds": ["cross_asset_pair_health"],
    "documents": [
      {
        "id": "cross_asset_pair_health",
        "path": "/playbooks/cross_asset_pair_health/playbook.json",
        "title": "Cross Asset Pair Health"
      }
    ],
    "diagnostics": []
  }
}
```

Naming can follow the current snapshot structure, but the old term `catalog` should no longer imply a loaded `playbooks.json` file. **Reviewer decision:** retain **`catalogIds`** as a backwards-compatible derived list of **successfully loaded** ids; add **`discoveryPattern`** and per-document **`path`** so operators can see the directory-discovery model. A later rename (for example `loadedPlaybookIds`) is optional cleanup outside this topic’s critical path.

`ValidateAgentConfigurationRepository` should validate discovered `playbook.json` files only. It should not warn about legacy files or unrelated files under `/playbooks`.

**Authoring `items[]` severity:** invalid discovered packages (JSON parse failure, root schema rejection, document validation, unreadable file where a load was attempted) MUST surface as **`severity: "error"`** and increment **`summary.errors`**. The **32-package cap** and **diagnostics-list truncation** (when very many invalid packages would flood diagnostics) remain **`severity: "warning"`**. Runtime **`playbooks.diagnostics[]`** remains a list of human-readable strings derived from the same structured diagnostic records (`path`, `code`, severity).

The collection tool should capture:

- the discovered playbook paths,
- loaded playbook ids,
- invalid discovered playbook diagnostics,
- the `discoveryPattern`.

It should not attempt to collect ignored draft files unless a future explicit diagnostics mode asks for repository inventory.

## 9. Documentation and contract coupling

- **Formal agent docs:** Any path or registration narrative that still describes `/playbooks/playbooks.json` plus `*.playbook.json` must be updated in the same implementation slice as the Java change — especially `docs/agent/playbook-engine.md` (§3 registration and examples) and `docs/agent/playbook-app-tool-workflows.md`.
- **Collection / diagnostics docs:** If runbooks enumerate repository paths for playbooks, update them when discovery moves to `/playbooks/*/playbook.json`.
- **Normative wire:** Today the detailed `GetAgentRuntimeSnapshot` playbook subsection is not fully duplicated in `CONTRACTS/API_CONTRACT.md`. If a future change promotes new snapshot field names to normative wire text, update **`CONTRACTS/*.md`** and bump **`CONTRACTS/CONTRACT_VERSION.md`** in the **same** change as the Java that emits those fields (repo rule: contract changes stay coupled to code).

## 10. Implementation Touchpoints

Expected code areas:

- `PlaybookRegistryBuilder`
  - Replace `/playbooks/playbooks.json` loading with directory discovery.
  - Build runtime entries from discovered `playbook.json` documents.
  - Preserve deterministic ordering and **partial success** semantics (§6.2).
  - **Reuse** `RepositoryReader.getFileListing("/playbooks", "")` (already used for `/skills` in `RepositorySkillScanner`); filter, sort, then `RepositoryTextLoads.loadText` per `playbook.json` — no new ad-hoc repository protocol.

- Repository file access layer
  - **No new listing primitive is required** for Phase A if `FileRepositoryRepositoryReader` already exposes `getFileListing` for arbitrary repository paths (it does for skills). Playbook discovery should use that same visibility-aware listing entry point.

- `PlaybookIds`
  - Replace old path constants with:
    - `PLAYBOOK_ROOT = "/playbooks"`
    - `PLAYBOOK_FILE_NAME = "playbook.json"`
    - `DISCOVERY_PATTERN = "/playbooks/*/playbook.json"`
  - Add helper methods for deriving path from id and id from path.

- `PlaybookCatalogEntry`
  - Either rename it to something like `PlaybookRegistryEntry`, or update its meaning so it represents derived runtime metadata rather than a row from `playbooks.json`.

- `PlaybookValidator`
  - Remove `playbookPath` catalog validation.
  - Validate document `id` against directory id.
  - **Extend** document validation per §6.1 (metadata admission + re-homed catalog guarantees).
  - Keep existing DAG, input schema, tool, operation, and budget guards.

- `AgentThing`
  - Update runtime snapshot rows and `start_playbook` description text.
  - Avoid references that claim ids come from `/playbooks/playbooks.json`.

- `ConfigurationRepositoryLoadedFileCaptures`
  - Capture discovered effective playbook files, not the old catalog plus body-file pair.

- Tests and fixtures
  - Convert `dev_data/playbooks`.
  - Convert playbook test resources.
  - Remove assertions that require `/playbooks/playbooks.json` or `/playbooks/<id>.playbook.json`.

- Docs
  - Update `docs/agent/playbook-engine.md`.
  - Update `docs/agent/playbook-app-tool-workflows.md`.
  - Update collection-tool docs if they enumerate playbook repository paths.

## 11. Testing Plan

> **Design record (shipped @ 0.1.185).** Bullets below describe the original implementation scope and verification plan — not an open work list.

Unit and integration tests should cover:

1. A single valid `/playbooks/<id>/playbook.json` is discovered and loaded.
2. Multiple valid playbook directories are discovered in deterministic order.
3. `/playbooks/playbooks.json` is ignored without warning.
4. `/playbooks/<id>.playbook.json` is ignored without warning.
5. Draft files such as `/playbooks/<id>/playbook_v2.json` are ignored.
6. Nested files such as `/playbooks/<id>/drafts/playbook.json` are ignored.
7. Invalid discovered `playbook.json` reports a path-specific diagnostic.
8. Directory id and document id mismatch reports a path-specific diagnostic.
9. One invalid playbook does not prevent other valid playbooks from loading.
10. **Zero successfully loaded playbooks** (empty tree, all-invalid packages, or only dot-prefixed dirs) results in `isLoaded() == false` and **no** model-facing `start_playbook` tool advertisement.
11. A dot-prefixed sibling directory (for example `/playbooks/.draft/playbook.json`) is ignored **without** diagnostics.
12. Runtime snapshot reports `discoveryPattern` and loaded paths for successful packages.
13. `ValidateAgentConfigurationRepository` reports only discovered invalid playbooks (non–dot-prefixed packages).

Live validation should include:

1. Upload one playbook directory only, without `playbooks.json`.
2. Refresh the agent configuration.
3. Confirm runtime snapshot lists the playbook.
4. Run a prompt that should select the playbook.
5. Add a second playbook directory, refresh, and confirm both are available.
6. Remove one playbook directory or its `playbook.json`, refresh, and confirm it disappears.
7. Add ignored draft files and confirm they do not affect diagnostics.

## 12. Migration For Repo Artifacts

There is no runtime migration. Repository artifacts should be changed directly after implementation.

Example conversion:

Before:

```text
dev_data/playbooks/playbooks.json
dev_data/playbooks/cross_asset_pair_health.playbook.json
dev_data/playbooks/cross_region_health.playbook.json
```

After:

```text
dev_data/playbooks/cross_asset_pair_health/playbook.json
dev_data/playbooks/cross_region_health/playbook.json
```

ParlerGuidance workshop files are intentionally outside this topic. After this topic is implemented, merged, and accepted, those training files should be converted once in a separate follow-up performed outside the implementation branch.

## 13. Suggested Phasing

**Phase A — Discovery core**

- Use existing `RepositoryReader.getFileListing("/playbooks", "")` (same seam as skills); add filtering, sorting, and per-dir `playbook.json` loads.
- Implement `/playbooks/*/playbook.json` discovery with dot-prefix exclusion (§4).
- Build registry entries from merged document metadata.
- Switch `isLoaded` / `reservedSlashIds` semantics to partial-success model (§6.2) and keep `start_playbook` gating consistent.

**Phase B — Validation and diagnostics**

- Enforce directory id equals document id.
- Report invalid discovered `playbook.json` paths.
- Ignore non-effective files silently.
- Update runtime snapshot and repository validation output.

**Phase C — Fixtures and docs**

- Convert `dev_data/playbooks`.
- Convert test fixtures.
- Update agent docs and collection-tool path references.

**Phase D — Live acceptance**

- Import a repository containing only directory-style playbooks.
- Refresh the agent.
- Verify prompt selection and `GetAgentRuntimeSnapshot`.
- Verify ignored files do not create diagnostics.

## 14. Out-Of-Topic Follow-Up

After this topic is merged, the ParlerGuidance training repository should be converted separately:

- Convert workshop playbook examples to `/playbooks/<id>/playbook.json`.
- Remove workshop instructions that ask users to edit a central `playbooks.json`.
- Keep any draft or comparison files as ignored authoring artifacts beside the effective `playbook.json`.

This follow-up should not be performed by the implementation branch for this topic.

## 15. Reviewer decisions (from `playbook-directory-packaging-review-0`, 2026-06-07)

Codex and Claude **continue** toward implementation after this design refresh. Resolved items (normative for this topic):

1. **Schema example and constants:** All design examples and fixtures MUST use `schema: "parler-playbook-v1"` (`PlaybookIds.SCHEMA_V1`). No schema rename in this topic.
2. **`catalogIds` vs rename:** Retain `catalogIds` as the derived successful-id list; add `discoveryPattern` and paths for operator clarity.
3. **Dot-prefixed directories:** **Exclude** from discovery; silent ignore, no diagnostics (§4).
4. **Empty catalog / tools:** **Normative** — zero successful loads ⇒ `isLoaded() == false` ⇒ no `start_playbook`, no reserved playbook slash ids (§6.2, §7).
5. **Slash reservation under partial success:** Only **valid, loaded** playbook ids appear in `reservedSlashIds()`; invalid package ids never suppress same-named skills (§6.2).
6. **Parse/validation delta:** §6.1 is the controlling spec for merged-field ownership and validator/parser work formerly split across catalog + body file.
7. **Repository listing:** Reuse `RepositoryReader.getFileListing` + the same proven pattern as `RepositorySkillScanner` (§4, §10).

## 16. Acceptance Criteria

> **Design record (shipped @ 0.1.185).** Bullets below describe the original implementation scope and verification plan — not an open work list.

This topic is complete when:

- No runtime code path requires `/playbooks/playbooks.json`.
- No authored playbook requires `playbookPath`.
- A playbook can be added by adding only `/playbooks/<id>/playbook.json`.
- Old catalog files are ignored without warning.
- Discovered invalid playbook files produce clear diagnostics (for non–dot-prefixed packages).
- **Partial success:** at least one valid sibling loads while invalid packages diagnose; failed packages never appear in `reservedSlashIds()`.
- **Dot-prefixed** package directories are excluded from discovery with no diagnostics.
- Runtime snapshot shows the discovery pattern and effective loaded paths; `catalogIds` lists only successful loads.
- Tests cover valid discovery, ignored files, invalid discovered files, empty / all-failed catalog behavior, and dot-dir silence.
- Agent docs and development fixtures use only the new layout.

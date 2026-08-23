# Document chunk tools

Status: design draft.
Date: 2026-06-17.

This document specifies the Java `parler-agent` implementation plan for
document-grounded recommendations over already-converted PDF/manual packages.
It implements the Parler-side runtime described in `docs/core/pdf-search.md` and
consumes package output shaped by `docs/future/30-pdf-extraction.md`.

The motivating flow is:

```text
health-status evidence
  -> normalize issue / alarm / component
  -> search_document_chunks
  -> get_document_chunk
  -> final answer with FileRepository PDF page links
```

This topic does not implement PDF extraction. The extraction pipeline remains
outside Parler.

## 1. Placement

This design belongs under `docs/agent/` because it is about the ThingWorx Java
agent extension:

- Java built-in tool schemas.
- Java execution and FileRepository reading.
- JVM cache/index behavior.
- LLM routing and final-answer rules.
- Agent tests and fault-tolerance behavior.

Related documents:

- `docs/core/pdf-search.md` - package and answer contract.
- `docs/future/30-pdf-extraction.md` - external extraction pipeline and scale
  plan.
- `dev_data/future_repo/document-knowledge/earthly-cici-ops-v2/` - concrete
  fixture package.

## 2. Scope

Implement two built-in tools:

```text
search_document_chunks
get_document_chunk
```

These tools are exposed only when the AgentThing option
`documentKnowledgeBuiltinsEnabled` is `true`. The default is `false`.

These tools read document packages from a ThingWorx FileRepository-compatible
layout:

```text
document-knowledge/
  <docId>/
    manifest.json
    source/original.pdf
    markdown/manual.md
    chunks/chunks.jsonl
    pages/page-0001.png
```

The v1 implementation should be boring and deterministic:

- no embeddings;
- no external search system;
- no UI artifact lane;
- no new AlwaysOn frame type;
- no PDF parsing;
- no OCR;
- no extraction code.

The default-off switch is intentional. The Java implementation is a short-term
fallback and validation path. The long-term path may expose the same tool
contract through ThingWorx JavaScript wrapper services that call an external Rust
document pipeline/search service.

### 2.1 Supported retrieval contract — asset-type context is a precondition (User decision, 2026-06-26)

A document-symptom turn is supported **only when an asset-type context is
available**. Real users are not expected to name the specific manual, but every
such question MUST carry an asset-type context — one of:

- explicitly stated in the user message ("the KK&K steam turbine …"); or
- supplied via **host-context** (the bound Mashup / page asset scope); or
- readily derivable from the bound Thing's **identity** (thingName, ThingTemplate,
  asset model).

Parler will **not** build for the context-free generic symptom question ("how do I
respond to symptom X?" with no asset-type at all). That case is **out of supported
scope**, not a defect.

Consequences (binding):

1. **Asset-type context MUST reach document ranking from the first search of the
   turn** — threaded into `assetContext` / document scoring — not left for the model
   to volunteer on a retry. Where host-context or Thing identity is available, the
   runtime SHOULD populate asset-type so the first `search_document_chunks` already
   carries it. (Live evidence `c8cd3b8f` / the 2026-06-26 post-0.1.198 read: a
   symptom-only first search with no asset-type context ranked the wrong document
   family first; the turn recovered only because the model later added identity.)
2. **"Weak identity" means weak *document* identity *with* asset-type context** — the
   user did not name the manual, but the turn still carries asset type. It does
   **not** mean zero context. Acceptance and smoke tests MUST exercise the supported
   contract (asset-type context present); a zero-context symptom prompt is an
   explicit out-of-scope case, not a passing/failing fixture.
3. This is also the standing answer to §12.3 ("when to add embeddings"): the
   context-free symptom case that would motivate semantic recall is declared out of
   scope, so deterministic identity-aware ranking remains sufficient and embeddings
   stay deferred.

## 3. UI and wire contract impact

No UI change is required for v1.

The final answer uses ordinary Markdown links. `<parler-ui>` already renders
Markdown links, and the source link uses the same ThingWorx FileRepository path
style as existing table CSV download links:

```text
/Thingworx/FileRepositories/{repository}{path}#page={page}
```

Example:

```text
/Thingworx/FileRepositories/AIDocRepository/document-knowledge/earthly-cici-ops-v2/source/original.pdf#page=25
```

No `CONTRACTS/UI_CLIENT_PROTOCOL.md` or `CONTRACTS/API_CONTRACT.md` change is
required for v1 because:

- document citations are assistant Markdown, not a new wire artifact;
- no `type: "document"` frame is emitted;
- no reducer state is added;
- no chart/table contract is changed.

Possible future UI enhancements, out of scope for v1:

- right-side or modal PDF viewer;
- citation cards;
- highlighted chunk text;
- source preview thumbnails;
- a dedicated document artifact wire frame.

## 4. Configuration

The agent needs a small configuration surface. Field names are proposed; final
Composer naming can follow existing `AgentSettings` conventions.

| Field | Type | Default | Role |
| --- | --- | --- | --- |
| `documentKnowledgeBuiltinsEnabled` | boolean | `false` | When `true`, advertise the internal Java `search_document_chunks`, `get_document_chunk`, and `resolve_document_set` tools. When `false`, do not expose them to the model. |
| `documentKnowledgeRepository` | `THINGNAME` | empty | FileRepository Thing containing document packages. |
| `documentKnowledgeRootPath` | string | `/document-knowledge` | Root folder below the repository. |
| `documentKnowledgeIndexTtlSeconds` | integer | `300` | JVM cache TTL. |
| `documentKnowledgeMaxDocuments` | integer | `100` | v1 scan cap for package manifests. |
| `documentKnowledgeMaxChunks` | integer | `10000` | v1 cap for total indexed chunks. |
| `documentKnowledgeSearchDefaultLimit` | integer | `5` | Default search result count. |
| `documentKnowledgeSearchMaxLimit` | integer | `10` | Maximum search result count. |
| `documentKnowledgeSearchSnippetMaxChars` | integer | `400` | Maximum chars per search-match `snippet`. |
| `documentKnowledgeChunkMaxChars` | integer | `6000` | Maximum markdown chars returned by `get_document_chunk`. |

If the repository field is empty, the tools must return a structured degraded
result rather than throwing.

### 4.1 Built-in registration and name reservation

`documentKnowledgeBuiltinsEnabled` controls **whether the Java extension registers
these tools at all**, not merely whether they appear on the merged LLM tool list.

This is **not** the legacy-discovery executor-only pattern
(`advertiseLegacyServiceDiscoveryTools`). Executor-only registration still reserves
names through `AgentThing.builtinToolDefinitionNames()` via
`ToolRegistry.getExecutorOnlyAliases()`, which would block
`/tools/extended_tools.json` wrappers from using the same tool names.

When `documentKnowledgeBuiltinsEnabled` is `false`:

- do **not** call `ToolRegistry.register` for any of the document tools;
- do **not** call `ToolRegistry.registerExecutorOnly` for them;
- do **not** add any of those names to the built-in name-reservation set;
- leave `search_document_chunks`, `get_document_chunk`, and `resolve_document_set`
  available for extended tools backed by ThingWorx JavaScript services or a future
  Rust pipeline.

When `documentKnowledgeBuiltinsEnabled` is `true`:

- register the document tools with `ToolRegistry.register` like other model-facing
  built-ins;
- include their names in the merged LLM tool list;
- reserve their names through `builtinToolDefinitionNames()` for extended-tool
  collision checks.

`BuiltInTools.registerAll` should branch on the setting and **skip the document tools
entirely** when disabled. Implementation and tests must cover both states.

## 5. Tool: `search_document_chunks`

### 5.1 Purpose

Find document chunks relevant to a user question or normalized health issue.

This tool returns matching metadata and snippets. It should not return full
chunk markdown by default. The model should call `get_document_chunk` for the
best 1-3 matches it intends to cite.

### 5.2 Input schema

```json
{
  "type": "object",
  "properties": {
    "query": {
      "type": "string",
      "description": "Natural-language query built from the user question or normalized health issue."
    },
    "signals": {
      "type": "array",
      "description": "Optional alarms, properties, symptoms, or components.",
      "items": {
        "type": "object",
        "properties": {
          "kind": {"type": "string"},
          "name": {"type": "string"},
          "value": {"type": "string"}
        }
      }
    },
    "assetContext": {
      "type": "object",
      "description": "Optional asset context such as asset model, component, or document type hints."
    },
    "documentTypes": {
      "type": "array",
      "items": {"type": "string"},
      "description": "Optional document type filters such as operations_manual or troubleshooting_guide."
    },
    "documentIds": {
      "type": "array",
      "items": {"type": "string"},
      "description": "Optional explicit document id filter. When resolve_document_set returns a non-empty documents[], pass those documents[].documentId values here to scope this search to the resolved set (selectionMode documentIds-filter)."
    },
    "limit": {
      "type": "integer",
      "description": "Maximum matches to return. Clamped to configured bounds."
    }
  },
  "required": []
}
```

`query` is optional when useful `signals` or `assetContext` fields are present.
If all are empty, return an empty success with a warning. `documentIds` is the
schema-backed handoff for the explicit resolver-tool path: when `resolve_document_set`
returns a non-empty `documents[]`, pass those ids here to scope the search to that set
(see §3.4 rung 2 of `docs/operations/knowledge-retrieval-pipeline.md`).

### 5.3 Success result

```json
{
  "status": "success",
  "degraded": false,
  "matches": [
    {
      "docId": "earthly-cici-ops-v2",
      "chunkId": "troubleshooting-chiller-high-pressure-bpr",
      "heading": "Chiller High Pressure Shutdown - back pressure regulator",
      "sectionPath": ["9. Troubleshooting Tips"],
      "contentType": "troubleshooting",
      "pageStart": 25,
      "pageEnd": 25,
      "score": 91,
      "snippet": "Probable causes: Back pressure regulator bumped. Recommended actions: Adjust BPR...",
      "sourceLinks": [
        {
          "label": "Earthly Labs manual, 9. Troubleshooting Tips, page 25",
          "repository": "AIDocRepository",
          "path": "/document-knowledge/earthly-cici-ops-v2/source/original.pdf",
          "page": 25,
          "href": "/Thingworx/FileRepositories/AIDocRepository/document-knowledge/earthly-cici-ops-v2/source/original.pdf#page=25"
        }
      ]
    }
  ],
  "warnings": [],
  "searchedDocuments": 1,
  "searchedChunks": 45,
  "skippedDocuments": 0,
  "skippedChunks": 0,
  "documentScores": [
    {
      "docId": "earthly-cici-ops-v2",
      "score": 45,
      "matchedEvidence": ["manufacturer:Earthly Labs", "asset:CiCi CO2 Capture Solution"]
    }
  ],
  "selectedDocIds": ["earthly-cici-ops-v2"],
  "selectionMode": "hard-single"
}
```

Optional diagnostic fields (`documentScores`, `selectedDocIds`, `selectionMode`) are
emitted by the Java scorer for review and live smoke. They are bounded (document
count and evidence strings capped) and safe to ignore by older clients.

When `documentIds` is supplied, unknown ids are omitted from `selectedDocIds` and
may yield empty `matches` with a success envelope (not a turn failure).

### 5.3.1 Manifest `documentProfile` (optional)

Packages MAY include a `documentProfile` object on `manifest.json` for deterministic
document-level scoring (`docs/operations/document-retrieval-stability.md`):

```json
{
  "documentProfile": {
    "aliases": ["KK&K operating manual"],
    "manufacturers": ["KK&K"],
    "assetModels": ["CA 36 GT5"],
    "documentKinds": ["operating_manual"],
    "domainTerms": ["trouble", "elimination", "bearing"],
    "profileSource": {
      "aliases": "curated-identity",
      "domainTerms": "derived-from-toc-headings-and-section-headings"
    }
  }
}
```

When `documentProfile` is absent, the scorer falls back to `docId`, `title`,
`documentType`, `assetModels`, and `sourceFileName`. Conversion tooling SHOULD
derive `domainTerms` from headings — not from acceptance prompts.

Legacy success shape (diagnostics omitted for brevity in older examples):

```json
{
  "status": "success",
  "degraded": false,
  "matches": [],
  "warnings": [],
  "searchedDocuments": 1,
  "searchedChunks": 45,
  "skippedDocuments": 0,
  "skippedChunks": 0
}
```

Search snippets must be bounded to `documentKnowledgeSearchSnippetMaxChars`. When
truncating, prefer word boundaries when practical and never return full chunk
markdown from search.

### 5.4 Output budgets

To avoid quiet token amplification:

- `snippet` length is capped by `documentKnowledgeSearchSnippetMaxChars`.
- `get_document_chunk` `markdown` is capped by `documentKnowledgeChunkMaxChars`
  (see §6.3).
- `warnings` must be compact and aggregated per §8; do not emit hundreds of
  per-line warnings.
- `matches` length is clamped to the effective search limit.

Add explicit tests for generated `search_document_chunks` parameter schema
(provider compatibility), empty-input behavior, snippet truncation, warning
aggregation, and `get_document_chunk` truncation.

### 5.5 Degraded success

Most failures should return success with no matches and warnings:

```json
{
  "status": "success",
  "degraded": true,
  "matches": [],
  "warnings": [
    {
      "code": "DOCUMENT_REPOSITORY_NOT_CONFIGURED",
      "message": "Document knowledge repository is not configured."
    }
  ],
  "searchedDocuments": 0,
  "searchedChunks": 0,
  "skippedDocuments": 0,
  "skippedChunks": 0
}
```

The user-facing agent answer can continue from live health evidence even when
document search is unavailable.

## 6. Tool: `get_document_chunk`

### 6.1 Purpose

Fetch the full markdown and source provenance for one chunk selected by search.

### 6.2 Input schema

```json
{
  "type": "object",
  "properties": {
    "docId": {"type": "string"},
    "chunkId": {"type": "string"}
  },
  "required": ["docId", "chunkId"]
}
```

### 6.3 Success result

```json
{
  "status": "success",
  "degraded": false,
  "docId": "earthly-cici-ops-v2",
  "chunkId": "troubleshooting-chiller-high-pressure-bpr",
  "heading": "Chiller High Pressure Shutdown - back pressure regulator",
  "sectionPath": ["9. Troubleshooting Tips"],
  "contentType": "troubleshooting",
  "pageStart": 25,
  "pageEnd": 25,
  "markdown": "## Chiller High Pressure Shutdown - back pressure regulator\n\nProbable causes:\n- Back pressure regulator bumped\n\nRecommended actions:\n- Adjust BPR...",
  "sourceLinks": [
    {
      "label": "Earthly Labs manual, 9. Troubleshooting Tips, page 25",
      "repository": "AIDocRepository",
      "path": "/document-knowledge/earthly-cici-ops-v2/source/original.pdf",
      "page": 25,
      "href": "/Thingworx/FileRepositories/AIDocRepository/document-knowledge/earthly-cici-ops-v2/source/original.pdf#page=25"
    }
  ],
  "warnings": []
}
```

If `markdown` exceeds `documentKnowledgeChunkMaxChars`, return a bounded prefix
and include a warning:

```json
{
  "code": "CHUNK_MARKDOWN_TRUNCATED",
  "message": "Chunk markdown was truncated to the configured maximum."
}
```

### 6.4 Not found result

`get_document_chunk` may return a structured error because the model asked for a
specific object. It must not throw an unhandled exception through the agent loop.

```json
{
  "status": "error",
  "code": "CHUNK_NOT_FOUND",
  "message": "Document chunk was not found.",
  "docId": "earthly-cici-ops-v2",
  "chunkId": "missing"
}
```

## 6.5 Tool: `resolve_document_set`, the overridable resolver, and host-context scoping

Design: `docs/operations/knowledge-retrieval-pipeline.md` §3. Gated by
`documentKnowledgeBuiltinsEnabled` with the other document tools.

**`resolve_document_set(key STRING)`** returns the bounded document set that applies to
an asset/Thing `key`:
`{ status, resolverSource, documents: [ { documentId, alwaysInclude, appliesToMany } ] }`.
The model passes the resolved `documents[].documentId` into `search_document_chunks`'s
`documentIds` (§5.2) to scope retrieval. Empty `documents` (`resolverSource:
default-empty`) ⇒ no confident scope; proceed with a normal search.

**`resolverSource` diagnostic:** `custom | default-match | default-empty | none`.

**Cross-cutting documents (`alwaysInclude` / `appliesToMany`).** Each row carries two
boolean flags (design §3.5). `alwaysInclude` marks a cross-cutting safety/general
document that applies regardless of the asset key; the runtime **unions** every
`alwaysInclude` document into the scoped `documentIds` even when it sits outside the
key-matched set, so it is always retrievable — sourced from the resolver, **never** from
unbounded global search. `appliesToMany` is parsed and surfaced on the wire but is
**inert** in the current runtime (a classifier with no separate effect yet). Cross-cutting
docs are **resolver-sourced only**: the built-in default matcher never sets either flag,
so they enter scope solely through a custom `ResolveDocumentSet` override. Built-in
`default-match`/`default-empty` rows always report both flags `false`.

**Overridable `ResolveDocumentSet(key STRING): INFOTABLE<ResolvedDocument>` service**
(on the AgentThing, `isAllowOverride = true`). App developers override it to map a key
(typically a bound ThingName) to documents. The Java default returns an **empty
`ResolvedDocument` table** — the sentinel for "no custom mapping." The runtime invokes
it first; a non-empty result (≥ 1 valid `documentId`) ⇒ `resolverSource: custom`, else
the agent falls through to its built-in high-confidence matcher (`default-match` when
the key exactly matches a manifest `documentProfile.assetModels[]`/`assetModels[]`
entry, else `default-empty`). An override that returns no usable rows is treated as
"no custom mapping" and reported `default-*`, not `custom`.

**Host-context server-side scoping (§3.2).** When a turn's host context was accepted via a
**registered template** (`outcome: ACCEPTED` only — **not**
`UNREGISTERED_GENERIC_FALLBACK` / `genericFallback: true`) and carries
`context.thingName`, the runtime resolves the document set at turn start (same
override-first → built-in matcher) and, on a non-empty result, **auto-defaults**
`search_document_chunks.documentIds` for that turn — so the *first* search is already
scoped without the model calling `resolve_document_set`. This is a **default, not a hard
filter**: it applies only when the model omits `documentIds` from the call (the model
supplying `documentIds` at all, including `[]`, suppresses it), and it never turns an
intent-less search into a scoped one. Scoped searches report
`documentScopeSource: host-context-resolver` plus the carried
`documentScopeResolverSource`. Everything fails open: absent/rejected host-context,
blank `thingName`, tools disabled, an unavailable index, or an empty resolve all leave
the turn unscoped. Because the built-in matcher compares the key by exact string
equality, host-context `default-match` only fires when `thingName` happens to equal a
curated `assetModels` entry; the **custom override** is the intended ThingName→documents
path, with fail-open `default-empty` otherwise.

## 7. FileRepository package discovery

V1 discovery is deliberately simple:

1. Read `documentKnowledgeRepository`.
2. Read `documentKnowledgeRootPath`, default `/document-knowledge`.
3. List immediate child folders under the root.
4. For each child folder, try `<root>/<docId>/manifest.json`.
5. If the manifest is usable, read the configured `chunksPath`.
6. Parse `chunks.jsonl` line by line.

The implementation must tolerate partial packages. One bad package must not
block all document search.

The fixture maps as follows:

| Concept | Value |
| --- | --- |
| Repository Thing | `AIDocRepository` |
| Repository root | `dev_data/future_repo/` in git, `/` in the FileRepository |
| Package root | `/document-knowledge/earthly-cici-ops-v2/` |
| Source PDF path | `/document-knowledge/earthly-cici-ops-v2/source/original.pdf` |
| Page 25 link | `/Thingworx/FileRepositories/AIDocRepository/document-knowledge/earthly-cici-ops-v2/source/original.pdf#page=25` |

## 8. Fault tolerance

The tools must be extremely tolerant. File, repository, JSON, and shape errors
should not stop the agent from answering.

| Failure | Behavior |
| --- | --- |
| Repository not configured | `search`: empty degraded success. `get`: structured error `DOCUMENT_REPOSITORY_NOT_CONFIGURED`. |
| Repository Thing cannot be resolved | Degraded result with `DOCUMENT_REPOSITORY_UNAVAILABLE`. |
| Root path missing | Empty degraded success with `DOCUMENT_ROOT_NOT_FOUND`. |
| Manifest missing | Skip that package and add `MANIFEST_MISSING`. |
| Manifest JSON invalid | Skip that package and add `MANIFEST_INVALID_JSON`. |
| Required manifest fields missing | Skip that package and add `MANIFEST_INVALID_SHAPE`. |
| `chunks.jsonl` missing | Skip that package and add `CHUNKS_FILE_MISSING`. |
| `chunks.jsonl` cannot be read | Skip that package and add `CHUNKS_FILE_READ_ERROR`. |
| One JSONL line invalid | Skip that line and add/increment `CHUNK_LINE_INVALID_JSON`. |
| One chunk shape invalid | Skip that line and add/increment `CHUNK_INVALID_SHAPE`. |
| Source PDF missing | Keep chunk result; source link may still be returned because the target path is known. Add `SOURCE_FILE_NOT_VERIFIED` if verification is attempted and fails. |
| Query empty and no signals/context | Empty success with `EMPTY_SEARCH_INPUT`. |
| Limit invalid | Clamp to configured bounds and add `LIMIT_CLAMPED`. |
| Index build exceeds max docs/chunks | Build partial index, set `degraded: true`, add `INDEX_LIMIT_REACHED`. |
| Unexpected exception | Catch at tool boundary, log, return degraded empty search or structured get error `DOCUMENT_TOOL_INTERNAL_ERROR`. |

Warnings should be compact and bounded. Do not return hundreds of per-line
warnings to the LLM. Aggregate repeated warnings:

```json
{
  "code": "CHUNK_LINE_INVALID_JSON",
  "message": "Some chunk lines were skipped because they were invalid JSON.",
  "count": 3
}
```

## 9. Java implementation components

Proposed classes:

```text
DocumentKnowledgeToolSchemas
DocumentKnowledgeToolsExecutor
DocumentKnowledgeRepositoryReader
DocumentKnowledgePackageManifest
DocumentKnowledgeChunk
DocumentKnowledgeIndex
DocumentKnowledgeSearchScorer
DocumentKnowledgeLinkBuilder
DocumentKnowledgeWarnings
```

Responsibilities:

| Class | Responsibility |
| --- | --- |
| `DocumentKnowledgeToolSchemas` | OpenAI/Azure tool parameter schemas. |
| `DocumentKnowledgeToolsExecutor` | Dispatch `search_document_chunks`, `get_document_chunk`, and `resolve_document_set`. |
| `DocumentKnowledgeRepositoryReader` | FileRepository listing and text reads. |
| `DocumentKnowledgePackageManifest` | Parse and validate `manifest.json`. |
| `DocumentKnowledgeChunk` | Parse and validate chunk JSONL rows. |
| `DocumentKnowledgeIndex` | JVM cache of manifests, chunks, and lookup maps. |
| `DocumentKnowledgeSearchScorer` | Deterministic metadata + lexical scoring. |
| `DocumentKnowledgeLinkBuilder` | Build FileRepository PDF links with `#page=N`. |
| `DocumentKnowledgeWarnings` | Bounded warning aggregation. |

`BuiltInTools.registerAll` must apply §4.1: when
`documentKnowledgeBuiltinsEnabled` is `false`, none of the document-knowledge
classes above register tool names in `ToolRegistry`.

## 10. Cache and index lifecycle

V1 should avoid reading every file on every tool call.

Recommended cache:

```text
key: repository Thing name + root path
value:
  loadedAt
  expiresAt
  manifestsByDocId
  chunksByDocAndChunkId
  searchableChunks
  warning summary from last load
```

Behavior:

- Lazy-load on first search/get.
- Reuse until TTL expires.
- On TTL expiry, try to rebuild synchronously for v1.
- If rebuild fails and a previous index exists, return results from the stale
  index with warning `INDEX_REBUILD_FAILED_USING_STALE`.
- If rebuild fails and no previous index exists, return degraded empty search.
- Bound documents and chunks using configuration.

Future options:

- explicit `RefreshDocumentKnowledgeIndex` service;
- background refresh;
- checksum-based reload;
- persisted chunk catalog;
- lexical inverted index.

## 11. Scoring

Document search uses a two-phase deterministic scorer
(`DocumentKnowledgeSearchScorer`):

1. **Document score** — identity evidence from query, `signals`, `assetContext`,
   manifest fields, and optional `documentProfile`.
2. **Chunk score** — lexical/metadata scoring (below), plus the parent document
   score as a first-class boost.

When multiple documents remain plausible, results are **diversified** with a
per-document cap (`max(1, ceil(limit/2))`). Hard single-document selection applies
only for explicit `documentIds` or high-confidence identity matches.

Candidate fields:

- `heading`
- `sectionPath`
- `contentType`
- `tags`
- `signals[].name`
- `summary`
- `markdown`
- `assetModels` / manifest `documentProfile`
- `documentType`
- `assetContext` scalar/string values (threaded into document scoring)

Initial chunk scoring:

| Match | Score |
| --- | ---: |
| Exact signal name match | +50 |
| Exact heading phrase match | +35 |
| Tag/component exact match | +25 |
| Document type match | +15 |
| Content type `troubleshooting` for alarm/shutdown queries | +15 |
| Query token in heading | +8 each |
| Query token in summary/tags | +5 each |
| Query token in markdown | +1 each, capped |

Document-level evidence weights (representative):

| Evidence | Weight class |
| --- | ---: |
| Exact alias/title phrase | +100 |
| docId / filename token phrase | +60 |
| Document kind | +40 |
| Asset model phrase | +25 |
| Manufacturer | +20 |
| Derived domain heading term | +10 |

Sort order:

1. score descending;
2. `troubleshooting` before `maintenance` before `section` before `page`;
3. semantic chunks before page chunks for equal score;
4. stable `docId`, then `chunkId`.

Search success responses include bounded `documentScores`, `selectedDocIds`, and
`selectionMode` for diagnostics (§5.3).

## 12. Deterministic first, embeddings later

V1 should use deterministic metadata and lexical scoring before embeddings.

This is not an anti-embedding position. It is a sequencing decision based on the
health-status + manual-recommendation use case.

A health-status turn usually provides strong structured signals:

```text
alarm: Chiller High Pressure Shutdown
component: chiller
property: Dewar Pressure
assetModel: CiCi CO2 Capture Solution
documentType: operations_manual
contentType: troubleshooting
```

Those signals are better handled by deterministic matching than by broad vector
similarity:

- Exact alarm names should dominate.
- Component/tag matches should be explainable.
- Troubleshooting sections should outrank general page chunks for alarm/shutdown
  questions.
- Heading and section matches should be testable.

Benefits of deterministic-first retrieval:

- **Explainable:** the tool can say a result matched an alarm, tag, heading, or
  section type.
- **Testable:** fixture tests can assert that `Chiller High Pressure Shutdown`
  returns the page 25 troubleshooting chunks.
- **Stable:** results do not drift when an embedding model or vector store
  changes.
- **Small implementation:** Java can start with a FileRepository-backed JVM
  index and a scoring function.
- **Industrial-friendly:** equipment names, alarm names, property names, and
  maintenance terms are often exact identifiers, not fuzzy prose.

Embeddings become useful when the user describes a condition using language that
does not overlap with the manual. Example:

```text
Why is the unit cold but not making liquid?
```

The relevant manual row is:

```text
Negative Temperature but no liquid being generated
```

Lexical matching may be weak if the query lacks "negative temperature" or "no
liquid generated". Embeddings can help recover that candidate.

### 12.1 Future embedding roles

Embeddings should be an optional enhancement behind structured filters, not the
first retrieval layer.

Preferred uses:

1. **Recall expansion:** when deterministic results are empty or low-confidence,
   run an embedding search inside the already-filtered document set.
2. **Reranking:** after deterministic retrieval finds a bounded candidate set,
   use embedding similarity as a secondary signal.

Avoid this shape:

```text
user query -> whole corpus vector search -> final top chunks
```

Prefer this shape:

```text
asset/doc filters
  -> deterministic lexical candidates
  -> optional embedding recall/rerank
  -> deterministic boosts and final top chunks
```

One possible scoring blend for a later phase:

```text
finalScore = deterministicScore * 0.75 + embeddingScore * 0.25
```

The exact weights should be based on evaluation results, not guessed in the
first implementation.

### 12.2 Embedding artifacts

If embeddings are added, the package or index needs versioned embedding
metadata. Do not hide model identity.

Possible chunk fields:

```json
{
  "embeddingText": "Chiller High Pressure Shutdown. Probable causes...",
  "embeddingModel": "text-embedding-...",
  "embeddingVersion": "2026-xx",
  "embeddingVectorRef": "index/embeddings.jsonl#..."
}
```

Possible sidecar index:

```text
index/
  embeddings.jsonl
```

Example row:

```json
{
  "docId": "earthly-cici-ops-v2",
  "chunkId": "troubleshooting-negative-temperature-no-liquid",
  "embeddingModel": "text-embedding-...",
  "embeddingVersion": "2026-xx",
  "vector": [0.0123, -0.0456]
}
```

For production-scale deployments, vectors may belong in a search index rather
than inside the FileRepository package. The portable package contract should
still retain enough metadata to trace vectors back to `docId` and `chunkId`.

### 12.3 When to add embeddings

Do not add embeddings solely because the document count grows. Add them when
quality evidence shows deterministic retrieval is insufficient.

Reasonable triggers:

- Evaluation top-3 hit rate for real health/manual questions is too low.
- Users often describe symptoms in words that differ from manual headings.
- The document set expands beyond manuals into service bulletins, field notes,
  SOPs, training slides, or technician logs.
- Lexical ranking returns too many similarly named chunks and deterministic
  tie-breakers are not enough.

Recommended rollout:

1. Build deterministic search.
2. Create an evaluation set from real health-status questions and expected
   document sections.
3. Measure top-1 and top-3 retrieval accuracy.
4. Add embeddings only for recall expansion or reranking when the measurements
   justify it.

The architecture name should remain document-grounded recommendation with
structured chunk retrieval. Embeddings are an optional retrieval implementation
detail.

## 13. Link generation

The link builder must reuse the same FileRepository download rules as table CSV
export (`CONTRACTS/TABLE_CONTRACT.md` §4, `docs/ui/table-view-solution.md` §5.5,
reference implementation `parler-ui/parler-ui.js`:
`thingworxFileRepositoriesHref`, `thingworxFileRepositoryDownloaderHref`,
`fileRepoPathNeedsDownloaderQuery`).

Agent `href` values are mashup-relative platform paths (no browser origin
prefix), matching existing table export examples.

### 13.1 Path normalization

Before choosing link form:

- trim `repository` and `path`;
- normalize backslashes to `/` in `path`;
- treat `path` as FileRepository-relative (same semantics as table
  `exportFile`);
- for path-style links, ensure `path` begins with `/` when non-empty;
- for downloader query form, strip a leading `/` from `download-path` (reference
  `parler-ui` behavior).

`path` must point to the original PDF, not markdown or chunk files.

### 13.2 Path-style link (方式 A)

Use when `path` does **not** contain `?` or `#`:

```text
/Thingworx/FileRepositories/{encodeURIComponent(repository)}{encodeURI(path)}
```

Append the PDF page fragment only when `page` is a positive integer:

```text
#page={page}
```

Example:

```text
/Thingworx/FileRepositories/AIDocRepository/document-knowledge/earthly-cici-ops-v2/source/original.pdf#page=25
```

### 13.3 Downloader query link (方式 B)

When `path` matches `/[?#]/`, path-style URLs would mis-parse. Use the table-export
downloader form for the repository/path portion, request inline rendering, then
append the page fragment:

```text
/Thingworx/FileRepositoryDownloader?download-repository={repository}&download-path={pathWithoutLeadingSlash}&directRender=true#page={page}
```

Build query parameters with URL encoding equivalent to `URLSearchParams`
(`download-repository`, `download-path`, `directRender=true`). Do not hand-join
unencoded `?` / `#` into a path-style URL.

`<parler-ui>` also rewrites rendered FileRepository PDF links from 方式 A to this
direct-render downloader form while preserving `#page=N`; the tool output can
keep the stable path-style `sourceLinks[].href`.

### 13.4 Page and verification rules

- `page` should be `pageStart` unless the chunk explicitly has a better page.
- If `page` is missing, zero, or negative, omit `#page=...` but keep the file
  link.
- v1 does **not** verify that the source PDF exists in FileRepository when
  building links; trust manifest/configured paths and avoid extra repository
  reads. If optional verification is added later, failure adds
  `SOURCE_FILE_NOT_VERIFIED` but still returns the link.

## 14. LLM routing and answer rules

The system prompt / routing guide should teach:

- Use live health/status tools first when the user asks about current state.
- Use `search_document_chunks` after a concrete issue, alarm, component, or
  symptom is known.
- Use `get_document_chunk` for the top matches that will affect the answer.
- Do not cite document text unless it came from `get_document_chunk` or a
  returned search snippet.
- Do not invent source links.
- When citing document sources, render each cited source as a markdown link using
  `sourceLinks[].href` from tool results (copy href exactly; include `#page=` when
  present). Plain-text source lines are not clickable in the UI.
- If document search is degraded or empty, say that no matching manual section
  was found and continue from live evidence.

Recommended final answer structure:

```markdown
## Observed status

...

## Manual guidance

...

## Recommendation

...

Sources:
- [Earthly Labs manual, section 9, page 25](/Thingworx/FileRepositories/AIDocRepository/document-knowledge/earthly-cici-ops-v2/source/original.pdf#page=25)
```

## 15. Tests

Minimum tests:

| Test | Expected |
| --- | --- |
| `documentKnowledgeBuiltinsEnabled=false` | None of the document tools registered in `ToolRegistry`; names absent from `builtinToolDefinitionNames()`; extended tools may use the same names. |
| `documentKnowledgeBuiltinsEnabled=true` | All three document tools (`search_document_chunks`, `get_document_chunk`, `resolve_document_set`) registered with `ToolDefinition`s; names present in merged LLM list and `builtinToolDefinitionNames()`. |
| Generated `search_document_chunks` schema | Provider-compatible parameters object (no invalid array `items` omissions). |
| Good fixture search for `Chiller High Pressure Shutdown` | Returns troubleshooting chunk on page 25. |
| Search with exact alarm signal | Signal match outranks broad page chunk. |
| `get_document_chunk` for known chunk | Returns full markdown and FileRepository link. |
| Missing repository config | Search returns degraded empty success. |
| Missing manifest | Package skipped; search still succeeds. |
| Invalid manifest JSON | Package skipped with warning. |
| Missing chunks file | Package skipped with warning. |
| Invalid JSONL line | Bad line skipped; good lines indexed. |
| Invalid chunk shape | Bad chunk skipped. |
| Empty query with no signals/context | Empty success with warning. |
| Limit too large | Limit clamped with `LIMIT_CLAMPED`. |
| Snippet over budget | `snippet` truncated to `documentKnowledgeSearchSnippetMaxChars`. |
| Repeated load warnings | Aggregated warning with `count` (bounded list). |
| `get_document_chunk` over budget | `CHUNK_MARKDOWN_TRUNCATED` at `documentKnowledgeChunkMaxChars`. |
| Link builder path with spaces | Encodes path consistently with table download style (`encodeURI`). |
| Link builder path without leading `/` | Normalized to leading `/` for path-style links. |
| Link builder path with `?` or `#` | Uses downloader query form; leading `/` stripped from `download-path`. |
| Link builder missing/invalid page | `href` omits `#page=...`. |
| Cache rebuild failure with stale index | Uses stale index with warning. |

Fixture:

```text
dev_data/future_repo/document-knowledge/earthly-cici-ops-v2/
```

## 16. Scale path

The v1 Java tool can load a bounded number of packages into a JVM cache. This is
enough for Phase 1 and early deployments.

Expected evolution:

1. FileRepository packages + JVM cache.
2. Chunk catalog table or persisted index.
3. Lexical inverted index and better ranking evaluation.
4. Optional embeddings only if deterministic ranking is not good enough.

The built-in tool contract should remain stable across those changes.

The same contract should also remain stable if the backing implementation moves
from Java built-ins to external-service wrappers.

## 17. Implementation phases

### Phase A: schemas and fixture reader

- Add tool schemas.
- Add `documentKnowledgeBuiltinsEnabled` AgentThing option, default `false`.
- Apply §4.1 registration semantics (skip the document tools entirely when disabled).
- Add manifest/chunk Java models.
- Add fixture-backed unit tests and registration-gating tests.

### Phase B: FileRepository reader and index

- Read manifests and JSONL from configured repository/root.
- Build bounded JVM index.
- Implement warning aggregation.

### Phase C: search and get executors

- Implement `search_document_chunks`.
- Implement `get_document_chunk`.
- Add scoring tests.
- Add link builder tests.

### Phase D: prompt/routing integration

- Add tool routing guidance.
- Add final-answer citation rules.
- Add eval or harness prompt covering health-status + manual recommendation — live suite: **`docs/agent/evals/document_knowledge_v1.yaml`** (gated by **`AGENT_EVAL_HAS_DOCUMENT_KNOWLEDGE=1`**; health-status case also requires **`AGENT_EVAL_DOCUMENT_KNOWLEDGE_LIVE_STATUS=1`** when live alert/property fixtures exist).

### Phase E: hardening

- Add stale-cache fallback — response-level **`INDEX_REBUILD_FAILED_USING_STALE`** when TTL rebuild fails but a prior index exists (§10).
- Add configuration clamps — **`CONFIG_VALUE_CLAMPED`** warnings from **`DocumentKnowledgeSettings.fromAgent`** when AgentThing values are out of bounds.
- Add live diagnostic log lines for index load/rebuild and skipped packages (**`DocumentKnowledgeIndexCache`**, **`DocumentKnowledgeIndex.load`**).

## 18. Resolved decisions (v1)

| Question | v1 decision |
| --- | --- |
| Verify source PDF exists when building links? | **No** — trust manifest/configured paths; avoid extra FileRepository reads (§13.4). |
| `get_document_chunk` lookup by `docId + page`? | **No** — require `docId + chunkId` only; page-level chunks still have stable `chunkId` values in packages. |
| Collapse duplicate same-page search matches? | **No** — return one row per chunk; ranking/tie-breakers handle ordering. |
| Return page chunks only as low-confidence fallback? | **No special rule** — use deterministic scoring and tie-breakers from §11 for all chunk types. |
| Expose index refresh as an AgentThing service in v1? | **No** — TTL rebuild only; operator refresh waits until cache behavior is proven (future §10 option). |

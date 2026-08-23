# Entity hierarchy and Parler

**Status:** Architecture draft (not a normative contract). **Source of truth:** this document.

**Framing:** **Taxonomy** and **Hierarchy** are two **separate, first-class, orthogonal** tracks. A typical analysis flow is **narrow by hierarchy first, then run taxonomy queries within that scope**.

**Type surface:** Structured **`identity-types.json`** and optional **`type-taxonomy.md`** → **`docs/agent/AGENT-TAXONOMY.md`**, **`CONTRACTS/AGENT_TAXONOMY_RENDERING.md`**, **`CONTRACTS/TAXONOMY_RESOLVER.md`**. Contract changes must ship in the same change as code and a bump of **`CONTRACTS/CONTRACT_VERSION.md`** (bundle version and repo release policy aligned; **PR descriptions must call out semantic contract changes explicitly**).

**`idKind` enum (v1):** This document is the **authoritative source for allowed `idKind` values and `UNSUPPORTED_ID_KIND` semantics**; host JSON and **`docs/architecture/host-context.md`** reference this file only and do not duplicate the list.

![Example client-side hierarchy UI](assets/entity-hierarchy-example.png)

---

## 1. ThingWorx working assumption

On a single **Entity**, **Property / Service / Event / Subscription** names **must not** collide. There is no “two Description fields compete for precedence” style of branching.

---

## 2. Two tracks

| Track | Question | Typical ThingWorx anchor |
|------|----------|---------------------------|
| **Taxonomy** | **What kind of asset is this**, templates/shapes, **CriticalProperties**, synonyms | **`identity-types.json`** + built-in resolver tools |
| **Hierarchy** | **Where the instance sits in a tree**, subtrees, rollup | Network, parent/child Things, relationship tables, Site, etc. |

**`CriticalProperties`** only drives **which columns appear first in results**; it does **not** define hierarchy resolution.

---

## 3. Context and prompt (v1 checklist)

- **Skeleton / full-tree topology:** **Not** included in LLM-visible context; the **Agent JVM** may cache a skeleton **for tool implementation only** (performance, fewer repeated platform calls). **LLM-visible** subtree membership, node lists, and path details **must** arrive only via **tool results** or **`cacheId` / LARGE** paths. **Do not** silently splice skeleton cache into the prompt to bypass tool-based rules.  
- **System prefix:** Only **very short, stable** hierarchy **meta** is allowed (**no** concrete node-name examples; **do not** narrate the UI’s current scope body—scope data is side-channel per **`host-context.md`**); **target cap ≤ 200 tokens** (tunable, must be auditable).  
- **Subtree membership / node lists / expand row sets:** Small results as **INLINE tool payloads**; above the INLINE threshold, **require LARGE + `cacheId`** (same idea as **`query_entities*`** and **`CONTRACTS/TABULAR_INSIGHT.md`**).

---

## 4. Working model

**Nodes:** **`ThingName`** + **`DisplayName`**. **Edges:** At least a four-tuple (parent/child keys and display).

**Three steps:** (1) Load the **network skeleton** (not the full set of related business Things per node); (2) **Resolve** (natural language → **`id` + `idKind`** + disambiguation); (3) **Expand** (bounded **`thingNames[]` / `cacheId`** + **`hasMore`** + stable sort, **SecurityContext**).

**`idKind` (v1):** Only **`"Thing"`**; any other value → **`UNSUPPORTED_ID_KIND`** (exact literals per **`CONTRACTS/API_CONTRACT.md`**).

**Invalid explicit hierarchy scope** (deleted node, or entity not visible to the current user): **Must** surface as an **explicit error** when the scope was supplied as a tool argument such as **`hierarchyNodeName`** / **`intersectThingNames`**; **do not** silently fall back to pure NL resolution so the user believes scope still applies. Host Context v2 itself is rendered prompt guidance only; it does not perform server-side tool-argument injection.

---

## 5. Error semantics, permissions, and platform

**Invalid / invisible scope:** Use an **explicit error path**; hierarchy errors **per contract** propagate to UI / LLM-visible payloads.

**No permission vs not found:** **Same** user-facing feeling and outward behavior; **do not** split into another user-perceivable branch at the product layer. The Parler Agent focuses on **DataInsight**, **not** a coding assistant; ThingWorx on common paths **often cannot reliably distinguish** “does not exist” from “not visible to the current principal”, so we expose **one unified user-visible story** (error codes per **`CONTRACTS/API_CONTRACT.md`**; implementation may map to a single code such as **`NOT_FOUND`**).

**Error code inventory:** The **full set of hierarchy-related `code` values** lives in **`CONTRACTS/API_CONTRACT.md`** (when finalized, anchor **`hierarchy` / intersection-related errors** there and maintain them). This document only sets **user-visible semantic intent** (including **`UNSUPPORTED_ID_KIND`** and other **`idKind`-related intent**); it does **not** maintain a second code table in parallel with the contract.

**ThingWorx inputs (whether permission / visibility are modeled):** Coding and code-review conventions are in **`docs/agent/AGENT-CONTEXT.md` §5.3.1** (this document does not expand SDK signature details).

---

## 6. v1 phase one: result intersection

**Definition:** Intersect the candidate row set from **`query_entities`** / **`query_entities_by_taxonomy`** with the bounded expand **Thing name set** **B** (**∩**).

**Intersection result envelope:** **Prefer reusing** existing **list-class** tool fields and truncation semantics from **`query_entities*`**; do **not** invent a separate list shape. Intersection is a **scope-shrinking layer**, not a new tool family. **New or differing fields** (e.g. **`preIntersectMatchCount`**, **`intersectedRowCount`**, **`expandHasMore`**, **`queryHasMore`**) are defined **only in `CONTRACTS/API_CONTRACT.md`** in the intersection / list-class sections (anchor when finalized); this document does not list the full field table.

### 6.1 Expand and Mashup

- **Inputs:** **`id` + `idKind`** (and depth, etc.). Host **`context`** hierarchy fields participate in **rendered prompt** only (see **`host-context.md`**); resolve/expand for tools uses explicit **`hierarchyNodeName`** or **`intersectThingNames`** per **`CONTRACTS/API_CONTRACT.md`**.  
- **`path` (`string[]`, host payload):** **Not** passed into resolve/expand **inputs**; for **UI breadcrumbs** and **LLM-visible summaries** only.  
- **Thing names:** Expand, QIT, and intersection all keep **`name`’s UTF-8 byte sequence unchanged** (**no** `toLowerCase` or other folding at the tool layer); **intersection compares with byte `equals`**—names that differ only by case are different entities. If the platform matches case-insensitively on some path, the returned Thing name must still flow through the tool chain **in the platform’s stored casing**.  
- **v1 shipped tool parameters / JSON field definitions:** **`CONTRACTS/API_CONTRACT.md`** § **`query_entities` / `query_entities_by_taxonomy` — expand intersect** (**`intersectThingNames`**, **`preIntersectMatchCount`**, **`intersectedRowCount`**, **`queryHasMore`**, **`expandHasMore`**, **`hasMore`** = OR; see §6.4).

### 6.2 `query_entities` intersection

Intersect the **QIT** row set with set **B** on **`name`**.

### 6.3 `query_entities_by_taxonomy` intersection

Intersect **`rootEntityList`** with **B** on **`name`**.

### 6.4 Two-sided truncation, counts, and `hasMore`

When **B′⊆B** and **A′⊆A**, **A′∩B′** need not equal **A∩B**; the intersection is the intersection of **what each side has returned so far**.

- **`hasMore`:** **`hasMore` = Boolean(`expandHasMore`) OR Boolean(`queryHasMore`)`**. If a side **did not run**, that side’s **`*HasMore` = `false`** (does not contribute “more behind”). If a side **failed**, that side’s **`*HasMore` = `null`** (treat as **`false`** for OR; logs may distinguish); optionally add **`expandStatus` / `queryStatus`** enums for troubleshooting.  
- **Counts (must be defined in `API_CONTRACT`; do not silently reuse old `totalCount` semantics):** Pre-intersection hit count **`preIntersectMatchCount`** (**`query_entities`** and **`query_entities_by_taxonomy`** **share** the field name; values are per-tool projection); post-intersection row count **`intersectedRowCount`**.

**`CONTRACT_VERSION.md`:** bundle bump **level and rationale** must be explicit in the contract PR (if the repo only allows patch, then **patch + PR text stating “semantic change”**).

### 6.5 Boundary with host documentation

- **`host-context.md`:** **`key + context`**, registered templates, rendered prompt (advisory scope for the LLM; **no** server-side tool auto-binding).
- **This document:** **`id` + `idKind`**, resolve/expand, intersection, Network/description.

---

## 7. Two network APIs + candidate service names

| Capability | Meaning | **Candidate service names** (implementation may tweak; must be override-friendly) |
|------------|---------|-----------------------------------------------------------------------------------|
| **Related Things** | Business Things attached at a node | **`GetHierarchyRelatedThings`** |
| **Child network nodes** | Child node list | **`GetHierarchyChildNodes`** |

Same style as **`GetHierarchyPresentationDefaults`** (§8); **`processServiceRequestDirect`** calls the **effective** implementation.

### 7.1 Five overridable service surfaces (v0 authoritative surface)

**Normative service names, parameters, row-count assumptions, and boundary vs PTCTS reference JS** are in **`docs/architecture/hierarchy-network-services.md`** (**`GetFlattenNameDescription`**, **`ResolveNetworkID`**, **`GetRootNode`**, **`GetAssetList`**, **`GetChildNodes`**); orchestration with **`query_entities*`** and **`intersectThingNames`** and **empty results** are in the same doc **§6**. The §7 table entries **`GetHierarchyRelatedThings`** / **`GetHierarchyChildNodes`** remain **candidate aliases**; production may use **stable names** from **`hierarchy-network-services.md`** and **override** them on the customer Thing.

---

## 8. Network name and description: configuration + override; LLM does not participate

**Normal coverage:** 1, 2. **Edge fallbacks:** 3, 4, 5 (including **unparseable override response JSON**). **Forbidden:** 6.

1. **Configuration defaults:** **`hierarchy.networkName`**, **`hierarchy.descriptionPropertyName`** (may be empty).  
2. **Override service** (e.g. **`GetHierarchyPresentationDefaults`**): return value **overrides configuration field-by-field**.  
3. **Override returns `null` / empty string for a single field (valid JSON):** treat as **that field not overridden** → **fall back to configuration default** for that field; **do not** drop the whole override because one field is empty.  
4. **Effective `networkName`:** **must not** be an empty string; if override + config merge is still empty → **fall back to configuration default** (if config is also invalid, treat like **service failure**) + **one aggregated warning**.  
5. **Service failure / timeout**, or **override call succeeds but response JSON does not parse to the agreed schema** (missing fields, wrong types, etc.) → treat like **service failure**: **fall back to configuration default** + **one aggregated warning**.  
6. **The LLM** must not supply these two values via tools.

**Multiple root Networks:** Configuration uses **v1 single default `networkName`**; multiple roots are selected via rendered host context + explicit tool args (**`host-context.md`**).

---

## 9. Child nodes vs related Things

### 9.1 Child nodes

**INLINE**, **`maxChildNodes = 200`**; beyond that → **same schema truncation + `hasMore` paging**.  
- **Sort (v1 fixed):** ascending by **`id` UTF-8 byte order**.  
- **Assumption:** Thing names in customer environments are **mostly ASCII**; if **`id` later includes non-ASCII** and must align strictly with **Unicode code-point order**, **revisit** the sort key (UTF-8 byte order and code-point order are **not** equivalent in edge cases).  
- **Paging:** **`afterId`** = last row **`id`** from the previous page (**not** a pure offset, to avoid drift when the tree changes).  
- **v1 does not** use LARGE/cache for this path.

### 9.2 Related Things

**EntityList (`name` + `description`)**; **INLINE/LARGE** matches **`query_entities*`** and **`CONTRACTS/API_CONTRACT.md`**, **`CONTRACTS/TABULAR_INSIGHT.md`** **list-class / `cacheId`** rules (thresholds and field names are defined in contracts; this document does not invent numbers).

**`description`:** If no extended property is configured → **Thing entity Description**; if a **single Property name** is configured → read that property; on failure or empty → **`""` (v1 fixed)** + **aggregated warning**. Rendering `"—"` in the UI is a presentation concern.

---

## 10. Mashup host background

**Authoritative reference:** **`docs/architecture/host-context.md`** (rendered prompt); tool intersect semantics in **`CONTRACTS/API_CONTRACT.md`**.

---

## 11. Non-goals and deferrals

Network **version/fingerprint**; LLM switching Network/description.

**Bulk `thingNames[]` (other built-in tools):** Revisit after intersection stabilizes; **example objective to reopen:** production logs show the same turn repeatedly calling **single-Thing** tools against **the same subtree membership** above a threshold, then reassess bulk APIs.

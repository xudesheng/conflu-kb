# Extended LLM tools (configuration repository)

**Supersedes:** the old `_tool_*` prefix-scan model on `AgentThing`. Custom capabilities exposed to the model are **not** discovered by scanning Services whose names start with `_tool_`.

## Current model

- Configure **`AgentSettings.configurationRepository`** on the AgentThing to a **FileRepository** Thing.
- Author **`/tools/extended_tools.json`** in that repository (see **`./configuration-repository.md`**).
- Each entry maps an **LLM tool name** to a **concrete Thing** + **service** the agent may invoke (subject to HITL and PASSWORD discovery rules documented there).

Built-in tools remain implemented in Java (`BuiltInTools`, `ToolRegistry`). The merged tool list for each turn is **built-ins** plus **registered extended tools** from the prompt-context snapshot — not prefix-harvested Services.

### Optional `executorOnly`

Per entry in **`/tools/extended_tools.json`**, an optional boolean **`executorOnly`** (default **`false`**) means: register the tool for **execution** (same target service and HITL semantics) but **omit** it from the **merged LLM tool list**. When the field is present it **must** be a JSON boolean; any other type **invalidates the entire manifest** (same strictness as **`hitl`** / **`playbookSafe`**). Normative behavior and snapshot reporting: **`./legacy-discovery-executor-only.md`**; repository operator notes: **`./configuration-repository.md`** (`GetAgentRuntimeSnapshot`).

## When INFOTABLE shaping is awkward

If **`invoke_service`** or schema generation rejects an INFOTABLE parameter (VARIANT, depth, unresolvable shape, etc.), the error message includes a hint to implement a **normal** ThingWorx service with a supported parameter layout and register it through **`/tools/extended_tools.json`** instead of relying on removed `_tool_*` patterns.

## References

- **Normative layout and policies:** **`./configuration-repository.md`**
- **`invoke_service` design:** **`./invoke_service_design.md`**
- **Agent architecture:** **`./AGENT-CONTEXT.md`**

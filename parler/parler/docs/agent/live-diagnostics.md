# Live runtime diagnostics — pulling ApplicationLog + AgentMessageStream

When a manual smoke test fails (or any production-side question comes up), you can pull server-side log, the conversation stream, **AgentThing runtime status**, and bounded raw **configurationRepository** files against the live dev server without GUI access. Use this any time the user says "the prompt failed" / "it didn't work" / "请帮我抓 log" — don't make them paste large logs into chat.

## Primary bundle — `parler-collect-live` (recommended)

One command writes three schema-versioned JSON files plus per-AgentThing repository-file evidence under a new UTC timestamped directory (see **`docs/agent/collection-tool.md`** for the full contract):

```bash
uv run parler-collect-live --window 1h --conversation-id <threadOrSource> -o logs
```

- Reuses **`test_scripts/agent_eval.py`** HTTP helpers (`DEV_SERVER` / `DEV_KEY` from process env + repo-root **`.env`**).
- Stream read prefers **`QueryStreamEntriesWithData`** with **`QueryStreamData` fallback** (warning recorded in **`agent-message-stream.json`**).
- Agent status uses **`GetAgentRuntimeSnapshot`** with **`refresh:false`** and opts into **`includePlaybooks`** / **`includeRepositoryFiles`**; **`ValidateAgentConfigurationRepository`** and **`GetTaxonomyDiagnostics`** are included per agent.
- **`agent-status.json`** includes **`driftChecks[]`** when repository file hashes differ between loaded snapshot metadata and current repository bytes.
- For each discovered AgentThing with a configured **`configurationRepository`**, the collector downloads bounded raw files from the known first-level folders (`/taxonomies`, `/skills`, `/playbooks`, `/policies`, `/tools`) through ThingWorx **`FileRepositories`** REST URLs. It does **not** refresh the AgentThing and does **not** recursively scan the repository root.

Lower-level scripts below remain valid for targeted pulls and scripting.

## Prerequisites

- `.env` at repo root with `DEV_SERVER` and `DEV_KEY` populated (already there for active dev work).
- `uv` installed. Verify with `which uv` (should be `/Users/desheng/.local/bin/uv` or wherever you installed it).
- `test_scripts/agent_eval.py` and `test_scripts/GetApplicationLog.py` checked into the repo.

If `DEV_KEY` is missing from `.env`, ask the user — don't guess paths.

## Recipe 1 — Pull ApplicationLog for a time window

`test_scripts/GetApplicationLog.py` wraps `QueryLogEntries` on `Thingworx/Logs/ApplicationLog`. The wrapper is exposed as the `get-application-log` script.

```bash
uv run get-application-log \
  --start-date 2026-05-25T13:11:50.000Z \
  --end-date 2026-05-25T13:13:00.000Z \
  --max-items 180 \
  --no-indent > /tmp/parler-app-log-<timestamp>.json
```

Tips:

- `--start-date` / `--end-date` are ISO-8601 UTC with `Z` suffix. Time the window tight around the user-reported moment — `maxItems` caps results, and a wide window can drop the row you need.
- Default `--max-items` is 100; bump to 500–2000 for busy turns.
- The server may rotate logs aggressively; if you query "last 4 hours" you may still only get the last ~20 minutes. Page back in 30-minute chunks if needed.
- Output is `{"meta": {...}, "rows": [...]}` per `slim_rows` in the script.

## Recipe 2 — Filter ApplicationLog to agent-relevant lines

```bash
uv run python - <<'PY'
import json, re
p = "/tmp/parler-app-log-<timestamp>.json"
data = json.load(open(p))
need = re.compile(
    r"(ParlerStreamToRemoteThing|Agent loop iteration|Running tool|Tool .* returned|"
    r"extended tool|LLM_|Azure OpenAI request|Agent completed|AgentMessageStream append|"
    r"chart|CHART|ERROR|Exception|failed|returned in)",
    re.I,
)
for r in reversed(data.get("rows", [])):
    c = r.get("content", "")
    if need.search(c):
        print(f"{r.get('timestamp')} [{r.get('thread')}] {c}")
PY
```

What to look for, in order:

1. `ParlerStreamToRemoteThing start ... requestId=<uuid>` — confirms the turn started and gives you the request id.
2. `Agent loop iteration N/10` — every iteration boundary; helps count rounds.
3. `Azure OpenAI request: ... messages=M tools=T` — `tools=0` here is the "post-marker no-tool round" symptom from review-5; `tools=1` with `schemaTools=build_chart_from_tabular_result` is a chart-rescue round.
4. `Running tool from LLM: <name>` followed by `Executing tool: <name> argsPreview=...` — what the model actually called and with what args.
5. `LLM_TOOL_SCHEMA_USAGE` — what schema was sent on this round, plus `calledCount` / `idleCount`.
6. `LLM_TURN_PERFORMANCE` — terminal performance line, includes `chartExpectedButMissing`, `chartRescueAttempted`, `noToolFinalAnswerApplied`, etc.
7. `Agent completed after N iteration(s)` + `ParlerStreamToRemoteThing done status=SUCCESS|ERROR` — turn end.

## Recipe 3 — Query AgentMessageStream for a conversation

This gives you the user prompts, assistant tool_calls JSON, tool result JSON, and final assistant content with `llmUsageJson`.

```bash
uv run python - <<'PY'
import json
from test_scripts.agent_eval import (
    ENV_PATH,
    build_thing_service_url,
    extract_rows_from_service_result,
    load_dotenv,
    post_json,
    require_env,
)

load_dotenv(ENV_PATH)
server = require_env("DEV_SERVER")
key = require_env("DEV_KEY")
url = build_thing_service_url(server, "AgentMessageStream", "QueryStreamData")
payload = {"maxItems": 20, "source": "demo_conversationId", "oldestFirst": False}
data = post_json(url, key, payload, timeout_s=120)
rows = extract_rows_from_service_result(data)
rows.reverse()

for i, r in enumerate(rows):
    print("\n--- ROW", i, "---")
    for k in ["timestamp", "role", "toolCalls", "content", "llmUsageJson", "hostContextSnapshotJson", "assistantMessageId"]:
        v = r.get(k)
        if v in (None, ""):
            continue
        s = json.dumps(v, ensure_ascii=False) if isinstance(v, (dict, list)) else str(v)
        if len(s) > 5000:
            s = s[:5000] + f"...[truncated {len(s)}]"
        print(k + ":", s)
PY
```

- `source` is the conversation id (e.g. `demo_conversationId` for dev runs); ask the user if unclear.
- `maxItems` 20 covers one full turn of 4-6 LLM rounds with multiple tool calls. Bump to 50 for multi-turn investigations.
- The `Thing` name is `AgentMessageStream` (not `data_AgentMessageStream`; that path returns 404 on this server).

What each row contains:

- `role: user` — the user's prompt text in `content`.
- `role: assistant` with `toolCalls` populated — LLM requested tool calls; `content` is usually empty or short.
- `role: assistant` without `toolCalls` — final or intermediate prose; `llmUsageJson` carries `chartExpectedButMissing` / `chartRescueAttempted` on the terminal assistant row.
- `role: tool` — `toolCallId` matches the assistant's prior tool call; `content` is the actual JSON returned to the LLM.
- `role: user` with **`hostContextSnapshotJson`** — per-turn Host Context snapshot (`parler-host-context-snapshot-v1`: `key`, `hash`, `changedFromPreviousUserTurn`, optional anchor `rawJson`). Prefer **`parler-collect-live`** normalized **`hostContextSnapshot`** companion; UI history may omit `rawJson` on unchanged rows (anchor forward-scan — **`docs/architecture/host-context-turn-state.md`** §4.1).

## Recipe 3b — Host Context turn-state checklist

After a stable-`conversationId` smoke (e.g. §7.2 in **`docs/architecture/host-context-turn-state.md`**):

| Check | Where |
| --- | --- |
| Uplink accepted / rejected | ApplicationLog `hostContext rendered key=` or `hostContext … ignored` |
| Snapshot on user Stream row | `hostContextSnapshotJson` on newest `role=user` row (`parler-collect-live` or Recipe 3) |
| `changedFromPreviousUserTurn` | Parsed snapshot on second user turn after page filter change |
| Anchor `rawJson` | Present on changed/anchor user rows; omitted on unchanged rows with matching `hash` |
| History / UI parity | `GetConversationHistoryJson` user row nested `hostContext` matches Stream snapshot |
| Direct-id tool path | Tool args use **`hierarchyNodeId`** (not **`hierarchyNodeName`**) when page supplied node id; tool error must not silent-fallback to unscoped listing |

## Recipe 4 — Interpret the turn

Put both data sources side by side:

| Question | Where to look |
|----------|---------------|
| Did the user actually ask for a chart? | Stream row 0 (`role:user`); product interpretation is **not** encoded in `chartExpectedButMissing` (that flag is behavior-derived: chart builder invoked with zero chart wires so far). |
| What tools were called and in what order? | Stream `assistant.toolCalls` array per round; cross-check `Running tool from LLM` in ApplicationLog |
| Did `tabulate_cached_result` return `answerSetComplete:true`? | Stream `role:tool` rows whose `toolCallId` matches a `tabulate_cached_result` call |
| Did `AgentLoop` schedule a post-marker round (review-5 rescue)? | ApplicationLog `Azure OpenAI request: ... tools=1` immediately after a `answerSetComplete=true` tabular result |
| Did the end-turn chart-rescue fire (review-6/7)? | ApplicationLog: extra `Azure OpenAI request: ... tools=1` after a `finishReason=STOP` round with no tool calls; or terminal `LLM_TURN_PERFORMANCE` shows `chartRescueAttempted=true` |
| Why was no chart emitted? | Cross-reference: did model call `build_chart_from_tabular_result`? Did it error? Was `chartExpectedButMissing=true` AND `chartRescueAttempted=false`? Then check `qualifyingTabularSuccessCount` (multiple tabular successes block rescue per the review-7 count gate) |

## Recipe 5 — Playbook diagnosis (Slice G)

Use **`parler-collect-live`** first. For **`cross_asset_pair_health`**, see the eval pack
[`playbook-eval-pack/cross_asset_pair_health.md`](playbook-eval-pack/cross_asset_pair_health.md).

| Failure mode | Where to look in the bundle |
| --- | --- |
| Agent too old for Playbook | `agent-status.json` → `playbookRuntime.agentVersion` vs converter `requiredAgentVersion` |
| Stale `/playbooks/.../playbook.json` | `driftChecks[]`; compare `repository-files/playbooks/` to loaded metadata |
| Playbook JSON invalid | `playbookDocumentValidations[]` (structured codes per §6.3.1) |
| Node / runner failure | `lastPlaybookRun` on agent status; ApplicationLog `PlaybookRunner` / node ids |
| Missing evidence in answer | `lastPlaybookRun` outcomes; stream final `llm_summary` row vs `evidenceRefs` |
| Wrong tool path | ApplicationLog internal tool calls; compare to eval pack expected path |

Structured validation beats scraping raw `playbook.json` in logs — use `playbookDocumentValidations` and
`ValidatePlaybookDocument` output when authoring.

## Common gotchas

- **Empty result window**: server-side logs may rotate fast on a busy dev server. If your window returns nothing, narrow to a specific minute and increase `--max-items`. Check `meta.returnedRows` to know what you actually got.
- **Wrong conversation id**: dev runs default to `demo_conversationId` but a customer-style demo may use a different one. Check the first row of `ParlerStreamToRemoteThing start remoteThing=<name>` for the actual id.
- **Streamed `done` frames**: the live Parler stream emits `done` over WebSocket, not into `AgentMessageStream`. To see what the UI saw, you need either a browser HAR capture or the persisted assistant row's `llmUsageJson` (which is the sanitized subset of what the UI got).
- **`uv run` first time**: may take a few seconds to resolve dependencies on the very first run; subsequent calls are fast.
- **Local output folder**: in this repo, prefer `-o logs` for ad hoc support bundles; `/logs/` is already ignored by git.

## When NOT to use these recipes

- Don't run repeated polling without need — each call hits the live dev server.
- Don't pull `--max-items 10000` "just in case"; large payloads are slow and consume your context window.
- Don't paste full ApplicationLog output into review docs; extract relevant 3-10 lines.

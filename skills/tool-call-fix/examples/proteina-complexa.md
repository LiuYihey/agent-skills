# Example: Proteina-Complexa

Project-specific walkthrough for the generic [tool-call-fix](../SKILL.md) skill. Replace paths and tool names when copying this template for another repo.

## Symptom archetypes in this codebase

| Tool / surface | Typical failure |
|----------------|-----------------|
| `write_todos` | Modes A, C, F, H — plan sidebar empty |
| `launch_experiment` | Mode B — wrong tool vs plan intent |
| SSE `tool_activity` | Mode D — card late |
| `metadata.llm_messages` | Mode H — turn 2+ degradation |

## Pipeline file map

```
User → Backend/main.py (_generate_sse)
     → agent_loop.stream_agent_turn_events
     → llm_client.stream_complete_with_tools   ← parser / tool_use_start
     → agent_loop tool execution               ← _sort_tool_calls, skip lists
     → executors/*.py                          ← payload validation
     → SSE → Interface/AgentChat.jsx           ← HIDDEN_TOOL_NAMES, tool_activity
     → session.messages (UI text) + metadata.llm_messages (API chain)
     → agent_sessions.db
```

| Concern | Primary file |
|---------|--------------|
| SSE generation | `Backend/main.py` — `_generate_sse`, `_build_design_agent_llm_messages`, `_persist_llm_transcript` |
| LLM transcript | `ProteinHarness/llm_transcript.py` |
| Tool loop | `ProteinHarness/agent_loop.py` — `stream_agent_turn_events`, `_TOOL_ACTIVITY_SKIP` |
| Stream parser | `ProteinHarness/llm_client.py` — `_merge_message_stop_blocks`, `_resolve_tool_use_input` |
| write_todos exec | `ProteinHarness/executors/plan.py`, `ProteinHarness/plan_manager.py` |
| Recovery nudges | `ProteinHarness/action_guard.py` — only when `not turn.tool_calls` |
| Frontend SSE | `Interface/src/components/AgentChat.jsx` |
| Session DB | `agent_sessions.db` |
| Raw SSE trace | `agent_config.json` → `log_sse_trace: true` or `AGENT_LOG_SSE_TRACE=1` |

## Orchestration (`Backend/agent_config.json`)

```json
"orchestration": {
  "force_tool_choice": false,
  "recovery_nudges": true,
  "pending_action_reminders": true
}
```

Implemented in `ProteinHarness/agent_loop.py` → `_orchestration_enabled()` / `_forced_round_tools()`.

**Jun 2026 lesson:** After fixing Mode H (`metadata.llm_messages`), `tool_choice: auto` + `thinking: adaptive` worked without `force_tool_choice`.

## Investigation commands

```bash
# Round summary + raw SSE trace
journalctl -u bindagent.service --since "1 hour ago" | \
  grep -E "Round|tool_activity|assembled blocks|write_todos|sse_trace|MISMATCH"

# Session transcript + LLM-native chain
cd /path/to/Proteina-Complexa && python3 -c "
import sqlite3, json
c=sqlite3.connect('agent_sessions.db'); c.row_factory=sqlite3.Row
r=c.execute('SELECT session_id,messages,plan_steps,metadata_json FROM agent_sessions WHERE session_id LIKE ?',
            ('<prefix>%',)).fetchone()
meta=json.loads(r['metadata_json'] or '{}')
llm=meta.get('llm_messages') or []
print('llm_messages blocks:', sum(1 for m in llm if isinstance(m.get('content'), list)))
for m in json.loads(r['messages'] or '[]'):
    print(m.get('role'), m.get('toolName'), (m.get('content') or '')[:80])
print('plan_steps', r['plan_steps'][:200] if r['plan_steps'] else '[]')
"

rg "write_todos|HIDDEN_TOOL|_TOOL_ACTIVITY_SKIP|_PERSIST_TOOLCALL_SKIP" \
  ProteinHarness Backend Interface
```

## Mode-specific fixes (this repo)

### D — Early tool card

`llm_client` yields `tool_use_start` on `content_block_start` (type=tool_use).  
`agent_loop.stream_agent_turn_events` forwards as `tool_activity` phase=start; dedupe with `stream_announced_tool_ids`.

Expected log order:
```
yielding tool_activity start (stream): write_todos
Round 0 complete: tool_calls=1 ['write_todos'], stop_reason=tool_use
```

### H — LLM transcript persistence

Save `working_messages` to `session.metadata["llm_messages"]` after each turn (`llm_transcript.py`).  
`_build_design_agent_llm_messages` prefers native transcript; legacy sessions fall back to text bootstrap.

**Bug pattern:** persisting only flattened assistant text → `_llm_visible_messages` strips blocks → turn 2+ amnesia.

### C — write_todos payload

- Schema expects `{"todos": [...]}` not `{"todos": {"item": [...]}}`
- `coerce_todos_payload()` in `plan_manager.py`
- Empty payload → explicit executor error
- Emit `ui:plan_updated` + `done.plan_steps` for sidebar sync

### E — Thinking round-trip

`_append_content_block` keeps `thinking` / `redacted_thinking` in `assistant_content`.  
Stream handler accumulates `signature_delta`.

### F — Visibility alignment

| Layer | File | Skip/hide set |
|-------|------|---------------|
| Agent loop | `ProteinHarness/agent_loop.py` | `_TOOL_ACTIVITY_SKIP` |
| Backend persist | `Backend/main.py` | `_PERSIST_TOOLCALL_SKIP` |
| Frontend | `Interface/src/components/AgentChat.jsx` | `HIDDEN_TOOL_NAMES` |

`write_todos` must **not** be in any hide list.

### G — action_guard gaps

- 「建立…待办清单」 did not match `_WRITE_TODOS_INTENT` (only 「已建立」 in completion claim)
- `pick_recovery_nudge` only runs when `not turn.tool_calls` — Mode B bypasses recovery

## Investigation timeline (project history)

1. Symptom: Agent says plan, calls `read_wiki` instead of `write_todos`.
2. Observability gaps: INFO logs not in journal; `write_todos` hidden in 3 layers (Mode F).
3. Parser fixes: Thinking stripped (E); start-block input ignored; `tool_use_start` not forwarded (D).
4. Payload fix: Tool arrived — `{'item': {...}}` rejected (Mode C). Not prose-only (A).
5. Raw SSE trace: `[sse_trace]` proves wire vs parser (`MISMATCH`).
6. False lead: `force_tool_choice` masked transcript loss (H).
7. Root cause: `_llm_visible_messages` flattened history. Fixed with `metadata.llm_messages`.

## Tests

```bash
python3 -m unittest \
  ProteinHarness.test_llm_client \
  ProteinHarness.test_llm_transcript \
  ProteinHarness.test_write_todos_executor \
  ProteinHarness.test_agent_loop_plan_order \
  ProteinHarness.test_action_guard \
  ProteinHarness.test_plan_progress -q
```

## Provider notes (MiniMax / Anthropic-compatible)

- `text_sanitize.py` strips `]<]...[<[` routing artifacts.
- Tool input may arrive on `content_block_start.input` instead of `input_json_delta`.
- Final tool block may appear only on `message_stop.message.content`.

## Post-deploy verification

On **turn 2+** of the same session:

- `metadata.llm_messages` contains list-typed content with `tool_use` / `tool_result` / `thinking`
- `[sse_trace]` shows `tool_choice=null` when `force_tool_choice` is false
- Stream `(stream): tool_name` before `Round N complete`
- `assembled blocks=[('text', None), ('tool_use', 'write_todos')]`

Restart backend after Python changes; CORE.md is cached at process start.

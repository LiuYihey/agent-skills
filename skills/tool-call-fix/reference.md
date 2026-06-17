# Tool Call Fix — Reference

Generic failure-mode catalog (A–H). For stack-specific notes see [frameworks.md](frameworks.md). For a full project walkthrough see [examples/proteina-complexa.md](examples/proteina-complexa.md).

## Failure mode map (A–H)

Same user symptom ("said it would call a tool, nothing happened") can break at **eight different layers**.
Use logs + persisted session data to locate the marker before patching.

```
                              USER MESSAGE
                                   │
                                   ▼
    ┌──────────────────────────────────────────────────────────────┐
    │  G  PROMPT / GUARD                                           │
    │      recovery nudges · pending-action reminders              │
    │      ✗  intent pattern gap → nudge never fires               │
    └──────────────────────────────┬───────────────────────────────┘
                                   │
                                   ▼
    ┌──────────────────────────────────────────────────────────────┐
    │  H  TRANSCRIPT RECONSTRUCTION                                │
    │      UI chat history → text-only API input                   │
    │      native tool/thinking blocks NOT persisted               │
    │      ✗  turn 2+ API sees no tool_use/tool_result/thinking    │
    └──────────────────────────────┬───────────────────────────────┘
                                   │
                                   ▼
    ┌──────────────────────────────────────────────────────────────┐
    │      LLM API  (Anthropic / OpenAI / compatible stream)       │
    │                                                              │
    │   ┌─────────────┐                                            │
    │   │  thinking   │  E ← if stripped on in-turn round-trip    │
    │   └──────┬──────┘                                            │
    │   ┌──────▼──────┐                                            │
    │   │    text     │  A ← prose promises tool; no tool block    │
    │   └──────┬──────┘    stop_reason=end_turn, tool_calls=0      │
    │   ┌──────▼──────┐                                            │
    │   │  tool_use   │  B ← wrong tool name                       │
    │   │  name+input │  C ← right name, bad/missing input         │
    │   └──────┬──────┘                                            │
    └──────────┼───────────────────────────────────────────────────┘
               │
               ▼
    ┌──────────────────────────────────────────────────────────────┐
    │  E  STREAM PARSER                                            │
    │      content_block_start / delta / stop                      │
    │      ✗  thinking/signature dropped from assistant message    │
    │      ✗  tool input on start-block ignored → silent {}        │
    └──────────────────────────────┬───────────────────────────────┘
                                   │
               ┌───────────────────┴───────────────────┐
               │  D  STREAM → UI BRIDGE (timing)       │
               │  text/thinking → UI immediately       │
               │  tool_use      → UI often delayed     │
               │      ✗  early tool_start not forwarded  │
               └───────────────────┬───────────────────┘
                                   │
                                   ▼
    ┌──────────────────────────────────────────────────────────────┐
    │      TOOL EXECUTION                                          │
    │  B  wrong tool runs                                          │
    │  C  executor rejects payload                                 │
    └──────────────────────────────┬───────────────────────────────┘
                                   │
                                   ▼
    ┌──────────────────────────────────────────────────────────────┐
    │  F  VISIBILITY FILTERS                                       │
    │      backend skip · persist skip · frontend hide lists       │
    │      ✗  tool ran in logs but card/DB silent in UI            │
    └──────────────────────────────┬───────────────────────────────┘
                                   │
                                   ▼
                         USER-VISIBLE UI
```

### Quick classifier

```
                    ┌─ tool_calls=0, end_turn ──────────────► A
                    │
  Check round log ──┼─ tool_calls=1, wrong name ──────────► B
                    │
                    ├─ tool_calls=1, right name, empty result ► C
                    │
                    └─ tool_calls=1, correct name ──────────┐
                                                            │
              ┌─ no early stream tool event before complete ► D
              │
              ├─ turn 2+ breaks; API history flat/missing ───────► H
              │
              ├─ turn 2+ breaks; force tool_choice "fixes" ────────► E (or masked H)
              │
              ├─ server log exec; UI/DB silent ───────► F
              │
              └─ guard/reminder path never reached ───────► G
```

| Mode | Breaks at | Key signal |
|------|-----------|------------|
| **A** | API — no tool block on wire | Wire trace: no tool block; `stop_reason=end_turn` |
| **B** | API — wrong tool name | `tool_calls=1 ['other_tool']`, prose wanted different tool |
| **C** | Executor — bad payload | Tool in round log; executor rejection / empty result |
| **D** | Stream bridge — late UI | Parser has tool block; no early UI tool event |
| **E** | In-turn thinking round-trip | Second round in *same turn* breaks; thinking missing from working messages |
| **F** | UI/persist — hidden channel | Server executed; no tool row/card in UI |
| **G** | Guard — dead branch | Prose promises tool; no nudge/reminder log |
| **H** | Cross-turn transcript loss | Turn 1 tools OK; turn 2+ prose-only; API history text-only |

---

## Failure mode catalog

### A — Prose-only turn (no tool block on wire)

**Symptoms:** Assistant says it will call a tool; UI shows text only; long "thinking" pause then nothing.

**Evidence:**
```
[sse_trace] start index=0 block_type='text'
[sse_trace] message_delta stop_reason='end_turn'
Round 0 complete: tool_calls=0 [], stop_reason=end_turn
```
No parser mismatch (provider and parser agree: no tool block).

**Root cause:** Provider stream ended without a `tool_use` / `function_call` block. Parser is not involved.

**Not a fix:** Parser patches, payload coercion, SSE wiring (no block to wire), forced `tool_choice` (may mask H).

**Distinguish from H:** Mode A is typical on **turn 1 / first attempt**. If tools worked on turn 1 and fail on turn 2+, check Mode H first.

**Fix direction:** Same-turn recovery nudge, prompt tuning, optional `tool_choice: required` for that specific product path only.

---

### B — Wrong tool executed

**Symptoms:** Prose promises tool A; UI shows tool B's surface or side effect.

**Evidence:**
```
Round 0 complete: tool_calls=1 ['other_tool'], stop_reason=tool_use
```
No trace of intended tool in executor or UI state.

**Root cause:** Parser and executor worked; model chose wrong tool. Recovery that only fires when `tool_calls=0` will **not** run.

**Not a fix:** Payload coercion, stream parser patches.

**Fix direction:** Prompt, tool descriptions, routing graph, or post-hoc validation of tool choice vs user intent.

---

### C — Tool called, empty or malformed payload

**Symptoms:** Round log shows correct tool name; result empty or error card; business state unchanged.

**Evidence:**
```
assembled blocks=[('text', None), ('tool_use', 'my_tool')]
executor rejected: empty / invalid payload
truncated or invalid tool input JSON
tool_use has empty input and no input_json_delta
```

**Root cause:** Tool block arrived and parsed; executor rejected shape. **This is not Mode A.**

**Fix direction:** Schema validation, coercion layer, explicit executor errors (never silent `{}` success), surface errors in UI.

**Framework note:** LangChain `StructuredTool` / Pydantic helps at validation boundary — see [frameworks.md](frameworks.md).

---

### D — Streaming visibility gap

**Symptoms:** User sees prose + long pause; tool card appears only after stream ends; feels like "system silent while agent continues."

**Evidence:** Parser/assembled blocks include tool_use but no early `tool_activity` / `on_tool_start` event before round complete.

**Root cause:** Stream parser emits tool-start internally but agent loop / SSE handler does not forward it to UI.

**Fix direction:** Forward `tool_use_start` (or equivalent) as soon as `content_block_start` type=tool_use; dedupe at execution time.

---

### E — Thinking round-trip break (in-turn)

**Symptoms:** First tool round in a **single user message** works; second round in same turn prose-only or empty payloads.

**Root cause:** Extended thinking blocks and `signature` not preserved in `assistant` message appended to `working_messages` before next model call within the turn.

**Fix direction:** Preserve thinking blocks in assistant content; accumulate `signature_delta` in stream handler.

**Do not confuse with H:** E is within one turn's working message loop. H is across separate user messages / session reload.

**Framework note:** LangChain message converters may strip thinking — verify before relying on LC for multi-round tool loops with thinking enabled.

---

### F — Hidden tool channel

**Symptoms:** Server logs show tool execution; UI or reload shows nothing.

**Root cause:** Tool name appears in one or more skip/hide lists (agent loop, persistence, frontend).

**Fix direction:** Grep all layers; align allow/deny sets when adding tools.

```bash
rg "HIDDEN_TOOL|SKIP|skip.*tool|filter.*tool|write_todos" --glob '*.{py,ts,tsx,js}'
```

---

### G — Dead guard / reminder branch

**Symptoms:** Assistant prose clearly promises a tool; no recovery nudge log; no pending reminder on next turn.

**Root cause:** Intent regex / classifier gap; or earlier branch always truthy so reminder else-path never runs.

**Fix direction:** Extend intent patterns with unit tests for promised-vs-completed phrasing; log when guard evaluates false.

---

### H — Cross-turn LLM transcript loss

**Symptoms:** Turn 1: tool fires, state updates. Turn 2+: agent prose-only or forgets tools; `thinking + auto` blamed incorrectly. **`tool_choice: required` appeared to fix it** (masks bug by disabling thinking and forcing tool).

**Evidence:**
```
# Turn 1 — OK
assembled blocks include tool_use, stop_reason=tool_use

# Turn 2 — degraded
API message store missing OR all content is plain strings
history rebuilt from UI chat text only
```

**Root cause:** UI chat store holds flattened assistant text. Rebuilding API input from it strips `thinking`, `tool_use`, and `tool_result`. The model cannot see prior tool calls.

**Fix direction:**
- Persist a **native API transcript** separately from UI chat rows.
- Prefer native transcript when building the next request; use UI store only for rendering.
- Never use UI-only history as the sole LLM input.

**Anti-pattern:** Enabling forced tool choice when turn 2+ fails — check Mode H first.

---

## Orchestration config (generic pattern)

Many agent stacks expose three toggles — keep them separate:

| Key / concept | When to enable |
|---------------|----------------|
| Force tool choice | **Rare.** API-level hard force. Often disables extended thinking. Only if auto + transcript fix still fail on turn 1 Mode A and product requires it. |
| Same-turn recovery nudges | Prose-only retry in same user turn. Default on for UX-heavy agents. |
| Next-turn pending reminders | Append hint on following user message. Default on. |

Default recommendation: **force off**, recovery on, verify multi-turn without force before shipping.

---

## Log patterns cheat sheet

| Pattern | Meaning |
|---------|---------|
| `Round N complete: tool_calls=0 [], stop_reason=end_turn` | Mode A — no tool block |
| `Round N complete: tool_calls=1 ['X'], stop_reason=tool_use` | Tool parsed; check if X matches intent |
| Early stream tool event before round complete | Mode D OK |
| `assembled blocks=[..., ('tool_use', 'name')]` | Parser saw blocks |
| Raw wire trace with no tool block events | Mode A — parser innocent |
| `MISMATCH stop_reason=tool_use but tool_calls=[]` | Provider signaled tool; parser lost block |
| `tool_use adopted from message_stop only` | Parser recovered block from final message payload |
| Forced tool_choice active | Masks H/E — disable to test auto |
| `request ... tool_choice=null` (auto) + thinking enabled | Confirms non-forced path |
| Executor rejected payload / truncated JSON | Mode C |
| Recovery: promised in prose but not called | Mode A guard fired |
| Turn 1 tool OK, turn 2 flat history | Mode H |

---

## Reusable lessons (checklist for any project)

1. **Prove layer with wire trace** — no tool events on wire → parser innocent (A). `MISMATCH` → parser bug.
2. **If executor logs rejection, tool WAS called** — fix payload (C), not API/model.
3. **If turn 1 OK, turn 2 bad** — inspect API message chain before touching thinking config (H).
4. **UI chat ≠ API input** — never debug tool history from rendered transcript alone.
5. **Forced tool choice is a mask, not a fix** — default off; enable deliberately.
6. **Recovery nudges ≠ force** — nudges let agent re-choose; they don't replace transcript fix.
7. **Generalize guard patterns** — one missed phrase silently disables recovery (G).
8. **Verify on multi-turn** — single-turn tests miss Mode H entirely.

---

## Provider notes (Anthropic-compatible streams)

Applies to Anthropic, many OpenAI-compatible proxies, MiniMax, etc.:

- Text deltas may include routing artifacts before tool blocks — sanitize if they corrupt UI.
- Tool input may arrive on `content_block_start.input` instead of `input_json_delta`.
- Some providers attach final tool block on `content_block_stop` or only on `message_stop.message.content` — parser must merge from all sources.
- Extended thinking requires `signature` round-trip in **in-turn** tool loops (E) and **cross-turn** transcript (H).
- `tool_choice: auto` with thinking is valid — if tools degrade, suspect transcript reconstruction before blaming the provider.

---

## Adapting this skill to your project

1. Copy `.cursor/skills/tool-call-fix/` to your repo or `~/.cursor/skills/tool-call-fix/`.
2. Add `examples/your-project.md` mapping pipeline layers to your files, log patterns, and test commands.
3. Keep `SKILL.md`, `reference.md`, and `frameworks.md` generic — do not fork the A–H taxonomy per project.
4. Point agents at your example file from `SKILL.md` or your project's `AGENTS.md`.

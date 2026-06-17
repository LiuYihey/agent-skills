# Framework comparison — what LangChain (etc.) actually fixes

This page answers: **if I use LangChain / LangGraph instead of a custom loop, which failure modes (A–H) go away?**

Short answer: **some plumbing, not model behavior or UI ownership.** LangChain reduces boilerplate for tool binding and message types, but **Modes A, B, D, F, G, and often H/E still require deliberate design.** LangGraph is better than classic `AgentExecutor` for transcript integrity.

## Mode-by-mode matrix

| Mode | Problem | LangChain classic | LangGraph | Raw SDK |
|------|---------|-------------------|-----------|---------|
| **A** Prose-only | Model emits text, no tool block | ❌ Same model behavior. `AgentExecutor` may retry via `handle_parsing_errors` but does not create tool blocks. | ❌ Same. Optional human-in-loop / retry nodes help only if you build them. | ❌ Prompt + optional same-turn nudge. |
| **B** Wrong tool | Model calls tool X, meant Y | ❌ Tool routing is still model choice unless you add structured routing (e.g. supervisor pattern). | ⚠️ Router node can constrain choices — product design, not automatic. | ❌ Same. |
| **C** Bad payload | Valid block, executor rejects shape | ✅ **Helps.** `StructuredTool`, Pydantic args schema, `@tool` validation catch shape errors early. Still need explicit error → UI path. | ✅ Same tool schemas; state reducers can validate. | ⚠️ You implement schema + coercion yourself. |
| **D** Stream UI lag | Tool card only after stream ends | ⚠️ **Partial.** `astream_events` / callbacks expose tool-start events — *if* you subscribe and forward to your UI. Default UIs often still end-of-turn. | ⚠️ Stream mode + custom event handler — same work as raw SSE bridge. | ❌ You wire `tool_use_start` → UI yourself. |
| **E** In-turn thinking loss | Second round in same turn loses thinking/signature | ❌ **Often worse.** LC `AIMessage` / conversion layers may drop provider-specific `thinking` blocks when rebuilding messages for the next model call. | ⚠️ Explicit state fields for thinking — you must model them in graph state, not assume LC preserves them. | ⚠️ You preserve blocks in `working_messages` (full control). |
| **F** Hidden tool channel | Tool ran; UI/DB filtered out | ❌ Application concern. LC does not know your `HIDDEN_TOOL_NAMES`. | ❌ Same. | ❌ Same — align skip lists. |
| **G** Dead guard | Intent regex never matches | ❌ Optional; LC has no standard "prose promised tool" guard. You still write `action_guard`-style logic or graph branches. | ⚠️ Conditional edges on parsed intent — still your patterns. | ❌ Custom guards. |
| **H** Cross-turn transcript loss | Turn 2+ API sees flat text history | ⚠️ **Mixed.** `ChatMessageHistory` + proper `tool_calls` / `ToolMessage` pairing *can* preserve chain — **if** you never round-trip through UI-only storage and use one canonical history. Many apps still flatten for chat UI and re-hit H. | ✅ **Best LC option.** Checkpointed graph state with explicit `messages` reducer (append-only, typed blocks) avoids ad-hoc rebuild. Still verify provider-specific blocks (thinking) are in state. | ⚠️ Full control — you persist native chain (e.g. `metadata.llm_messages`) or lose. |

**Legend:** ✅ helps if used correctly · ⚠️ possible but not automatic · ❌ does not fix by default

## What LangChain genuinely reduces

1. **Tool schema binding** — OpenAI function format / Anthropic tool format conversion, arg JSON parsing (helps **C** at the boundary).
2. **ToolMessage pairing** — Standard pattern for `assistant.tool_calls` + `tool` role messages (helps **H** *when this is the only history source*).
3. **Streaming callbacks** — Hooks for `on_tool_start` (foundation for **D** if you forward events).
4. **Retries / fallbacks** — Configurable retry on parser errors (adjacent to **A**, not a substitute for wire proof).

## What LangChain does *not* save you from

1. **Prose vs structured tool blocks (A)** — Fundamental LLM behavior; no framework eliminates it.
2. **UI transcript vs API transcript (H)** — If you store pretty chat strings and rebuild API input from them, LangChain will faithfully send broken history.
3. **Extended thinking round-trip (E, H with thinking)** — Anthropic `thinking` + `signature` are not first-class in all LC message converters; easy to strip silently.
4. **Multi-layer visibility filters (F)** — Backend skip lists + frontend hide sets remain your bug class.
5. **Provider stream quirks** — MiniMax-style `input` on `content_block_start`, blocks only on `message_stop`, etc. LC adapters abstract some of this but **adapter bugs become your bugs** — still need wire-level trace.

## Practical recommendations

### If you use LangChain `AgentExecutor` + `ConversationBufferMemory`

- High risk of **H**: memory often stores string content, not structured tool blocks.
- Mitigation: use `ChatMessageHistory` with full `AIMessage.tool_calls` + `ToolMessage`; never derive API input from a separate UI-only array.
- Test **turn 2+** after session reload.

### If you use LangGraph

- Treat `messages` in graph state as the **canonical API transcript** (append reducer).
- Persist checkpoints — UI can be a projection, not the source of truth.
- For extended thinking: store thinking blocks in state explicitly; do not rely on generic `messages` serialization.
- Stream custom events to your frontend for **D**.

### If you stay on raw SDK (Anthropic, OpenAI, compatible providers)

- Implement the minimum observability from [reference.md](reference.md): wire trace, assembled blocks, separate `llm_messages` store.
- This skill's A–H taxonomy maps 1:1 — no framework magic to debug through.

## "Should we migrate to LangChain to fix tool calls?"

| Situation | Recommendation |
|-----------|----------------|
| Mostly **C** (payload validation) | LC `StructuredTool` / Pydantic — reasonable win. |
| Mostly **H** (turn 2 amnesia) | Fix transcript architecture first; LangGraph helps **if** you commit to checkpointed state, not `AgentExecutor` + UI strings. |
| Mostly **A** (prose-only) | Framework won't help; prompts, guards, same-turn nudge. |
| Mostly **D/F** (streaming UI, hidden tools) | Custom SSE/UI work either way. |
| **E** + thinking + multi-round in one turn | Raw SDK or explicit graph state; verify LC converter preserves thinking before adopting. |

## Debugging LangChain-specific smells

```text
# Transcript / H
"messages" content is all str, no tool_calls on AIMessage
ToolMessage missing or orphaned from prior turn
convert_messages() dropped tool_calls

# Parser / C
Invalid tool args JSON → LangChain retried 3 times (symptom, not root fix)
StructuredTool validation error (good — visible C)

# Stream / D
astream_events v2: on_chat_model_stream but no on_tool_start forwarded to client
```

When investigating an LC app, still apply the skill checklist: **wire → parser → executor → persist → UI**, and map LC components to those layers instead of assuming the framework handled it.

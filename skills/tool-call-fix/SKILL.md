---
name: tool-call-fix
description: >-
  Diagnose and fix agent tool-call failures in any LLM agent stack: prose promises
  a tool but nothing runs, tool_calls=0 in logs, wrong tool executes, empty tool
  payload, streaming UI lag, thinking round-trip loss, or cross-turn transcript
  amnesia. Use when debugging tool_use / function_call pipelines with streaming,
  multi-turn sessions, or Anthropic-compatible providers.
---

# Tool Call Fix

Systematic playbook for **"the agent said it would call a tool, but nothing happened"** and related failures.

Works on **any stack** (raw SDK, LangChain, LangGraph, Vercel AI SDK, custom agent loops). Project-specific notes live in [examples/](examples/) — add your own file there when adapting.

## Core mental model

Prose and tool execution are **two weakly coupled channels**:

```
LLM stream
  ├─ text_delta        → UI text (usually immediate)
  ├─ thinking_delta    → "Thought for Ns" / hidden reasoning (immediate)
  └─ tool_use block    → parser → executor → tool card (often delayed)
```

**Never infer tool execution from assistant prose.** Only structured `tool_use` / `function_call` blocks count.

**UI transcript ≠ LLM transcript.** Chat history shown to users is often flattened text. The API needs structured blocks (`thinking`, `tool_use`, `tool_result`). Rebuilding API input from UI text alone causes **Mode H** — tools look "broken" on turn 2+ even when turn 1 worked.

## Before you patch: classify the layer

Same user symptom can break at **eight layers (A–H)**. Classify with logs before adding nudges, `tool_choice: required`, or prompt changes.

| Log / signal | Mode | Fix layer |
|--------------|------|-----------|
| Raw stream has no tool block; `stop_reason=end_turn` | **A** Wire prose-only | Model/guard; not parser |
| Turn 1 OK, turn 2+ degrades; flat API history | **H** Transcript loss | Persist native message chain |
| `tool_calls=1` but wrong name vs intent | **B** Wrong tool | Routing/prompt; recovery won't fire |
| Right tool, executor rejects input | **C** Bad payload | Schema validation / coercion |
| Parser has tool block; UI late or silent mid-stream | **D** Stream bridge | Forward early tool events to UI |
| Second round *in same turn* breaks | **E** In-turn thinking loss | Preserve thinking + signature in working messages |
| Server logs exec; UI/DB silent | **F** Visibility filter | Align skip/hide lists across layers |
| Guard should fire; didn't | **G** Dead guard branch | Fix intent detection patterns |

Full catalog + decision tree: [reference.md](reference.md)  
Framework comparison (LangChain, LangGraph, etc.): [frameworks.md](frameworks.md)

## Investigation checklist

Copy and adapt paths to your project:

```
Tool-call investigation:
- [ ] 1. Reproduce + capture session / trace id
- [ ] 2. Raw provider stream (before your parser) — prove tool block on wire or not
- [ ] 3. Parsed round summary — tool_calls, stop_reason, assembled blocks
- [ ] 4. API input for next turn — structured blocks, not UI text alone
- [ ] 5. Classify failure mode (A–H)
- [ ] 6. Fix the correct layer — not a force/nudge on the wrong layer
- [ ] 7. Restart services; rebuild frontend if SSE/UI changed
- [ ] 8. Verify on turn 2+ of the same session (catches Mode H)
```

### What to log (minimum viable observability)

| Layer | What to capture |
|-------|-----------------|
| **Wire** | Raw SSE events *before* assembly (`content_block_start`, `input_json_delta`, `message_stop`) |
| **Parser** | `assembled blocks`, `MISMATCH` when `stop_reason=tool_use` but `tool_calls=[]` |
| **Round** | `tool_calls`, `stop_reason`, tool names per round |
| **Executor** | Rejection reason for bad payloads (never silent `{}` success) |
| **Transcript** | Separate store for API-native messages vs UI chat rows |
| **UI** | Early `tool_activity` / `tool_call` events vs end-of-turn only |

Generic grep starting points (replace tool names):

```bash
rg "tool_use|tool_calls|stop_reason|assembled blocks|tool_choice" --glob '*.{py,ts,tsx,js,go,rs}'
rg "HIDDEN_TOOL|SKIP|skip.*tool|filter.*tool" --glob '*.{py,ts,tsx,js}'
rg "llm_messages|chat_history|working_messages|message_history" --glob '*.{py,ts,js}'
```

## End-to-end pipeline (where bugs hide)

```
User message
  → prompt / guard (G)
  → transcript builder (H)          ← most common "turn 2 broke" bug
  → LLM API stream
       ├─ text only (A)
       ├─ wrong tool (B)
       └─ tool + bad input (C)
  → stream parser (E)
  → SSE / UI bridge (D)
  → tool executor (C)
  → visibility filters (F)
  → persist UI + API transcripts (H)
  → user-visible UI
```

## Orchestration tiers (do not conflate)

Three separate mechanisms — mixing them up causes false fixes:

| Mechanism | What it does | When |
|-----------|--------------|------|
| **`tool_choice: required` / force** | API hard-forces a tool; often disables extended thinking | Last resort; masks transcript bugs |
| **Same-turn recovery nudge** | Prose-only → inject system hint → retry; agent still chooses | Mode A on turn 1 |
| **Next-turn pending reminder** | Append hint to next user message | Mode A / G across turns |

**Rule:** If turn 1 tools work and turn 2+ fail with `tool_choice: auto`, suspect **Mode H** before enabling force.

## Fix principles

1. **Wire trace before theory** — prove whether a tool block existed on the provider stream (Mode A vs parser drop).
2. **Classify A vs C vs H first** — "called tool but plan empty" can mean never called, bad payload, or history lost.
3. **API input ≠ UI chat** — inspect the message list actually sent to the model.
4. **Do not use forced tool choice as the permanent fix** — it hides transcript and thinking round-trip bugs.
5. **Fix source + consumers together** — parser, transcript, executor, SSE, frontend filters, persistence.
6. **Generalize once** — if a tool is hidden in one layer, grep all skip lists and mirrors.
7. **Test multi-turn** — single-turn tests miss Mode H entirely.

## Stack-specific guidance

| Stack | Start here |
|-------|------------|
| Raw Anthropic / OpenAI SDK | [reference.md](reference.md) — parser + transcript sections |
| LangChain / LangGraph | [frameworks.md](frameworks.md) — what LC helps vs what you still own |
| Custom agent loop + SSE UI | Map your files to the pipeline diagram above; add `examples/your-project.md` |

## Anti-patterns

| Do not | Why |
|--------|-----|
| Add guard nudge for every missed tool | Misses wrong-tool (B), payload (C), transcript (H) |
| Default `tool_choice: required` | Masks H/E; disables thinking; reduces agent autonomy |
| Blame "model didn't call" without wire trace | Turn 2+ failures are often Mode H, not Mode A |
| Blame parser when wire trace shows no tool block | Parser never had a block (Mode A) |
| Rebuild LLM input from UI messages only | Strips thinking/tool_use/tool_result (Mode H) |
| Patch one visibility skip list | Same tool name likely mirrored in 3+ places |

## When to stop investigating pipeline

- **Turn 1, Mode A:** Wire shows text only + `end_turn` — pipeline worked; no block to wire. Prompt/guard tuning, not parser patches.
- **Turn 2+, tools worked before:** suspect **Mode H** first — inspect API message chain before model config.
- **"Force fixed it":** usually masked transcript or thinking round-trip bug — do not ship force as the solution.

## Verification (generic)

After fixes, on **turn 2+** of the same session:

- API message store contains list-typed assistant/user content with `tool_use` / `tool_result` / `thinking` (if used)
- Wire trace shows expected `tool_choice` when force is off
- UI receives tool activity before round completes when streaming
- Executor errors surface in UI, not silent empty success

Run your project's unit tests for parser, transcript persistence, executor validation, and guard patterns.

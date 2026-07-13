---
name: agent-engineering
description: >-
  Production-hardened practices for building, debugging, and optimizing LLM
  agents with tools, streaming UI, and multi-turn sessions. Covers dual-store
  transcripts, artifact ownership, phase gating, orchestration tiers, failure
  classification, and root-cause fixes over patches. Use when designing agent
  loops, fixing "agent got dumber" regressions, streaming/persistence bugs, or
  prompt/orchestration behavior.
---

# Agent Engineering

Principles distilled from building a multi-agent product with streaming chat,
tool executors, and long-running sessions. Stack-agnostic - map layers to your
repo. For tool-call failure taxonomy see [tool-call-fix](../tool-call-fix/SKILL.md);
for prompts see [concise-agent-prompts](../concise-agent-prompts/SKILL.md).

## Core lesson

Most "agent got dumber" bugs are **infrastructure**, not model quality:

1. **Transcript loss** - UI-friendly flat text replaces API-native blocks
   (`thinking`, `tool_use`, `tool_result`) -> turn 2+ amnesia
2. **Symptom patches** - force tool, dedupe, keyword routing, prose replay
   hide (1) instead of fixing it

After fixing transcript round-trip, `tool_choice: auto` + extended thinking
often works without forcing tools.

---

## Universal pipeline

Every agent stack has the same weak points:

```
User input
  -> prompt / phase guard
  -> transcript builder          <- turn 2+ bugs often start here
  -> LLM API (stream)
       |- text_delta
       |- thinking_delta
       `- tool_use block
  -> stream parser
  -> tool executor + validation
  -> visibility / SSE bridge      <- "logs show exec, UI silent"
  -> dual persistence
       |- UI store (render)
       `- API store (model input)
  -> client UI
```

Before patching, identify **which layer** broke. Same symptom, eight different
fixes - see [tool-call-fix/reference.md](../tool-call-fix/reference.md) modes A-H.

---

## Non-negotiable principles

### 1. Prose != tool execution

Assistant text and structured tool blocks are **two weakly coupled channels**.
Never infer execution from prose. Only parsed `tool_use` / `function_call`
blocks count.

### 2. Dual-store transcript

Maintain two representations:

| Store | Purpose | Typical content |
|-------|---------|-----------------|
| **UI store** | Render chat | Flat text, custom message types, tool cards |
| **API store** | Next LLM call | Native blocks: thinking, tool_use, tool_result |

**Rules:**

- Rebuilding API input from UI text alone strips blocks -> Mode H
- After each turn, persist the **working message list** the provider expects
- UI store is never the sole source for model context

### 3. No patches - fix the source

| Symptom patch (avoid) | Root fix (prefer) |
|-----------------------|-------------------|
| Keyword/regex user-intent routing | Inject session facts into context |
| dedupe / strip / trim on model output | Fix prompt scope or persistence model |
| `tool_choice: required` as default | Fix transcript + thinking round-trip |
| Hard quotas in prompt ("max 3 calls") | Phase gating + role scope |
| Hard quotas in code (budget counters) | Structural tool filter; overlap guard only |
| `turn_complete` prose replay | Single artifact ownership |
| Backend re-parsing agent output | Agent supplies structured tool args; validate/coerce |
| Fixed follow-up copy / backend regex UI slots | Same segment rule in stream + persist; agent writes prose |
| `repositionAfterSameTurnToolcards` / `normalizeAssistantToolOrder` | Birth segments in API block order; update by `segment_id` only |
| Regex-stripping false agent claims from saved text | Session state prefix + prompt scope in ReAct loop |
| One-off examples in prompts | Encode principle in one sentence |

### 4. Generalize once

One bug often implies a **class**. When fixing:

- Grep sibling skip lists, guards, persistence paths, prompt copies
- Change producer + all consumers together
- Add multi-turn regression test (single-turn misses transcript bugs)

### 5. Structural constraints beat prompt pleading

Prefer disabling phases, tools, or code paths over "please don't repeat X" in
prompts. Inject **facts** (what was already done) instead of **rules**
(guess user intent from keywords).

### 6. Single artifact ownership

Each user-visible artifact has one owner:

| Pattern | Owner | Anti-pattern |
|---------|-------|--------------|
| Long-form deliverable (report, summary) | Dedicated message type / card | Same text in assistant bubble + card |
| Follow-up offer (export, confirm) | Separate bubble or CTA slot | Appended to deliverable bubble |
| Side-panel state (plan, todos) | Tool result -> dedicated UI channel | Hidden in generic tool list |
| Structured tool input | Agent via tool schema | Backend LLM re-extracts from free text |

Produce once in the right layer; align server persistence with client rendering.

---

## Investigation protocol

Copy before adding nudges, force tool, or dedupe logic:

```
1. Reproduce + capture session / trace id
2. Raw provider stream - tool block on wire or prose-only?
3. Parser round - tool_calls, stop_reason, assembled blocks
4. API input for next turn - structured blocks present?
5. Classify failure mode (A-H)
6. Fix the matching layer only
7. Restart services; invalidate prompt caches if applicable
8. Verify turn 2+ same session
```

**Red flags that suggest wrong-layer fix:**

- "Force tool fixed it" -> likely masked transcript loss
- "Only turn 1 broken" -> likely Mode A (model/guard), not H
- "Turn 2+ broken, turn 1 OK" -> suspect H before model config
- "Recovery nudge doesn't help" -> likely Mode B (wrong tool), not A

Minimum observability:

| Layer | Log |
|-------|-----|
| Wire | Raw SSE before assembly |
| Parser | `assembled blocks`, mismatch when `stop_reason=tool_use` but empty calls |
| Round | tool names, stop_reason per round |
| Executor | Rejection reason - never silent empty success |
| Transcript | API store vs UI store side by side |

Generic grep:

```bash
rg "tool_use|tool_calls|stop_reason|message_history|chat_history" --glob '*.{py,ts,tsx,js,go}'
rg "SKIP|HIDDEN|hide.*tool|filter.*tool" --glob '*.{py,ts,tsx,js}'
```

---

## Orchestration tiers (do not conflate)

Three separate mechanisms - mixing them causes false fixes:

| Mechanism | Effect | When |
|-----------|--------|------|
| **Force tool choice** | API mandates a tool; often disables thinking | Last resort; masks H/E |
| **Same-turn recovery** | Inject hint after prose-only; agent still chooses | Mode A, turn 1 |
| **Next-turn reminder** | Append hint to following user message | Mode A/G across turns |

Recovery that fires only when `tool_calls=0` **cannot fix wrong-tool (B)**.

Default posture: force off; recovery on; fix transcript first.

---

## Implementation norms

### Visibility alignment

Tool visibility is often duplicated:

- Agent loop (stream events)
- Persistence layer (what gets saved)
- Frontend (what gets rendered)

Adding/changing a tool -> grep and sync all mirrors. Critical user-facing tools
(e.g. plan/todo) must not appear in hide lists.

### Streaming UX

- Emit tool activity **on block start**, not after round completes
- Stream errors to client - never silent catch in SSE handlers
- Dedupe early tool announcements if parser fires twice

### Segment persistence (streaming UI)

Follow **LLM API block order** in the UI transcript. One ordered array; each
born segment keeps its index forever.

| Concept | Rule |
|---------|------|
| **Birth** | Assign stable `segment_id` + monotonic `seq` when a text block or toolcard first appears |
| **Streaming** | Only update `.content` on the same `segment_id` - never move rows |
| **Toolcards** | Another segment type; append at block-stream position, not "always physical end" detached from blocks |
| **Match key** | `segment_id` for upserts; `(seg_id, lane, turn)` is legacy fallback only |

**Structural constraints (not patches):**

- Integrated summary: buffer text until pending tools finish, then birth one segment
- `structure_preview`: defer persistence until the toolcard lands

**Anti-patterns (symptom patches - remove once source is fixed):**

- `repositionAfterSameTurnToolcards` - splices assistant bubbles after toolcards post-hoc
- `normalizeAssistantToolOrder` - replays the same reposition on hydrate/merge
- Dual semantics: assistant upsert by `(seg_id, lane, turn)` while toolcards only append

**WYSIWYG rule:** what the user sees during SSE streaming must match what
gets persisted. Apply the **same** birth + content-update rules in the client
`applyOp` path and the server `_apply_op` writer. **Do not** bulk-replace the
transcript from the server immediately after a stream completes. **Do** restore
from the server on cold page load when a `sessionId` is active.

Example: PDF invitation tail -> split at first `pdf` keyword (Latin letters
only - not 综述 in titles); sentence boundary before the PDF sentence; main
bubble + `_pdfTail` bubble in both UI and DB.

### Agent behavior vs backend text surgery

When the model falsely claims side effects (e.g. "PDF is already generating"):

| Avoid | Prefer |
|-------|--------|
| Regex-strip claims from persisted assistant text | Prompt scope + ReAct loop state injection so the agent knows job status |
| Post-hoc prose cleanup in finalize hooks | Facts in session prefix: report job pending/ready/failed |

The UI store should reflect what the agent **said**, not a sanitized rewrite.
Correct behavior at the source; persistence mirrors the stream.

### Phase-based multi-step agents

For research -> synthesize -> deliver workflows:

| Approach | Use |
|----------|-----|
| **Phase enum + tool filter** | Which tools are available now |
| **Session state prefix** | Facts: steps done, artifacts exist, job status |
| **Keyword intent routing** | Avoid - brittle across phrasing/locale |

- Agent plans once; state injection prevents re-planning loops
- No hard action budgets - use overlap/redundancy guards if needed
- Let agent decide retries; don't force retry on empty results

### Plan-first / prerequisite tools

- Hard gate (block until plan exists) -> brittle; removing gate drops behavior
- Prefer: strong role in system prompt + JIT reminder when prerequisite missing
- Watch for dead reminder branches (weak hint always truthy -> strong hint never runs)

### Payload validation

- Coerce common provider/schema variants at executor boundary
- Empty/invalid payload -> explicit error surfaced in UI
- Some providers deliver tool input on start block, not JSON deltas - parser must handle both

### Thinking round-trip

If using extended thinking:

- Preserve thinking + signature in working messages (in-turn and cross-turn)
- Stripping blocks breaks multi-round tool use within one turn (Mode E)

### Prompt scope

Follow [concise-agent-prompts](../concise-agent-prompts/SKILL.md):

- Litmus: "Does this line help the agent decide what to do?"
- Steer scope via role, not output quotas
- No inline worked examples that fossilize one scenario

---

## Post-fix checklist

```
- [ ] Turn 2+ same session works without force tool
- [ ] API store retains tool_use / tool_result / thinking blocks
- [ ] Tool UI appears before round completes
- [ ] Page refresh preserves all segments and special cards
- [ ] Streamed multi-bubble layout matches persisted transcript after turn end
- [ ] Post-tool / integrated prose stays below same-turn toolcards without reposition (new `segment_id` sessions)
- [ ] Visibility skip lists aligned across layers
- [ ] Services restarted after prompt/config changes
- [ ] Multi-turn tests added or updated
```

---

## Related skills

- Failure modes A-H + decision tree: [tool-call-fix](../tool-call-fix/SKILL.md)
- Prompt writing: [concise-agent-prompts](../concise-agent-prompts/SKILL.md)
- Symptom -> wrong fix -> right fix catalog: [reference.md](reference.md)

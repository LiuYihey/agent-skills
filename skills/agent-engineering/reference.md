# Agent Engineering - Pattern Catalog

Generic symptom catalog from production agent development. Map examples to your
stack. See [SKILL.md](SKILL.md) for norms.

---

## 1. Tool call pipeline

| Problem | Symptom | Root cause | Fix |
|---------|---------|------------|-----|
| Prose-only (A) | "I'll do X" but no tool block | `stop_reason=end_turn`, wire has text only | Prompt/guard; same-turn recovery - not parser |
| Wrong tool (B) | Intent X, tool Y executes | Routing or prompt; recovery skips when any tool fired | Fix role/routing; recovery won't help |
| Bad payload (C) | Executor rejects input | Schema variant, empty body, wrong nesting | Coerce at boundary; explicit error to UI |
| Stream delay (D) | Long pause before tool card | Start event not forwarded to client | Forward on block start |
| Thinking loss (E) | 2nd round same turn fails | thinking/signature dropped from working msgs | Preserve full content blocks |
| Hidden tool (F) | Server logs exec, UI empty | Tool in skip/hide list somewhere | Align all visibility mirrors |
| Transcript loss (H) | Turn 1 OK, turn 2+ degrades | UI flat text -> API input | Persist native API message chain |
| False fix | Force tool "fixes" it | Masks H; may disable thinking | Fix transcript; disable force |

---

## 2. Transcript, persistence, UI bubbles

| Problem | Symptom | Root cause | Fix |
|---------|---------|------------|-----|
| Refresh loss | Messages missing after reload | Single-row upsert overwrites segments | Segment-keyed persistence |
| Duplicate content | Same summary twice | End-of-turn replay + live stream | One owner per artifact |
| Client dedupe | Frontend strips "noise" | Hiding server/UI model mismatch | Fix persistence; remove client patch |
| Card not saved | Special UI card gone on reload | Extraction too strict or no persist hook | Dedicated persist event + metadata fallback |
| Session drift | Wrong content after navigation | Component state not reset | Reset on session id change |
| Stream surgery | Mid-stream text replace corrupts UI | Post-hoc prose editing | Remove; fix at source |

---

## 3. Multi-step / domain agents

Patterns observed in research, planning, and deliverable workflows:

| Problem | Symptom | Root cause | Fix |
|---------|---------|------------|-----|
| Re-planning loop | Repeats planning each turn | No session facts in context | State prefix with completed steps |
| Recap bloat | Deliverable repeats user question | Prompt asks for recap | Direct output; cut recap instruction |
| Hard budget | Stops at N actions | Counter in code/prompt | Remove budget; redundancy guard only |
| Double extraction | Backend re-parses agent text | Tool schema too loose | Agent supplies structured args |
| Mixed bubbles | Offer + deliverable in one message | Single assistant role for both | Extract to separate UI slot |
| Fixed copy | Follow-up feels non-native | Hard-coded template string | Agent writes; system extracts |
| Keyword routing | "try again" -> wrong mode | Regex on user text | Phase + tool filter |
| Early exit | Skips steps | Loose guard / handoff | Phase gating + session facts |
| Stuck async job | UI "generating" forever | Executor/UI/persist desync | Align job status events |
| Forced retry | Must retry empty results | Hard rule in guard | Let agent decide via state |

---

## 4. Prompt and orchestration

| Problem | Symptom | Root cause | Fix |
|---------|---------|------------|-----|
| Gate removal regression | Prerequisite tool never called | Hard gate was only enforcement | Strengthen role + fix reminder logic |
| Dead reminder | Strong hint never reaches model | Weak hint always truthy | Return empty when condition unmet |
| Instruction burial | Key rule ignored | Prompt too long | concise-agent-prompts: cut filler |
| Quota fossilization | Behavior breaks when task shifts | "Max N calls" in prompt | Scope via role + structural gates |
| Mechanism confusion | nudge vs force debate | Three tiers treated as one | See orchestration table in SKILL.md |

---

## 5. Observability

| Problem | Symptom | Root cause | Fix |
|---------|---------|------------|-----|
| Blind production | Can't see round summaries | Log level filters INFO | Round summary at visible level or trace flag |
| A vs parser ambiguity | Can't tell prose-only from drop | No raw stream capture | Wire trace + parser mismatch alert |
| Silent client failure | Stream stops, no error | Uncaught error in handler | Error event; safe finally |
| Thinking blamed | "Thinking breaks tools" | Usually E + H stacked | Fix transcript first |

---

## 6. Anti-pattern -> fix pattern

```
Symptom                          Wrong fix                    Right fix
-------------------------------------------------------------------------
Turn 2+ tools break              force tool choice            native API transcript store
Duplicate deliverable bubbles    client dedupe/strip          segment persistence + card type
User retry phrasing              keyword -> mode enum         phase + tool filter
Agent repeats planning           "don't repeat" in prompt     inject completed-step facts
Action budget hit                increment counter            structural gate; overlap guard
Offer mixed with summary         strip text from bubble       separate message type / slot
Empty tool payload               retry nudge only             coerce + explicit executor error
Late tool card                   wait for round end           forward start event immediately
Backend re-extracts agent text   add another LLM pass         agent structured tool args
```

---

## 7. Lessons that generalize

1. **Single-turn tests lie** - always verify turn 2+ in the same session.
2. **Force tool is a diagnostic**, not a shipping fix - if it "works", find what it masked.
3. **UI convenience != model input** - flattening for display must not become the memory format.
4. **Patches compound** - each dedupe/strip/regex adds debt; redesign when second patch appears.
5. **Agent should own structure** - push parsing into tool schemas; backend validates, not re-interprets.
6. **Artifacts need types** - treat summaries, offers, plans as first-class persisted entities.
7. **Prompt caches exist** - config/prompt changes may require process restart to take effect.
8. **Provider quirks are real** - tool input location, thinking signatures, sanitize routing artifacts in parser layer.

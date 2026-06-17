---
name: concise-agent-prompts
description: >-
  Universal principles for writing agent prompt templates in multi-agent
  pipelines. Covers role separation, scope steering, anti-patterns, and
  iterative review flows. Use when designing or refining any LLM agent prompts.
---

# Concise Agent Prompt Craft

General-purpose guidelines for writing prompt templates used by LLM agents in
multi-step pipelines.

## 1 — The Litmus Test

Every line you write that enters an LLM context must pass one question:
"Does the agent need this to decide what to do?"
If no, cut it.

Only write what the agent needs to **DO** or **KNOW** to act correctly. Never
include:
- **Delivery meta-commentary** — framing, transitions, or narration about the
  prompt itself (e.g. "You will now analyse…", "The following section covers…").
- **Prompt-writer notes** — rationale, caveats, or asides intended for a human
  reader rather than the executing agent.

## 2 — Defaults & Hygiene

- End every system prompt with a concise directive, e.g.:
  `Be concise. No preamble, no summary, no template sections.`
  (unless structured output is required downstream).
- Strip all filler: greetings, hedging phrases, meta-commentary about the task.
- If the prompt can be understood without a sentence, delete that sentence.

## 3 — Steer Scope, Not Quotas

- **Never** hard-code bullet counts, word limits, or truncation rules on model
  output. These are symptoms of an unclear scope.
- If output is too verbose → shorten and sharpen the **prompt**, not the output.
- If output misses a dimension → add it to the **role description**, not a checklist
  appended to the instruction.
- Regex-stripping or post-hoc trimming of model output is a code smell — fix the
  prompt first.

## 4 — Prefer Flowing Prose Over Structured Scaffolding

- Default output format for reasoning / analysis agents: short connected prose,
  not bullet inventories or category-labeled lists.
- Reserve numbered lists and structured schemas **only** for agents whose output
  is consumed by a parser or another agent.
- When you do need structure, specify the **minimum** structure required — every
  extra heading or field is a chance for the model to hallucinate filler.

## 5 — No Hard-Coded Limits or Examples in Prompts

- **Never** embed concrete length caps (e.g. "≤ 5 bullets", "max 200 words")
  directly in a prompt. These couple the prompt to a specific model's verbosity
  and silently break when the model or task shifts.
- **Never** hard-code specific cases, sample inputs, or worked examples as
  inline rules. Such cases fossilise one scenario and mislead the model on everything else.
- When you feel the urge to add a cap → the real problem is vague scope;
  go back and sharpen the **role** and **output contract** instead.
- When you feel the urge to add an example → encode the **principle** behind
  the example in one sentence; let the model generalise.
# Agent Engineering

Installable agent skills plus reusable Cursor rules for building and debugging
LLM agents with tools, streaming UI, and multi-turn sessions.

This repo was previously published as `LiuYihey/agent-skills`. The current repo
slug is `LiuYihey/Agent-Engineering`.

## Install Skills

The installable skills live under top-level `skills/` so they work with
[`npx skills add`](https://skills.sh).

Install all published skills:

```bash
npx skills add LiuYihey/Agent-Engineering
```

Install a specific skill:

```bash
npx skills add LiuYihey/Agent-Engineering --skill agent-engineering
npx skills add LiuYihey/Agent-Engineering --skill concise-agent-prompts
npx skills add LiuYihey/Agent-Engineering --skill hero-motion-sync-ux
npx skills add LiuYihey/Agent-Engineering --skill tool-call-fix
```

Works with Cursor, Claude Code, Codex, Gemini CLI, GitHub Copilot, and 35+
other agents.

## Published Skills

| Skill | Description |
|-------|-------------|
| [agent-engineering](https://skills.sh/LiuYihey/Agent-Engineering/agent-engineering) | Production-hardened practices for building, debugging, and optimizing LLM agents with tools, streaming UI, and multi-turn sessions. |
| [concise-agent-prompts](https://skills.sh/LiuYihey/Agent-Engineering/concise-agent-prompts) | Universal principles for writing agent prompt templates in multi-agent pipelines: role separation, scope steering, anti-patterns, and iterative review flows. |
| [hero-motion-sync-ux](https://skills.sh/LiuYihey/Agent-Engineering/hero-motion-sync-ux) | Sync hero text and molecule motion using one cadence, one timeline, and no flicker/blank-frame artifacts. |
| [tool-call-fix](https://skills.sh/LiuYihey/Agent-Engineering/tool-call-fix) | Diagnose and fix agent tool-call failures: prose promises a tool but nothing runs, `tool_calls=0`, wrong tool, empty payload, streaming UI lag, thinking round-trip loss, or cross-turn transcript amnesia. |

## Use Rules Manually

Rules are published separately under top-level `rules/`. They are not installed
by `npx skills add`; copy the ones you want into your repo's `.cursor/rules/`
directory.

Example:

```bash
mkdir -p .cursor/rules
cp rules/generalize-once.mdc .cursor/rules/
cp rules/no-patches.mdc .cursor/rules/
cp rules/request-decision-flow.mdc .cursor/rules/
```

You can also copy a single rule and rename it if your project uses a different
convention, as long as the file contents remain valid Cursor rule frontmatter +
Markdown.

## Usage

After installing, reference a skill in your agent session (for example
`@concise-agent-prompts` in Cursor) or describe a task that matches the skill
description and let the agent load it when appropriate.

## Repo Layout

```text
skills/   # installable skill packages for npx skills add
rules/    # reusable Cursor rule files to copy manually
```

## License

[MIT](LICENSE)

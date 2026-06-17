# Agent Skills

Open agent skills for AI coding assistants, installable via [skills.sh](https://skills.sh) and the `npx skills` CLI.

## Install

Install all skills:

```bash
npx skills add LiuYihey/agent-skills
```

Install a specific skill:

```bash
npx skills add LiuYihey/agent-skills --skill concise-agent-prompts
npx skills add LiuYihey/agent-skills --skill tool-call-fix
```

Works with Cursor, Claude Code, Codex, Gemini CLI, GitHub Copilot, and 35+ other agents.

## Skills

| Skill | Description |
|-------|-------------|
| [concise-agent-prompts](https://skills.sh/LiuYihey/agent-skills/concise-agent-prompts) | Universal principles for writing agent prompt templates in multi-agent pipelines — role separation, scope steering, anti-patterns, and iterative review flows. |
| [tool-call-fix](https://skills.sh/LiuYihey/agent-skills/tool-call-fix) | Diagnose and fix agent tool-call failures: prose promises a tool but nothing runs, `tool_calls=0`, wrong tool, empty payload, streaming UI lag, thinking round-trip loss, or cross-turn transcript amnesia. |

## Usage

After installing, reference a skill in your agent session (e.g. `@concise-agent-prompts` in Cursor) or describe a task that matches the skill description — the agent will load it automatically when appropriate.

## License

[MIT](LICENSE)

# client-skills

Agent-ready skills for integrating Helvetii AI products into your codebase.

Point your coding agent (Claude Code, Cursor, etc.) at a skill and describe your use case. The skill handles authentication, request shaping, and typed responses, in the context of your own app.

## Skills

- **`vindonissa/`** — Document intelligence: classify documents and extract structured JSON from PDFs.

## Usage

Clone the repo (or add it as a submodule) and reference the relevant skill directory from your agent configuration:

```bash
git clone https://github.com/helvetii-ai/client-skills.git
```

Each skill has its own `SKILL.md` describing inputs, outputs, and examples. Your agent reads it and generates the integration code.

## Requirements

- A Helvetii AI API key (`vnd_xxxxx`) — [docs.helvetii.ai](https://docs.helvetii.ai/docs/getting-started/)
- An agent that supports skills (Claude Code, Claude API with `computer-use` / `skills` tool, or compatible)

## License

MIT

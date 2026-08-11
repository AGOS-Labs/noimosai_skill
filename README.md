# NoimosAI Skills

Agent skills for [NoimosAI](https://noimosai.com) - AI-powered social media marketing platform.

## Available Skills

| Skill                                | Description                                                                     |
| ------------------------------------ | ------------------------------------------------------------------------------- |
| [noimosai-cli](skills/noimosai-cli/) | NoimosAI CLI reference for social media content generation and publishing       |
| [noimosai-mcp](skills/noimosai-mcp/) | NoimosAI MCP toolkit playbooks: personalized posts, analytics, research, website |

## Installation

### As an Agent Plugin (ChatGPT, Codex, Cursor, GitHub Copilot, Kiro, VS Code)

This repository is an [Agent Plugins 1.0.0](https://agent-plugins.org/) package: `plugin.json` + `skills/` + `mcp.json`. Installing it in a compatible client delivers both the skills and the NoimosAI remote MCP server (`https://mcp.noimosai.com/mcp`, OAuth-authorized) in one step — follow your client's plugin install flow and point it at this repository.

### As a Claude Code plugin

The same repository carries a Claude Code manifest (`.claude-plugin/` + `.mcp.json`):

```bash
/plugin install noimosai@<marketplace>   # or add this repo as a marketplace first
```

### Skills only

```bash
pnpm dlx skills add AGOS-Labs/noimosai_skill
```

To install a specific skill:

```bash
pnpm dlx skills add AGOS-Labs/noimosai_skill -s noimosai-cli
```

## License

[Apache-2.0](LICENSE)

---
name: noimosai-cli
description: "NoimosAI CLI reference for social media marketing. Use when working with noimosai commands, publishing social media posts, or automating content workflows. Triggers on: 'noimosai', 'NoimosAI', 'noimos', 'social media post', 'marketing', 'SNS', 'content creation', 'schedule post'."
---

# NoimosAI CLI Reference

CLI tool for NoimosAI - generate and publish social media content via AI agents.

**Supported platforms:** X (Twitter), Threads, Facebook, Instagram, YouTube, LinkedIn, TikTok, Bluesky, Pinterest, Note, WordPress

## Installation

```bash
npm install -g @agos-labs/noimosai-cli
```

## Quick Start

```bash
# 1. Authenticate
noimosai login

# 2. Start interactive chat
noimosai

# 3. Or one-shot: generate posts as JSON
noimosai chat "Create X and Instagram posts about AI trends" --output json > posts.json

# 4. Publish
noimosai post posts.json --now
```

## Commands

### Authentication

| Command           | Description                               |
| ----------------- | ----------------------------------------- |
| `noimosai login`  | Authenticate via browser OAuth or API key |
| `noimosai logout` | Remove all stored credentials             |

### Chat

```bash
noimosai chat [prompt] [options]
```

Without `[prompt]`, starts interactive chat UI. With `[prompt]`, runs one-shot and exits.

| Option                  | Description                                                     |
| ----------------------- | --------------------------------------------------------------- |
| `-o, --output <format>` | `text` (default) or `json` - JSON extracts structured post data |
| `-w, --workspace <id>`  | Override workspace ID                                           |
| `-s, --session <id>`    | Continue an existing conversation                               |

### Post

```bash
noimosai post <file> [options]
```

Publish posts from a JSON file (NoimosPostJson format, as output by `chat --output json`).

| Option                  | Description                                               |
| ----------------------- | --------------------------------------------------------- |
| `--now`                 | Publish immediately (required if no `--schedule`)         |
| `--schedule <datetime>` | Schedule for later: `YYYY-MM-DD HH:MM` (30+ min from now) |
| `--dry-run`             | Preview what would be sent without publishing             |

### Configuration

| Command                             | Description                |
| ----------------------------------- | -------------------------- |
| `noimosai config show`              | Show current configuration |
| `noimosai config set <key> <value>` | Set a config value         |
| `noimosai config get <key>`         | Get a config value         |
| `noimosai config path`              | Show config file path      |

**Config keys:**

| Key           | Description            | Values                                             |
| ------------- | ---------------------- | -------------------------------------------------- |
| `workspaceId` | Active workspace ID    | workspace ID string                                |
| `appUrl`      | Application URL        | URL (default: `https://app.noimosai.com`)          |
| `agentRegion` | Agent execution region | `auto` (default), `us-central1`, `asia-northeast1` |

## Interactive Chat

When running `noimosai` without arguments, an interactive terminal UI launches.

**Slash commands:**

| Command             | Description                            |
| ------------------- | -------------------------------------- |
| `/agent`            | Select an AI agent (workflow)          |
| `/workspace`, `/ws` | Switch workspace (resets conversation) |
| `/clear`, `/reset`  | Clear conversation history             |
| `/exit`, `/quit`    | Exit the chat                          |

**Post actions:** When the agent generates social media posts, an action bar appears with options to **Edit**, **Post Now**, or **Schedule** each post.

## Non-Interactive Workflow

Generate, review, and publish in a pipeline:

```bash
# Generate posts as JSON
noimosai chat "Write 3 engaging tweets about sustainable tech" --output json > posts.json

# Review and edit the JSON file as needed

# Publish immediately
noimosai post posts.json --now

# Or schedule for later
noimosai post posts.json --schedule "2026-03-01 10:00"
```

To continue a conversation:

```bash
# First message
noimosai chat "Analyze our competitor's social strategy" --output json
# Note the sessionId from output metadata

# Follow-up in same session
noimosai chat "Now create posts based on that analysis" --session <sessionId> --output json
```

## Detailed References

| Reference                                                        | Contents                                                                  |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------- |
| [references/post-json-schema.md](references/post-json-schema.md) | NoimosPostJson format: schema, platform-dataKey mapping, validation rules |

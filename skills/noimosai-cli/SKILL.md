---
name: noimosai-cli
description: "Use this skill whenever the user mentions 'noimosai', 'NoimosAI', or 'noimos' in any context — including CLI usage, commands, options, login, config, posting, scheduling, workspace management, agent listing, session resumption, or troubleshooting. Also use when the user asks about publishing social media posts via a CLI tool, automating SNS content workflows, or generating and scheduling posts programmatically. This is the sole authoritative reference for the noimosai CLI tool. Do NOT trigger for general social media development (share buttons, webhooks, APIs) or unrelated CLI tools."
---

# NoimosAI CLI Reference

CLI tool for NoimosAI - generate and publish social media content via AI agents.

**Supported platforms:** X (Twitter), Threads, Facebook, Instagram, YouTube, LinkedIn, TikTok, Bluesky, Pinterest, Note, WordPress

## Commands

### Authentication

| Command           | Description                               |
| ----------------- | ----------------------------------------- |
| `noimosai login`  | Authenticate via browser OAuth or API key |
| `noimosai logout` | Remove all stored credentials             |

### Chat

`noimosai chat` without options starts interactive chat. With `-p`, runs one-shot and exits.

| Option                  | Description                                                     |
| ----------------------- | --------------------------------------------------------------- |
| `-p, --prompt <text>`   | One-shot message to the agent                                   |
| `-r, --resume <id>`     | Resume a previous session                                       |
| `-w, --workspace <id>`  | Override workspace ID                                           |
| `-o, --output <format>` | `text` (default) or `json` - JSON extracts structured post data |
| `--agent <id>`          | Specify an agent to chat with                                   |

### Agent

| Command               | Description                  |
| --------------------- | ---------------------------- |
| `noimosai agent list` | List agents in the workspace |

Options: `-w, --workspace <id>` to override default workspace ID.

### Workspace

| Command                   | Description              |
| ------------------------- | ------------------------ |
| `noimosai workspace list` | List available workspaces |

### Integration

| Command                       | Description                    |
| ----------------------------- | ------------------------------ |
| `noimosai integration list`   | List connected integrations    |

Options: `-w, --workspace <id>` to override default workspace ID.

### Post

```bash
noimosai post <file> [options]
```

Publish posts from a JSON file (NoimosPostJson format, as output by `noimosai chat -p "..." --output json`).

| Option                  | Description                                               |
| ----------------------- | --------------------------------------------------------- |
| `--now`                 | Publish immediately                                       |
| `--schedule <datetime>` | Schedule for later: `YYYY-MM-DD HH:MM` (30+ min from now) |
| `--draft`               | Save as an in-app draft the user approves in NoimosAI     |
| `--dry-run`             | Preview what would be sent without publishing             |

Exactly one of `--now` / `--schedule` / `--draft` is required. Use `--draft` when the user has not explicitly approved the exact post text.

### Upload media

```bash
noimosai upload-media <file> [--mime <type>] [-w <workspaceId>]
```

Uploads a local image/video/audio (max 32MB) to the workspace and prints the storage path a post's `media[].path` or `publish-article --header-image` accepts. Mime is inferred from the extension; pass `--mime` for unusual ones.

### Delete posts

```bash
noimosai delete-post <postId...> --yes
```

Deletes every platform copy of each post by its `postId` (from `noimosai post` output or `tools run workspace/fetch_my_posts`). Cancels drafts/scheduled; a post that already went out is also removed FROM the platform — irreversible, hence the required `--yes`. Max 25 per call.

### Publish an article

```bash
noimosai publish-article <file.md> --provider <WordPress|X|Substack> --account <providerAccountId> --yes
```

| Option | Description |
| ------ | ----------- |
| `--title <title>` | Article title (default: the file's first `# ` heading) |
| `--schedule <datetime>` | ISO datetime to schedule; omit to publish now |
| `--header-image <gcsPath>` | Storage path of an uploaded header image (not a URL) |
| `-w, --workspace <id>` | Override the configured workspace |

The file is the markdown body. Real public action (`--yes` required); asynchronous — returns an `idempotencyKey` once queued. Same title+body to the same account within 24h is rejected as a duplicate. X Articles need Premium+/Verified Organization.

### Tools

```
noimosai tools list [--server <group>]
noimosai tools run <server>/<name> --args '<json>' [-w <workspaceId>]
```

Typed NoimosAI tools: workspace reads (past posts, brand context), analytics (GSC, GA4, Semrush), social search, media generation, and more.

| Option | Description |
| ------ | ----------- |
| `--server <group>` | Filter `tools list` by group (e.g. `workspace`, `social`, `deep_analytics`) |
| `--args <json>` | Tool arguments as a JSON object |
| `-w, --workspace <id>` | Override the configured workspace |

Tools marked `[billed]` in the listing deduct NoimosAI team credits per call at actual external-API/AI cost; unmarked tools are free reads. Calls are idempotent per invocation — a network retry never double-charges.

Examples:

```
noimosai tools run workspace/fetch_my_posts --args '{"limit": 10}'
noimosai tools run deep_analytics/gsc_search_performance --args '{"days": 28}'
noimosai tools run social/search_x_posts --args '{"query": "AI agents"}'   # billed
```

### Configuration

| Command                             | Description                |
| ----------------------------------- | -------------------------- |
| `noimosai config show`              | Show current configuration |
| `noimosai config set <key> <value>` | Set a config value         |
| `noimosai config get <key>`         | Get a config value         |
| `noimosai config path`              | Show config file path      |

**Config keys:** `workspaceId`

## Workflow

Generate, review, and publish in a pipeline:

```bash
# Generate posts as JSON
noimosai chat -p "Write 3 engaging tweets about sustainable tech" --output json > posts.json

# Review and edit the JSON file as needed

# Publish immediately
noimosai post posts.json --now

# Or schedule for later
noimosai post posts.json --schedule "2026-03-01 10:00"
```

To continue a conversation:

```bash
# First message
noimosai chat -p "Analyze our competitor's social strategy" --output json
# Note the sessionId from output metadata

# Follow-up in same session
noimosai chat -p "Now create posts based on that analysis" --resume <sessionId> --output json
```

## Detailed References

| Reference                                                        | Contents                                                                  |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------- |
| [references/post-json-schema.md](references/post-json-schema.md) | NoimosPostJson format: schema, platform-dataKey mapping, validation rules |

---
name: noimosai-cli
description: 'This skill should be used whenever the user mentions "noimosai", "NoimosAI" or "noimos", or asks to publish or schedule social posts, run a marketing agent, or manage a brand guide, knowledge base or inbox from a CLI. Not for unrelated CLI tools.'
---

# NoimosAI CLI

`noimosai` drives NoimosAI — an autonomous marketing platform — from the terminal.
Every command takes flags (no interactive prompt is ever required), accepts a
global `-o json`, and is safe to retry.

This file covers WHICH command to run and in what order. For the full flag list of
any command, run `noimosai <command> --help` — that output is generated from the
CLI itself and cannot drift.

## Setup ladder

Run in this order; each step is a prerequisite for the next.

```bash
noimosai login --api-key "$NOIMOS_API_KEY"   # or --oauth (opens a browser)
noimosai workspace list                       # pick an existing workspace…
noimosai config set workspaceId <id>          # …and make it the default
```

With no workspace yet, create one instead of picking one:

```bash
noimosai init "Acme" --type company --website https://acme.com \
  --description "B2B analytics for logistics" --goals masterSearch,scaleSocial
```

`init` runs the full onboarding, including an AI research pass over the site —
several minutes. If it fails partway, the error names the workspace it created:
rerun with `--resume <that id>` and the same flags rather than starting over.
Goals are a comma-separated subset of `outsmartCompetitors`, `masterSearch`,
`maximizeSales`, `scaleSocial`, `buildWorkflows`, `growthRoadmap`,
`adsManagement`. `--website` is required for `--type company`.

Connecting social accounts needs a browser — do it in the NoimosAI app, then
confirm with `noimosai integration list`.

## Command inventory

Required arguments and flags are shown; optional ones are in `--help`.

| Command | What it does |
|---|---|
| `login [--api-key <key>\|--oauth]` | Authenticate. Bare `login` prompts; `-o json` requires one of the flags. |
| `logout` | Remove all stored credentials. |
| `init <name> --type <company\|personal> --description <text> --goals <list>` | Create a workspace and run onboarding. |
| `chat [-p <message>] [-r <sessionId>]` | Run the agent. Bare `chat` is interactive; `-p` is one-shot. |
| `post <file> (--now\|--schedule <datetime>\|--draft)` | Publish posts from a NoimosPostJson file. |
| `delete-post <postId...> --yes` | Delete every platform copy of each post. Max 25. |
| `publish-article <file.md> --provider <WordPress\|X\|Substack> --account <id> --yes` | Publish long-form. |
| `upload-media <file>` | Upload a local image/video/audio (≤32MB); prints the storage path. |
| `pinterest-boards <providerAccountId>` | List boards as `id<TAB>name (N pins, PRIVACY)`. |
| `tools list [--server <group>]` / `tools run <server>/<name> --args '<json>'` | Typed tools: workspace reads, analytics, social search, media generation. |
| `brand get` / `brand set [...]` | Read / patch the brand guide the agents are grounded on. |
| `kb list\|show <id>\|create <title>\|add <id>\|update <id>\|rm <id>` | Knowledge base the agents retrieve from. |
| `inbox dm <provider> <target> <text> -a <accountId> --yes` | Send a DM. X / Instagram / Facebook / TikTok. `--yes` is required; `--dry-run` previews. |
| `inbox reply <provider> <target> <text> -a <accountId> --yes` | Reply to a comment or mention. X / Instagram. `--yes` is required; `--dry-run` previews. |
| `integration list` | Connected accounts and their `providerAccountId`s. |
| `workspace list` | Available workspaces. |
| `config show\|set <key> <value>\|get <key>\|path` | CLI config. The only settable key is `workspaceId`. |

`-w, --workspace <id>` overrides the configured workspace on every
workspace-scoped command. `-o, --output <text\|json>` is global — it works on any
command, not just `chat`.

## Actions that cannot be undone

Approval is the only safety net on these; there is no staging state to fall back
on. Show the user the exact text and get an explicit go-ahead first.

- `inbox dm --yes` and `inbox reply --yes` — **send on invocation**. Both refuse
  to run without `--yes`; preview with `--dry-run` first.
- `post --now` — publishes immediately. `post --schedule` publishes later
  unattended.
- `publish-article --yes` — a real public action.
- `delete-post --yes` — a post that already went out is removed FROM the
  platform too.

`post --draft` and `post --dry-run` are always safe. Use `--draft` whenever the
user has not approved the exact post text: the post lands in NoimosAI as a draft
they approve in-app.

## Publishing a post

```bash
# 1. Generate. A full multi-step agent runs on the prompt, so brief it properly:
#    deliverable + count, platform/account, audience, brand & offer context,
#    voice, output language, hard constraints. A thin prompt gets a generic post.
noimosai chat -p "Create 3 Instagram feed posts (Japanese) for KISSA, a Tokyo
  specialty coffee shop, launching a summer cold brew (680 JPY, in-store only,
  from Aug 1). Audience: 25-35yo Tokyo office workers into single-origin.
  Casual warm tone, max 2 emoji, 2-3 hashtags, captions under 200 chars." \
  --output json > posts.json

# 2. Review and edit posts.json.

# 3. Ship.
noimosai post posts.json --draft                       # user approves in-app
noimosai post posts.json --schedule "2026-09-01 10:00" # 30+ min from now
noimosai post posts.json --now                         # approved content only
```

Exactly one of `--now` / `--schedule` / `--draft` is required.

Follow-ups reuse the session id from the first run's output metadata:

```bash
noimosai chat -p "Now shorten the second one" -r <sessionId> --output json
```

A paused turn (the agent asked a question and stopped) is answered the same way —
`-r <sessionId> -p "<answer>"` — not by starting a new session.

**Supported `post` platforms:** x, threads, facebook, instagram, youtube,
linkedin, tiktok, bluesky, pinterest, snapchat, mastodon, note. WordPress and
Substack are long-form: use `publish-article`, not `post`.

Per-platform payload rules (which platforms require media, what is derived vs.
what must be set) are in `references/post-json-schema.md`. Read it before hand-
writing a post file.

## Gotchas that cost a run

- **Pinterest needs a board id.** A pin cannot be created without one. Run
  `pinterest-boards <providerAccountId>` and put the id in the entry's
  `options.pinterest.boardId`.
- **Media is a storage path, not a URL.** `upload-media` prints the path that
  `media[].path` and `publish-article --header-image` accept.
- **`brand set --keywords` REPLACES the whole set** — pass the existing keywords
  too. Saving rebuilds the brand knowledge base and can take a minute.
- **`kb create` / `kb add` charge embedding credits**, and URL ingestion is slow.
- **`tools list` marks billed tools `[billed]`** — those deduct team credits at
  actual external-API/AI cost; unmarked ones are free reads. Calls are idempotent
  per invocation, so a network retry never double-charges, but each new call is a
  new charge.
- **Article duplicates are rejected**: the same title+body to the same account
  within 24h. That is the duplicate guard, not a transient failure. X Articles
  additionally need a Premium+/Verified Organization account.
- **`publish-article` is asynchronous** — it returns an `idempotencyKey` once
  queued, not a live URL.
- **X rejects a programmatic reply** unless that post's author @mentioned this
  account or quoted one of its posts. Report that refusal instead of retrying.
- **Ids come from a read, never from a URL.** DM conversation ids and comment ids
  come from the MCP read tools (`get_direct_messages`, `list_x_user_mentions`,
  `instagram_comments_list`); `providerAccountId`s come from
  `integration list`; `postId`s come from `post` output or
  `tools run workspace/fetch_my_posts`.

## Tool examples

```bash
noimosai tools run workspace/fetch_my_posts --args '{"limit": 10}'
noimosai tools run deep_analytics/gsc_search_performance --args '{"days": 28}'
noimosai tools run social/search_x_posts --args '{"query": "AI agents"}'   # billed
```

## Additional resources

- **`references/post-json-schema.md`** — NoimosPostJson format: schema,
  platform→dataKey mapping, per-platform requirements, validation rules.

---
name: noimosai-mcp
description: "Use this skill when the NoimosAI MCP server's tools are available and the user asks to create, personalize, schedule, or draft social media posts, analyze their accounts or website (GSC, GA4, SEO, social analytics), research trends/competitors, generate images or videos, or find leads — via NoimosAI. Covers tool selection, personalization workflows, billing awareness, and the draft/publish/schedule decision. Not for general social media API development."
---

# NoimosAI MCP Toolkit

NoimosAI exposes two layers of tools. Pick the right layer first:

| Layer | Tools | When |
|---|---|---|
| **Agent** | `chat` (+ `post` for its output) | Multi-step deliverables: full campaigns, article + image sets, deep reports. One call = a full autonomous agent run (minutes, credits). |
| **Direct tools** | everything else (`fetch_my_posts`, `gsc_*`, `search_*`, `generate_image`, …) | You orchestrate: read data, author content yourself, publish. Faster, cheaper, and you keep control of every step. |

Prefer direct tools when you (the host agent) can do the reasoning; use `chat` when the user wants NoimosAI's own agent to run the whole job.

## Billing — read before calling

- Tool descriptions carry `[Billed: …]` or `[Free: …]`. Free tools are workspace/database reads. Billed tools call paid external APIs or AI generation and deduct the team's NoimosAI credits at actual cost.
- Batch billed calls: one search with OR-joined keywords beats five narrow ones.
- Calls are idempotent per request — a network retry never double-charges, but each NEW call is a new charge.
- Insufficient credits returns an error asking the user to top up in the NoimosAI app; report it, don't retry.

## Playbook: personalized post (the core workflow)

1. `get_workspace_context` — brand, goals, output language, connected accounts (free).
2. `fetch_my_posts` — recent posts for the target account (free). Extract the account's real voice: tone, emoji usage, hashtag habits, typical length, hook style.
3. Author the post text YOURSELF in that voice, in the workspace's output language.
4. `post` with the right mode:
   - Always pass `mode` explicitly. `mode: "draft"` whenever the user has not explicitly approved the exact text; the post appears in NoimosAI as a draft the user approves in-app. Omitting it saves a draft, so a forgotten field never becomes a public post.
   - `mode: "publish"` — immediate. Only after the user approved the exact content.
   - `mode: "schedule"` + `scheduleAt` — approved content, later time.
   - `dryRun: true` first when unsure the payload is right.

Never publish or schedule content the user has not seen. Drafts are always safe.

### Per-platform requirements

`platform` decides the payload shape. Anything not listed is derived from the entry — set it in `options.<platform>` only to override.

| Platform | Media | Must set | Derived for you |
|---|---|---|---|
| x, threads, bluesky, mastodon | optional | — | one thread item per `textBlocks` entry; media rides the lead item |
| facebook, linkedin | optional | — | — |
| instagram | **required** | — | `igPostType`: REELS when the media is a video, else CAROUSEL |
| tiktok | **required** | — | `title` = caption's first line (≤90); `postType` from the media mime |
| youtube | **required — a video** | — | `title` = first line (≤100); `categoryId` 22; `privacyStatus` PUBLIC |
| pinterest | **required** | `options.pinterest.boardId` | `description` = your text |
| snapchat | **required** | — | `postType` SPOTLIGHT (dev environments only) |
| note | optional (cover = first media item with a public `url`; `path`-only uploads are NOT resolved for note) | — | `title` = first line; `isDraft` follows `mode` |

Pinterest: call `list_pinterest_boards` with the account's `providerAccountId` first and pass the chosen board's `id`. There is no default board — a pin cannot be created without one.

WordPress and Substack are NOT `post` targets — they are long-form providers. Use `publish_article` (below).

A missing requirement is rejected before anything is sent, with a message naming the field. Nothing in the batch publishes when one entry fails, so fix and re-send the whole call.

Attaching media: pass a workspace storage `path` as the post's `media[].path`. Three sources: `generate_image`/`generate_video` return it in their **Structured result** block (use the `path`, not the url); `upload_media` uploads a LOCAL file (your own image, an ffmpeg-edited video; max 32MB) and returns its path; a public `media[].url` also works — the server downloads it into the workspace.

Cancelling: `delete_posts` with the `postId`s (from the `post` tool's result or `fetch_my_posts` rows). All platform copies of each post are removed. A post that already went out is also removed FROM the platform — irreversible, so confirm before calling on published ones.

## Playbook: long-form article

1. Read the destination and voice: `get_workspace_context` + `fetch_my_articles` (free).
2. Write the article yourself — title + body.
3. Optional header image: `generate_image`, then pass its structured `path` as `headerImagePath`.
4. `publish_article` with the WordPress / X / Substack `providerAccountId`, plus `scheduleAt` to schedule instead of publishing now. It is a real public action — only after the user approved the exact text. X Articles need a Premium+/Verified Organization account. Publishing is asynchronous: you get an `idempotencyKey` once queued, not a live URL.

Resubmitting the same title+body to the same account within 24h is rejected — that is the duplicate guard, not a failure to retry around.

For a NoimosAI-generated article (research, SEO scoring, internal links) use `chat` instead and let its article agent write it.

## Playbook: analytics-grounded content

1. Read real numbers first: `gsc_search_performance` (queries/pages), `ga4_custom_report` / `ga4_analyze_pages` (traffic), `analyze_post_performance` (social), `semrush_*` (keywords/competitors — billed).
2. Cite only numbers the tools returned. Never estimate metrics.
3. Feed the findings into content: topics from rising queries, formats from top-performing past posts.

## Playbook: research & trends

- Own accounts: `fetch_my_posts`, provider analytics reads (free).
- External: `search_x_posts`, `search_tiktok_posts`, `search_youtube`, `search_reddit`, `google_trends_interest`, ad-library searches (all billed). Scope tightly; state the platform only if the user named one.

## Playbook: answering the inbox

1. `get_direct_messages` — read the threads (free). It carries both `providerAccountId` (ours) and the ids the writes need.
2. Draft the reply yourself, in the thread's language, grounded in what the person actually wrote.
3. Show the user the exact text and get their go-ahead.
4. `send_dm` (X / Instagram / Facebook / TikTok) or `reply_to_comment` (X / Instagram).

These two **send immediately** — unlike `stage_email_drafts` and `post`'s draft mode, there is no staging state to fall back on, so step 3 is the only approval that exists. Text only; media is an in-app action.

`target` is the conversation id — except Facebook, where Messenger addresses the recipient's PSID (the counterpart's `externalUserId`), and Instagram comment replies, which take the comment id. X rejects a programmatic reply unless that post's author @mentioned this account or quoted one of its posts; report that refusal rather than retrying.

## Playbook: lead generation

1. `apollo_search_organizations` / `apollo_search_people` — target companies/prospects (billed).
2. `find_work_email` — verified work email waterfall (billed); `verify_email` to double-check.
3. `stage_email_drafts` — stages outreach drafts for user review (free). Never send email without explicit approval; staging is the deliverable.

## Playbook: build & deploy a website

You build the site locally (Next.js or static — your own code, your own quality bar); NoimosAI hosts it.

1. `create_website` — new site record, returns `websiteId` (or `list_websites` to reuse one).
2. Build the site locally in a project directory.
3. `upload_website_source` — pass the project root's absolute path; it packs and pushes the SOURCE (`node_modules`/`.git`/`.next` excluded; the build runs server-side) and triggers a build. Re-upload to iterate — it replaces the previous source.
4. `get_website_build_status` — poll until `success` (or read `buildError` and fix).
5. `publish_website` — production hosting. ONLY after the user approved going live; `unpublish_website` reverses it.

## Playbook: media generation

`generate_image`, `edit_image`, `generate_video`, `generate_music`, `text_to_speech` (all billed — video is the most expensive). Confirm format/aspect/duration with the user before generating; regenerations cost the same as the first attempt.

## Playbook: the improvement loop (ship → measure → fix)

Marketing run from an agent should be a closed loop, not a fire-and-forget:

1. **Ship** — posts (`post`), articles (`publish_article`), site changes (`upload_website_source` → `publish_website`).
2. **Measure** — after enough time for data: `gsc_search_performance` (which queries/pages gained or lost), `ga4_analyze_pages` / `ga4_custom_report` (traffic, conversion paths), `analyze_post_performance` + `fetch_my_posts` (which posts worked and why).
3. **Diagnose** — compare against the goal, name the bottleneck: content (hooks, topics, timing), distribution (platform mix), or product (landing page copy, page speed, funnel friction).
4. **Fix at the right layer** — content problems: adjust the next round's topics/format from what the numbers say. Product problems: if you are running inside a coding agent with the user's app or site codebase available, fix the code itself — landing copy, meta/OG tags, structured data, page speed, signup friction — then redeploy and re-measure. Problems only NoimosAI's data can explain (brand positioning, keyword strategy): update `update_workspace_brand` / knowledge base so every later run inherits the fix.

Cite only numbers the tools returned; never estimate. One loop iteration per reporting period beats daily thrash — most channels need days for meaningful data.

## Ground rules

- `workspaceId` comes from config or `list_workspaces` — never invent one.
- Account-targeted calls need the `providerAccountId` from `get_workspace_context` / `list_integrations`.
- Write user-facing content in the workspace's output language (`aiResponseLanguage`) unless the user says otherwise.
- Report tool errors plainly with what you tried; don't silently substitute fabricated results.

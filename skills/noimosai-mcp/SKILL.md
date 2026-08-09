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
   - `mode: "draft"` — DEFAULT unless the user explicitly approved the exact text. The post appears in NoimosAI as a draft the user approves in-app.
   - `mode: "publish"` — immediate. Only after the user approved the exact content.
   - `mode: "schedule"` + `scheduleAt` — approved content, later time.
   - `dryRun: true` first when unsure the payload is right.

Never publish or schedule content the user has not seen. Drafts are always safe.

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

## Ground rules

- `workspaceId` comes from config or `list_workspaces` — never invent one.
- Account-targeted calls need the `providerAccountId` from `get_workspace_context` / `list_integrations`.
- Write user-facing content in the workspace's output language (`aiResponseLanguage`) unless the user says otherwise.
- Report tool errors plainly with what you tried; don't silently substitute fabricated results.

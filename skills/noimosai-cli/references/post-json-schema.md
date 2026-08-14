# NoimosPostJson

The format `noimosai chat --output json` emits and `noimosai post` consumes.
Canonical definition: `post-json-schema.ts` in `@agos-labs/noimos-client-core`.

## Top level

```typescript
{
  version: "1",                    // required, always "1"
  metadata: {
    generatedAt: string,           // ISO 8601
    sessionId?: string | null,     // pass to `chat -r` to continue the session
    prompt?: string
  },
  output?: string | null,          // the agent's prose reply
  posts: NoimosPostEntry[],
  canvas?: { kind: "a2ui" | "openui", title?, actionSummary?, markdown?, openuiLang? },
  pendingQuestions?: NoimosPendingQuestion[]
}
```

**`pendingQuestions` means the work is PAUSED, not finished.** The agent ended
the turn waiting on the user. Show the questions, collect answers, and send them
back on the SAME session (`chat -r <sessionId> -p "<answers>"`). Treating a
paused turn as a finished one silently drops the deliverable.

```typescript
NoimosPendingQuestion = {
  slug?: string,        // stable id to answer against; falls back to `question`
  question: string,
  header: string,
  subtitle?: string,
  detail?: string,      // long-form markdown to review (e.g. an outline)
  multiSelect: boolean,
  options: { label: string, description?: string }[],
  allowFileUpload?: boolean,   // needs a file — not answerable by text alone
  goal?: string
}
```

## Post entry

| Field | Type | Required | Notes |
|---|---|---|---|
| `platform` | string | Yes | `x` `threads` `facebook` `instagram` `youtube` `linkedin` `tiktok` `bluesky` `pinterest` `snapchat` `mastodon` `note`. Chat display names (`X (Twitter)`, `Instagram`, …) and short aliases (`fb` `ig` `yt` `twitter`) are accepted too. |
| `dataKey` | string | No | Internal routing key — **omit it**; it is derived from `platform`. Present in chat output: pass through unchanged. |
| `providerAccountId` | string | To publish | From `noimosai integration list`, or already filled in chat output. Required in the payload for `note`. |
| `username` | string | No | Display only. |
| `textBlocks` | `{label?, text}[]` | Yes | On x / threads / bluesky / mastodon each entry becomes one thread item. Elsewhere they are joined with a blank line. At least one non-empty `text`. |
| `media` | `{url?, path?, mimeType?}[]` | No | Each item needs `url` OR `path`. `path` is a workspace storage path (`upload-media` output); `url` is a public URL the server downloads. |
| `options` | object | Pinterest only | Per-platform overrides — see below. |

WordPress and Substack are **not** post targets and have no dataKey here. Long
form goes through `noimosai publish-article`.

## Per-platform `options`

Only the block matching the entry's platform is read; the others are ignored
silently. Everything is derived from the entry unless overridden — set a field
only to change the default.

| Platform | Media | Must set | Derived |
|---|---|---|---|
| x, threads, bluesky, mastodon | optional | — | one thread item per `textBlocks` entry |
| facebook, linkedin | optional | — | — |
| instagram | **required** | — | `igPostType`: REELS when the media is a video, else CAROUSEL |
| tiktok | **required** | — | `title` = caption's first line (≤90); `postType` from the media mime |
| youtube | **required — a video** | — | `title` = first line (≤100); `categoryId` `"22"`; `privacyStatus` PUBLIC |
| pinterest | **required** | `options.pinterest.boardId` | `description` = the text |
| snapchat | **required** | — | `postType` SPOTLIGHT |
| note | optional | — | `title` = first line; `isDraft` follows the post mode |

```typescript
options = {
  tiktok?:    { title?, postType?: "VIDEO"|"IMAGE",
                privacyLevel?: "PUBLIC_TO_EVERYONE"|"MUTUAL_FOLLOW_FRIENDS"|"FOLLOWER_OF_CREATOR"|"SELF_ONLY",
                disableDuet?, disableStitch?, disableComment?, isAigc? },
  youtube?:   { title?, tags?: string[], categoryId?,
                privacyStatus?: "PUBLIC"|"UNLISTED"|"PRIVATE", publicStatsViewable? },
  pinterest?: { boardId: string, title?, link?, altText? },      // boardId REQUIRED
  instagram?: { igPostType?: "CAROUSEL"|"REELS"|"STORIES", altText?,
                userTags?: { username: string, x?: number, y?: number }[] },
  snapchat?:  { postType?: "STORY"|"SPOTLIGHT"|"SAVED_STORY", link? },
  note?:      { title? }
}
```

**note's cover image is a URL, not a path.** The first `media[]` item with a
public `url` becomes the header; a `path`-only `upload-media` result is not
resolved for note. Instagram `altText` applies to image posts only.

## Validation

Checked before anything is sent, so one bad entry publishes nothing in the whole
batch. The message names the field.

- `textBlocks` must produce non-empty text.
- No `media[]` item may have neither `url` nor `path`.
- instagram / tiktok / pinterest / snapchat need ≥1 media item; youtube needs one
  with a `video/*` mimeType.
- pinterest needs `options.pinterest.boardId` (`noimosai pinterest-boards <providerAccountId>`).
- note needs `providerAccountId`.

Filling another platform's `options` block is not an error — it is ignored. When
a required field is missing and a foreign block is filled, the error says so;
that is almost always the actual mistake.

## Schedule format

`noimosai post --schedule "YYYY-MM-DD HH:MM"` — local time, at least 30 minutes
out, converted to ISO 8601 UTC internally.

## Example

```json
{
  "version": "1",
  "metadata": {
    "generatedAt": "2026-08-14T10:30:00.000Z",
    "sessionId": "sess_abc123",
    "prompt": "Create posts about AI safety for X and Pinterest"
  },
  "posts": [
    {
      "platform": "x",
      "providerAccountId": "acct_x_12345",
      "textBlocks": [
        { "text": "AI safety is one of the most important topics of our time. Here's why it matters. 🧵" },
        { "text": "AI systems are already deployed in healthcare, transport and finance. Without safety work, the downside is not hypothetical." },
        { "text": "Support alignment research, push for workable regulation, stay informed." }
      ]
    },
    {
      "platform": "pinterest",
      "providerAccountId": "acct_pin_67890",
      "textBlocks": [{ "text": "AI safety, explained in one chart." }],
      "media": [{ "path": "workspaces/ws_1/media/ai-safety.png", "mimeType": "image/png" }],
      "options": { "pinterest": { "boardId": "1234567890", "link": "https://example.com/ai-safety" } }
    }
  ]
}
```

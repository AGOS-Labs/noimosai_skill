# NoimosPostJson Schema

The JSON format used for data exchange between `noimosai chat --output json` and `noimosai post`.

## Top-Level Structure

```typescript
{
  version: "1",                          // Required: always "1"
  metadata: {
    generatedAt: string,                 // ISO 8601 timestamp
    sessionId?: string | null,           // Session ID for conversation continuity
    prompt?: string                      // Original prompt
  },
  posts: NoimosPostEntry[]               // Array of post entries
}
```

## Post Entry Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `platform` | string | Yes | Platform name (e.g., `"X (Twitter)"`, `"Instagram"`) |
| `dataKey` | string | No | Platform data key — derived from `platform` when omitted (see mapping below). Pass through unchanged when it came from `chat` output |
| `providerAccountId` | string | For publishing | Account ID (included in `chat --output json` output) |
| `username` | string | No | Display username |
| `textBlocks` | array | Yes | Array of `{ label?: string, text: string }` |
| `media` | array | No | Array of `{ url?: string, mimeType?: string, path?: string }` |

## Platform to dataKey Mapping

| Platform | dataKey |
|----------|---------|
| X (Twitter) | `xPostData` |
| Threads | `thPostData` |
| Facebook | `fbPostData` |
| Instagram | `igPostData` |
| YouTube | `ytPostData` |
| LinkedIn | `linkedinPostData` |
| TikTok | `ttPostData` |
| Bluesky | `blueskyPostData` |
| Pinterest | `pinterestPostData` |
| Note | `notePostData` |
| WordPress | `wpPostData` |

## Example

```json
{
  "version": "1",
  "metadata": {
    "generatedAt": "2026-02-23T10:30:00.000Z",
    "sessionId": "sess_abc123",
    "prompt": "Create posts about AI safety for X and Instagram"
  },
  "posts": [
    {
      "platform": "X (Twitter)",
      "dataKey": "xPostData",
      "providerAccountId": "acct_x_12345",
      "username": "@myaccount",
      "textBlocks": [
        { "label": "Thread 1", "text": "AI safety is one of the most important topics of our time. Here's why it matters for everyone. 🧵" },
        { "label": "Thread 2", "text": "First, AI systems are being deployed in critical areas: healthcare, transportation, and finance. Without proper safety measures, the risks are enormous." },
        { "label": "Thread 3", "text": "What can we do? Support organizations working on AI alignment, advocate for responsible regulation, and stay informed. The future of AI depends on the choices we make today." }
      ],
      "media": []
    },
    {
      "platform": "Instagram",
      "dataKey": "igPostData",
      "providerAccountId": "acct_ig_67890",
      "username": "@myinstagram",
      "textBlocks": [
        { "text": "AI safety isn't just a tech issue — it's a human issue. As AI becomes more powerful, ensuring it works for everyone becomes critical.\n\n#AISafety #ResponsibleAI #TechForGood #AIEthics" }
      ],
      "media": [
        { "url": "https://example.com/ai-safety-infographic.png", "mimeType": "image/png" }
      ]
    }
  ]
}
```

## Validation Rules

- `version` must be `"1"`
- `posts` must be an array
- Each post must have `platform` (string) and `textBlocks` (array); `dataKey` is optional (derived from `platform`)
- Each `textBlocks` entry must have `text` (string); `label` is optional
- `media` is optional; each entry is an object with optional `url`, `mimeType`, `path`
- `providerAccountId` is required when publishing via `noimosai post` (included automatically in `chat --output json` output)

## Schedule Format

When using `noimosai post --schedule`:
- Format: `YYYY-MM-DD HH:MM` (local time)
- Must be at least 30 minutes from current time
- Internally converted to ISO 8601 UTC

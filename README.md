# BlackTwist MCP Server

The BlackTwist MCP (Model Context Protocol) server allows AI assistants like Claude, Cursor, and other MCP-compatible clients to interact with your BlackTwist account — creating posts, checking analytics, managing drafts, and more.

## Quick Start

There are two ways to connect to the BlackTwist MCP server: **OAuth** (recommended for claude.ai) or **API Key** (for desktop clients and CLI tools).

### Option A: Connect via OAuth (claude.ai)

If you're using **claude.ai** as a custom connector, OAuth is the easiest option — no API key needed.

1. Go to **claude.ai** > **Settings** > **Integrations** (or add a custom connector)
2. Enter the MCP server URL: `https://blacktwist.app/api/mcp`
3. Claude will automatically discover the OAuth endpoints, register itself, and redirect you to sign in with your BlackTwist account
4. Once signed in, the connection is established — Claude can now use your BlackTwist account

OAuth tokens refresh automatically, so you won't need to re-authenticate unless you revoke access.

### Option B: Connect via API Key

For **Claude Desktop**, **Claude Code**, **Cursor**, and other MCP clients that don't support OAuth, use an API key.

#### 1. Generate an API Key

1. Open **BlackTwist** and go to **Settings** (gear icon)
2. Click the **MCP** tab in the sidebar
3. Click **Create API Key**, give it a name (e.g. "Claude Desktop"), and click **Create Key**
4. Copy the key immediately — it won't be shown again

#### 2. Configure Your MCP Client

Add the following to your MCP client configuration:

**Claude Desktop** (`claude_desktop_config.json`):

```json
{
  "blacktwist": {
    "command": "npx",
    "args": [
      "-y",
      "mcp-remote@latest",
      "https://blacktwist.app/api/mcp",
      "--header",
      "Authorization:${BLACKTWIST_TOKEN}"
    ],
    "env": {
      "BLACKTWIST_TOKEN": "Bearer YOUR_API_KEY"
    }
  }
}
```

You need node.js v22, if you are using `nvm` be sure to use the correct version:

```json
{
  "blacktwist": {
    "command": "/Users/<your_user>/.nvm/versions/node/v22.22.0/bin/npx",
    "args": [
      "-y",
      "mcp-remote@latest",
      "https://blacktwist.app/api/mcp",
      "--header",
      "Authorization:${BLACKTWIST_TOKEN}"
    ],
    "env": {
      "BLACKTWIST_TOKEN": "Bearer YOUR_API_KEY",
      "PATH": "/Users/<your_user>/.nvm/versions/node/v22.22.0/bin:/usr/local/bin:/usr/bin:/bin",
      "NODE_PATH": "/Users/<your_user>/.nvm/versions/node/v22.22.0/lib/node_modules"
    }
  }
}
```

**Claude Code**

Run in the terminal:

```bash
claude mcp add --transport http blacktwist https://blacktwist.app/api/mcp --header "Authorization: Bearer YOUR_API_KEY"
```

Or edit the file `.mcp.json` in project root:

```json
{
  "mcpServers": {
    "blacktwist": {
      "type": "url",
      "url": "https://blacktwist.app/api/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    }
  }
}
```

**Cursor** (Settings > MCP):

- **Name:** `blacktwist`
- **Type:** `url`
- **Server URL:** `https://blacktwist.app/api/mcp`
- **Headers:** `Authorization: Bearer YOUR_API_KEY`

### 3. Start Using It

Once connected, you can ask your AI assistant things like:

- "List my connected social accounts"
- "Schedule a post on Threads for tomorrow at 9am saying: Just shipped a new feature!"
- "Edit my latest draft to say: Updated copy with better hook"
- "Add a second post to my thread about product launches"
- "Show me my analytics for the last 7 days"
- "What are my best posting times?"
- "List my upcoming scheduled posts"
- "Tag my latest draft with growth and product launch"

---

## Server Details

| Property      | Value                                      |
| ------------- | ------------------------------------------ |
| **URL**       | `https://blacktwist.app/api/mcp`           |
| **Auth**      | OAuth 2.1 (PKCE) or Bearer token (API key) |
| **Transport** | Streamable HTTP (stateless)                |
| **Protocol**  | MCP via JSON-RPC over HTTP POST            |

---

## Team Context (`teamId`)

Most tools accept an optional `teamId` parameter to scope operations to a specific team.

**Behavior:**

1. If `teamId` is a team ID, the tool operates in that team's context (membership is verified).
2. If `teamId` is `"personal"`, the tool operates in **personal mode** (user's own data only, no team).
3. If `teamId` is omitted, the tool falls back to the user's **currently active team** (from user settings). If no team is active, this is equivalent to `"personal"`.

**Tools that support `teamId`:** `list_providers`, `list_posts`, `list_drafts`, `create_post`, `get_thread`, `delete_thread`, `reschedule_thread`, `list_time_slots`, `list_tags`, `get_subscription`, `get_follow_up_templates`, and all analytics tools.

**Tools without `teamId`:** `list_teams` (lists all teams), `get_user_settings` (personal settings), `list_viral_templates` / `list_viral_template_categories` (account-wide content library), `edit_post` / `edit_thread` / `get_thread_follow_up` / `set_thread_follow_up` / `get_thread_tags` / `set_thread_tags` (thread-based access handles team auth internally via `userCanAccessThread`).

**Subscription access:** Analytics tools that require a paid plan will check the user's own subscription first. If the user doesn't have one, the system also checks whether any team owner the user belongs to has an active plan. This means team members can access paid features through their team owner's subscription.

---

## Available Tools

### Providers

#### `list_providers`

List all connected social media accounts (Threads and Bluesky).

| Parameter | Type   | Required | Description                                                                                                                                               |
| --------- | ------ | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `teamId`  | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account. When set to a team, shows only providers linked to that team. |

**Returns:** Array of providers with `id`, `provider`, `providerUserId`, `providerUsername`, `displayName`, `profilePicture`.

> **Note:** Many other tools require a `providerId` (the `id` field) or `providerUserId` from this response.

---

### Teams

#### `list_teams`

List all teams the user belongs to, including their role and which team is currently active. Always includes a `"personal"` entry representing the user's personal (non-team) account.

**Parameters:** None

**Returns:** Array of teams with `id`, `name`, `role` (`OWNER`, `ADMIN`, `MEMBER`, `GUEST`), `isActive`, `createdAt`.

The personal entry has `id: "personal"` and is marked `isActive: true` when no team is currently selected.

> **Note:** The team `id` can be passed as `teamId` to other tools (`list_posts`, `list_drafts`, `create_post`, etc.). Pass `"personal"` to explicitly use the personal account. When `teamId` is omitted, those tools automatically use the user's currently active team.

---

### Posts & Threads

#### `create_post`

Create a new post or thread. A thread is multiple posts linked together.

| Parameter    | Type   | Required | Description                                                                                                                                                              |
| ------------ | ------ | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `providerId` | string | Yes      | Provider ID from `list_providers`                                                                                                                                        |
| `posts`      | array  | Yes      | Array of post objects (see below)                                                                                                                                        |
| `scheduleAt` | string | No       | ISO 8601 datetime. If omitted, saves as draft. Times without a timezone offset (e.g. `2025-03-15T09:00:00`) are interpreted in the user's configured timezone.           |
| `autoRepost` | object | No       | `{ enabled: boolean, delays: string[] }` where delays are hours, e.g. `["168", "336"]` for 7 and 14 days. If omitted, the user's default auto-repost setting is applied. |
| `teamId`     | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account.                                                                              |

**Default behaviors applied automatically:**

- **Auto-repost:** If `autoRepost` is not provided and the user has auto-repost enabled in settings, the default delays are applied to the first post.
- **Follow-up (auto-plug):** If the provider has a default follow-up template enabled, it is automatically copied and attached to the thread. The response includes `autoPlugApplied: true` when this happens.
- **Timezone:** Times without an offset are interpreted in the user's `notificationsTimezone` setting.

Each post object:

| Field            | Type     | Required | Description                                                                                       |
| ---------------- | -------- | -------- | ------------------------------------------------------------------------------------------------- |
| `text`           | string   | Yes      | Post content                                                                                      |
| `topic`          | string   | No       | Topic tag                                                                                         |
| `media`          | object[] | No       | Media attachments via public URL (see [Media Attachments](#media-attachments))                    |
| `spoilerRanges`  | object[] | No       | Text spoiler ranges (see [Spoilers](#spoilers-threads-only))                                      |
| `isSpoilerMedia` | boolean  | No       | If `true`, all media in this post is marked as a spoiler (see [Spoilers](#spoilers-threads-only)) |

**Example — single post:**


```json
{
  "providerId": "clx...",
  "posts": [{ "text": "Hello world!" }],
  "scheduleAt": "2025-03-15T09:00:00"
}
```

**Example — thread:**

```json
{
  "providerId": "clx...",
  "posts": [
    { "text": "Thread time! Here's what I learned this week 🧵" },
    { "text": "1/ First lesson: consistency beats perfection" },
    { "text": "2/ Second lesson: engage with your community daily" }
  ],
  "scheduleAt": "2025-03-15T09:00:00"
}
```

#### Media Attachments

Tools that accept `media` (`create_post`, `edit_post`, `edit_thread`, `set_thread_follow_up`) let you attach images and videos via public URLs. The server downloads, validates against Threads specs, and uploads them automatically.

Each media object:

| Field     | Type   | Required | Description                       |
| --------- | ------ | -------- | --------------------------------- |
| `url`     | string | Yes      | Public URL of the image or video  |
| `altText` | string | No       | Alt text for accessibility        |

**Threads image specs:**

- Formats: JPEG, PNG
- Max file size: 8 MB
- Max aspect ratio: 10:1

**Threads video specs:**

- Formats: MP4, MOV
- Max file size: 1 GB
- Duration: up to 5 minutes
- Max width: 1920px

**Example — post with an image:**

```json
{
  "providerId": "clx...",
  "posts": [
    {
      "text": "Check out this view!",
      "media": [
        {
          "url": "https://example.com/photo.jpg",
          "altText": "Sunset over the mountains"
        }
      ]
    }
  ],
  "scheduleAt": "2025-03-15T09:00:00"
}
```

#### Spoilers (Threads only)

Spoilers allow you to hide portions of text or media behind a blur effect on Threads. Viewers tap to reveal the hidden content.

**Text spoilers** are defined as an array of character ranges:

| Field    | Type   | Required | Description                       |
| -------- | ------ | -------- | --------------------------------- |
| `offset` | number | Yes      | Character offset (0-indexed)      |
| `length` | number | Yes      | Number of characters in the range |

- Maximum 10 spoiler ranges per post
- Ranges refer to character positions in the `text` field

**Media spoilers** are a single boolean flag per post. When `isSpoilerMedia` is `true`, all media (images and videos) in the post is blurred until the viewer taps to reveal.

Text and media spoilers are independent — a post can have both, either, or neither.

**Example — post with text spoiler:**

```json
{
  "providerId": "clx...",
  "posts": [
    {
      "text": "The winner is John!",
      "spoilerRanges": [{ "offset": 14, "length": 5 }]
    }
  ]
}
```

**Example — post with media spoiler:**

```json
{
  "providerId": "clx...",
  "posts": [
    {
      "text": "Spoiler alert! Check the image",
      "media": [{ "url": "https://example.com/reveal.jpg" }],
      "isSpoilerMedia": true
    }
  ]
}
```

**In responses** (`list_posts`, `list_drafts`, `get_thread`), each post includes:

- `spoilerRanges`: array of `{ offset, length }` (empty if no text spoilers)
- `isSpoilerMedia`: boolean (`true` if media is spoilered)

---

#### `list_posts`

List scheduled and published posts within a date range.

| Parameter    | Type   | Required | Description                                                                                 |
| ------------ | ------ | -------- | ------------------------------------------------------------------------------------------- |
| `from`       | string | Yes      | Start date (ISO 8601)                                                                       |
| `to`         | string | Yes      | End date (ISO 8601)                                                                         |
| `providerId` | string | No       | Provider ID (from `list_providers`). If provided, only returns posts for this provider.     |
| `teamId`     | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account. |

#### `list_drafts`

List all draft posts (not yet scheduled).

| Parameter | Type   | Required | Description                                                                                 |
| --------- | ------ | -------- | ------------------------------------------------------------------------------------------- |
| `teamId`  | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account. |

#### `get_thread`

Get a specific thread with all its posts, media, and analytics.

| Parameter  | Type   | Required | Description                                                                                 |
| ---------- | ------ | -------- | ------------------------------------------------------------------------------------------- |
| `threadId` | string | Yes      | The thread ID                                                                               |
| `teamId`   | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account. |

#### `edit_post`

Edit the text content, topic, or media of a single post. Only posts with status `READY` (drafts or scheduled) can be edited — published, processing, or failed posts are rejected. Use `get_thread` first to retrieve post IDs.

| Parameter        | Type     | Required | Description                                                                                                            |
| ---------------- | -------- | -------- | ---------------------------------------------------------------------------------------------------------------------- |
| `postId`         | string   | Yes      | The post ID (from `get_thread`)                                                                                        |
| `content`        | string   | No       | New text content                                                                                                       |
| `topic`          | string   | No       | New topic tag. Pass `null` to remove the topic.                                                                        |
| `media`          | object[] | No       | Replace all media via public URL (see [Media Attachments](#media-attachments)). Omit to keep existing media unchanged. |
| `spoilerRanges`  | object[] | No       | Replace text spoiler ranges (see [Spoilers](#spoilers-threads-only)). Omit to keep unchanged. Pass `[]` to remove all. |
| `isSpoilerMedia` | boolean  | No       | Set media spoiler flag. `true` = blurred, `false` = remove spoiler. Omit to keep unchanged.                            |

At least one of `content`, `topic`, `media`, `spoilerRanges`, or `isSpoilerMedia` must be provided.

**Returns:** `postId`, `threadId`, `updated` (object indicating which fields were changed).

**Example — update text:**

```json
{
  "postId": "clx...",
  "content": "Updated post content"
}
```

**Example — update text and clear topic:**

```json
{
  "postId": "clx...",
  "content": "New content",
  "topic": null
}
```

#### `edit_thread`

Edit a thread by adding, removing, or reordering its posts. Provide the full desired list of posts — existing posts (with `id`) are updated, new posts (without `id`) are created, and existing posts not in the list are removed. Only threads where all posts have status `READY` can be edited.

| Parameter  | Type   | Required | Description                                                                          |
| ---------- | ------ | -------- | ------------------------------------------------------------------------------------ |
| `threadId` | string | Yes      | The thread ID                                                                        |
| `posts`    | array  | Yes      | Ordered array of posts (min 1). Order determines `postOrder` (0-indexed). See below. |

Each post object:

| Field            | Type     | Required | Description                                                                    |
| ---------------- | -------- | -------- | ------------------------------------------------------------------------------ |
| `id`             | string   | No       | Post ID for existing posts. Omit to create a new post in this position.        |
| `text`           | string   | Yes      | Post content                                                                   |
| `topic`          | string   | No       | Topic tag                                                                      |
| `media`          | object[] | No       | Media attachments via public URL (see [Media Attachments](#media-attachments)) |
| `spoilerRanges`  | object[] | No       | Text spoiler ranges (see [Spoilers](#spoilers-threads-only))                   |
| `isSpoilerMedia` | boolean  | No       | If `true`, all media in this post is marked as a spoiler                       |

New posts inherit `scheduledAt`, `provider`, `providerUserId`, and `teamId` from the existing thread.

**Returns:** `threadId`, `postsUpdated`, `postsCreated`, `postsDeleted`, `totalPosts`.

**Example — reorder and add a post:**

```json
{
  "threadId": "abc123",
  "posts": [
    { "id": "existing-post-2", "text": "Now this is first" },
    { "id": "existing-post-1", "text": "Now this is second" },
    { "text": "Brand new third post" }
  ]
}
```

**Example — remove a post (omit it from the list):**

```json
{
  "threadId": "abc123",
  "posts": [{ "id": "existing-post-1", "text": "Keep only this one" }]
}
```

#### `delete_thread`

Permanently delete a thread and all its posts.

| Parameter  | Type   | Required | Description                                                                                 |
| ---------- | ------ | -------- | ------------------------------------------------------------------------------------------- |
| `threadId` | string | Yes      | The thread ID                                                                               |
| `teamId`   | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account. |

#### `reschedule_thread`

Change the scheduled date/time for a thread.

| Parameter     | Type   | Required | Description                                                                                                 |
| ------------- | ------ | -------- | ----------------------------------------------------------------------------------------------------------- |
| `threadId`    | string | Yes      | The thread ID                                                                                               |
| `scheduledAt` | string | Yes      | New datetime (ISO 8601). Times without a timezone offset are interpreted in the user's configured timezone. |
| `teamId`      | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account.                 |

---

### Tags

Tags organize posts (drafts, scheduled, and published) and can be used to filter the Posts Analytics view in the app. Tags are scoped per workspace: team tags are shared with the whole team, personal tags are private. Tags apply to a whole thread, not to individual posts within it.

#### `list_tags`

List the tags available in the workspace for organizing posts.

| Parameter | Type   | Required | Description                                                                                 |
| --------- | ------ | -------- | ------------------------------------------------------------------------------------------- |
| `teamId`  | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account. |

**Returns:** `{ tags }` — array of tags with `id`, `name`, `color`.

#### `get_thread_tags`

Get the tags currently on a thread.

| Parameter  | Type   | Required | Description                    |
| ---------- | ------ | -------- | ------------------------------ |
| `threadId` | string | Yes      | The thread ID to get tags for |

**Returns:** `{ threadId, tags }` where `tags` is an array of `{ id, name, color }`.

#### `set_thread_tags`

Set the tags on a thread. This single tool covers **setting, updating, and removing** tags: it replaces the thread's existing tags with the exact set you pass in.

| Parameter  | Type     | Required | Description                                                                                                                                                                         |
| ---------- | -------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `threadId` | string   | Yes      | The thread ID to set tags for                                                                                                                                                       |
| `tagNames` | string[] | Yes      | The **full desired set** of tag names for the thread. Replaces existing tags — not additive. Names that don't exist yet are created automatically. Pass `[]` to remove all tags. |

**Behavior:**

- Names are trimmed, empty entries dropped, and duplicates removed case-insensitively (`"Growth"` and `"growth"` are the same tag).
- New tag names are created automatically in the thread's workspace scope (team thread → team tag, personal thread → personal tag) with an auto-assigned color.
- Tag names are limited to 40 characters.

**Returns:** `{ threadId, tags }` — the final tag set on the thread.

**Example — tag a thread:**

```json
{
  "threadId": "abc123",
  "tagNames": ["growth", "product launch"]
}
```

**Example — remove all tags:**

```json
{
  "threadId": "abc123",
  "tagNames": []
}
```

---

### Analytics

All analytics tools require a `providerUserId` (from `list_providers`) and accept an optional `teamId` to scope access.

> **Paid plan required:** `get_metric_timeseries`, `get_post_analytics`, `get_follower_growth`, and `get_best_time_to_post` require a paid plan (user or team owner). Free-plan users will receive: _"A paid plan is required to access the analytics. Subscribe at https://blacktwist.app#pricing"_. The remaining tools (`get_live_metrics`, `get_consistency`, `get_daily_recap`, `get_recommendations`) are available on all plans.
>
> **Date ranges:** The `from` and `to` dates are both inclusive. For example, `from: "2026-03-01"` and `to: "2026-03-08"` includes all data from March 1 through the end of March 8.

#### `get_live_metrics`

Get engagement metrics with percentage change vs. the previous period.

| Parameter        | Type   | Required | Description                                                                                 |
| ---------------- | ------ | -------- | ------------------------------------------------------------------------------------------- |
| `providerUserId` | string | Yes      | Provider user ID                                                                            |
| `from`           | string | No       | Start date (defaults to 7 days ago)                                                         |
| `to`             | string | No       | End date (defaults to now)                                                                  |
| `teamId`         | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account. |

**Returns:** `views`, `likes`, `replies`, `reposts`, `quotes` — each with `value` and `percentageChange`.

#### `get_metric_timeseries`

Get daily data points for a specific metric over a date range.

| Parameter        | Type   | Required | Description                                                                                    |
| ---------------- | ------ | -------- | ---------------------------------------------------------------------------------------------- |
| `providerUserId` | string | Yes      | Provider user ID                                                                               |
| `metric`         | enum   | Yes      | One of: `VIEWS`, `LIKES`, `REPLIES`, `REPOSTS`, `QUOTES`, `ENGAGEMENT_RATE`, `FOLLOWERS_COUNT` |
| `from`           | string | Yes      | Start date (ISO 8601)                                                                          |
| `to`             | string | Yes      | End date (ISO 8601)                                                                            |
| `teamId`         | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account.    |

#### `get_post_analytics`

Get per-post engagement metrics for posts within a date range. Each post also carries its thread continuation parts (self-replies), so multi-post threads are returned in full.

| Parameter            | Type    | Required | Description                                                                                                                 |
| -------------------- | ------- | -------- | --------------------------------------------------------------------------------------------------------------------------- |
| `providerUserId`     | string  | Yes      | Provider user ID                                                                                                            |
| `from`               | string  | Yes      | Start date (ISO 8601)                                                                                                       |
| `to`                 | string  | Yes      | End date (ISO 8601)                                                                                                         |
| `includeThreadParts` | boolean | No       | Include each post's thread continuation parts (self-replies) in a `thread` array. Defaults to `true`; pass `false` to omit. |
| `teamId`             | string  | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account.                                 |

**Returns:** Up to 100 posts with `views`, `likes`, `replies`, `reposts`, `quotes`, `engagementRate`. Each post also includes a `thread` array of its continuation parts, reconstructed from the user's self-replies — so a multi-post thread comes back with all of its posts. Each part has `replyId`, `text` (truncated to 200 characters), `permalink`, `timestamp`, `partOrder` (starting at `2`; the root post is part 1), and its own `analytics` (`views`, `likes`, `replies`, `reposts`, `quotes`, `engagementRate`). Single posts return an empty `thread`. Pass `includeThreadParts: false` to skip reconstruction entirely and return empty `thread` arrays.

#### `get_follower_growth`

Get daily follower count over time.

| Parameter        | Type   | Required | Description                                                                                 |
| ---------------- | ------ | -------- | ------------------------------------------------------------------------------------------- |
| `providerUserId` | string | Yes      | Provider user ID                                                                            |
| `from`           | string | Yes      | Start date (ISO 8601)                                                                       |
| `to`             | string | Yes      | End date (ISO 8601)                                                                         |
| `teamId`         | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account. |

**Returns:** Daily `followers` data points and `totalGrowth`.

#### `get_consistency`

Get a 365-day posting consistency heatmap.

| Parameter        | Type   | Required | Description                                                                                 |
| ---------------- | ------ | -------- | ------------------------------------------------------------------------------------------- |
| `providerUserId` | string | Yes      | Provider user ID                                                                            |
| `teamId`         | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account. |

**Returns:** `days` (array of `{ date, count }`), `totalPosts`, `daysWithPosts`, `currentStreak`, `recordStreak`.

#### `get_daily_recap`

Get yesterday's summary.

| Parameter        | Type   | Required | Description                                                                                 |
| ---------------- | ------ | -------- | ------------------------------------------------------------------------------------------- |
| `providerUserId` | string | Yes      | Provider user ID                                                                            |
| `teamId`         | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account. |

**Returns:** `date`, `newFollowers`, `postsCount`.

#### `get_recommendations`

Get posting recommendations based on your analytics patterns.

| Parameter        | Type   | Required | Description                                                                                 |
| ---------------- | ------ | -------- | ------------------------------------------------------------------------------------------- |
| `providerUserId` | string | Yes      | Provider user ID                                                                            |
| `teamId`         | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account. |

**Returns:** `bestPostingHoursUtc`, `topPerformingPosts`, `currentStreak`, `recordStreak`.

#### `get_best_time_to_post`

Get the best times to post based on historical engagement over the last 10 weeks. Mirrors the "Best time to post" analytics chart — engagement bucketed by hour of day, both averaged across the whole week and broken down per day of the week. Hours are reported in the user's own timezone.

> **Active subscription required:** This tool requires an active subscription — the user must have their own paid plan, or be part of a **team** whose owner has an active subscription. Otherwise it returns: _"The best time to post requires an active subscription. You need your own paid plan, or to be part of a team whose owner has an active subscription. Subscribe at https://blacktwist.app#pricing"_

> **Timezone:** Resolved automatically server-side from the user's `notificationsTimezone` setting (defaulting to UTC). MCP callers normally cannot know the user's timezone, so they should leave `timezoneOffset` unset and let it be derived.

| Parameter        | Type   | Required | Description                                                                                                                                                                                                                     |
| ---------------- | ------ | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `providerUserId` | string | Yes      | Provider user ID                                                                                                                                                                                                              |
| `timezoneOffset` | number | No       | **Override** for the timezone used to bucket hours. Leave unset to use the user's configured account timezone (recommended). If set, use minutes matching JavaScript's `Date.getTimezoneOffset()` (behind UTC: UTC+2 is `-120`). |
| `view`           | enum   | No       | `"average"`, `"daily"`, or `"both"` (default). Selects which breakdown to return.                                                                                                                                            |
| `teamId`         | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account.                                                                                                                                  |

**Returns:** `provider`, `timezone` (IANA name, or `"custom"` when an override offset is used), `timezoneOffsetMinutes`, `timezoneInfo` (a human-readable note naming the timezone used and linking to `https://blacktwist.app/write?settings=notifications` so the user can correct it if it's wrong), and (depending on `view`) `averageView` (`bestHours` + 24 hourly `averageEngagement` values across the whole week) and `dailyView` (7 days, each with `dayName`, `bestHour`, and 24 hourly `averageEngagement` values).

---

### Follow-Up (Auto-Plug)

The follow-up system automatically replies to your post after it reaches engagement thresholds (likes, replies, reposts) or a time delay.

#### `get_follow_up_templates`

List saved follow-up templates for a provider.

| Parameter        | Type   | Required | Description                                                                                                                                    |
| ---------------- | ------ | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `userProviderId` | string | Yes      | Provider `id` from `list_providers`                                                                                                            |
| `teamId`         | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account. Scopes which team members' templates are included. |

#### `get_thread_follow_up`

Get the follow-up configuration for a specific thread.

| Parameter  | Type   | Required | Description   |
| ---------- | ------ | -------- | ------------- |
| `threadId` | string | Yes      | The thread ID |

#### `set_thread_follow_up`

Set or update the follow-up for a thread.

| Parameter                 | Type     | Required | Description                                                      |
| ------------------------- | -------- | -------- | ---------------------------------------------------------------- |
| `threadId`                | string   | Yes      | The thread ID                                                    |
| `isEnabled`               | boolean  | Yes      | Enable/disable the follow-up                                     |
| `text`                    | string   | No       | Follow-up reply text                                             |
| `images`                  | string[] | No       | Image URLs                                                       |
| `delayTimeMinutes`        | number   | No       | Time delay trigger (minutes)                                     |
| `delayTimeMinutesEnabled` | boolean  | No       | Enable time delay trigger                                        |
| `numberOfLikes`           | number   | No       | Likes threshold                                                  |
| `numberOfLikesEnabled`    | boolean  | No       | Enable likes trigger                                             |
| `numberOfReplies`         | number   | No       | Replies threshold                                                |
| `numberOfRepliesEnabled`  | boolean  | No       | Enable replies trigger                                           |
| `numberOfReposts`         | number   | No       | Reposts threshold                                                |
| `numberOfRepostsEnabled`  | boolean  | No       | Enable reposts trigger                                                         |
| `media`                   | object[] | No       | Media attachments via public URL (see [Media Attachments](#media-attachments)) |

---

### Scheduling

#### `list_time_slots`

List configured posting time slots for a provider. Time slots define preferred scheduling times.

| Parameter    | Type   | Required | Description                                                                                 |
| ------------ | ------ | -------- | ------------------------------------------------------------------------------------------- |
| `providerId` | string | Yes      | Provider `id` from `list_providers`                                                         |
| `teamId`     | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account. |

**Returns:** Array of `{ timeSlotHour, timeSlotMinute, weekDays }` where `weekDays` is `[0-6]` (0 = Sunday).

---

### Viral Templates

Browse the library of viral post templates extracted from high-performing posts. Each template includes the structure, usage instructions, and the original example post with its engagement metrics.

> **Paid plan required:** Both tools below require an active paid plan (user-owned plan, Lifetime Deal, or membership in a workspace whose owner has a paid plan). Free-plan users will receive: _"A paid plan is required to access viral post templates"_.

#### `list_viral_templates`

List viral post templates with filtering, free-text search, sorting, and pagination.

| Parameter  | Type   | Required | Description                                                                                                                             |
| ---------- | ------ | -------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `search`   | string | No       | Free-text search. Matches against the example post text (case-insensitive).                                                             |
| `category` | string | No       | Filter to a single category. Use a value returned by `list_viral_template_categories` or the `categories` field of a previous response. |
| `sort`     | enum   | No       | Sort by example-post engagement: `views` (default), `likes`, `replies`, or `engagementRate`. Descending.                                |
| `page`     | number | No       | Page number, 1-indexed. Defaults to `1`.                                                                                                |
| `pageSize` | number | No       | Items per page (1–50). Defaults to `20`. Values above 50 are clamped to 50.                                                             |

**Returns:** `{ templates, categories, totalCount, page, pageSize }`.

Each template:

| Field            | Type   | Description                                               |
| ---------------- | ------ | --------------------------------------------------------- |
| `id`             | string | Template ID                                               |
| `category`       | string | Template category                                         |
| `template`       | string | The template structure                                    |
| `instructions`   | string | How to use the template                                   |
| `exampleText`    | string | The original viral post text                              |
| `views`          | number | Views on the example post                                 |
| `likes`          | number | Likes on the example post                                 |
| `replies`        | number | Replies on the example post                               |
| `engagementRate` | number | Engagement rate of the example post (likes+replies/views) |

#### `list_viral_template_categories`

List all available template categories with the number of templates in each. Useful as a discovery step before calling `list_viral_templates` with a `category` filter.

**Parameters:** None

**Returns:** `{ categories: [{ category, templateCount }] }`, ordered alphabetically by category.

---

### Account

#### `get_subscription`

Get your current plan, remaining post limits, and account quotas.

| Parameter | Type   | Required | Description                                                                                 |
| --------- | ------ | -------- | ------------------------------------------------------------------------------------------- |
| `teamId`  | string | No       | Team ID. If not provided, uses the active team. Pass `"personal"` for the personal account. |

**Returns:** `planType`, `planName`, `isFreePlan`, `remainingPosts`, `limits` (maxTeams, maxMembers, maxAccounts).

#### `get_user_settings`

Get your user settings including timezone, date format, and auto-repost config.

**Parameters:** None

**Returns:** `notificationsTimezone`, `dateFormat`, `autoRepostEnabled`, `autoRepostDelays`, and more.

---

## Common Workflows

### Schedule a post at your next free time slot

1. `list_providers` → get your provider ID
2. `list_time_slots` → find available slots
3. `create_post` → schedule with the next slot time

### Check weekly performance

1. `list_providers` → get your provider user ID
2. `get_live_metrics` → see this week vs. last week
3. `get_recommendations` → get improvement suggestions

### Edit an existing post

1. `get_thread` → retrieve the thread and its post IDs
2. `edit_post` → update the text, topic, or media of a specific post

### Add a post to an existing thread

1. `get_thread` → retrieve current posts with their IDs
2. `edit_thread` → pass all existing posts (with IDs) plus the new post (without ID)

### Create a post with follow-up

1. `create_post` → create and schedule the post (returns `threadId`)
2. `set_thread_follow_up` → attach a promotional reply

### Organize posts with tags

1. `create_post` → create the post (returns `threadId`)
2. `set_thread_tags` → tag it (e.g. `["growth", "tips"]`) — new tag names are created automatically
3. `list_tags` / `get_thread_tags` → discover existing tags or check what's on a thread

---

## Prompts

The MCP server includes built-in prompts &mdash; reusable templates that guide your AI assistant through multi-step workflows. In **Claude Desktop**, access them by typing `/` in the chat. In **Claude Code**, run `claude prompts` to list them.

### `weekly-report`

Generate a weekly performance report with engagement metrics, top posts, follower growth, and recommendations.

**Arguments:**

| Parameter        | Required | Description           |
| ---------------- | -------- | --------------------- |
| `providerUserId` | Yes      | Your provider user ID |

**Example:** _"Run the weekly-report prompt for my Threads account"_

### `content-ideas`

Generate post ideas based on your best-performing content and posting patterns.

**Arguments:**

| Parameter        | Required | Description                                                             |
| ---------------- | -------- | ----------------------------------------------------------------------- |
| `providerUserId` | Yes      | Your provider user ID                                                   |
| `topic`          | No       | Topic to focus on (e.g. &quot;productivity&quot;, &quot;startups&quot;) |

**Example:** _"Give me content ideas about marketing"_

### `optimize-schedule`

Analyze your posting patterns and suggest an optimal posting schedule for the week.

**Arguments:**

| Parameter        | Required | Description           |
| ---------------- | -------- | --------------------- |
| `providerUserId` | Yes      | Your provider user ID |
| `providerId`     | Yes      | Your provider ID      |

**Example:** _"Help me optimize my posting schedule"_

### `draft-review`

Review your current drafts and get suggestions to improve them before publishing.

**Arguments:** None

**Example:** _"Review my drafts and tell me how to improve them"_

### `monthly-recap`

Generate a comprehensive monthly recap with key metrics, top posts, and growth trends.

**Arguments:**

| Parameter        | Required | Description           |
| ---------------- | -------- | --------------------- |
| `providerUserId` | Yes      | Your provider user ID |

**Example:** _"Generate my monthly recap"_

---

## API Key Management

- **Create keys:** Settings > MCP > Create API Key
- **View keys:** Settings > MCP (shows prefix, creation date, last used)
- **Delete keys:** Click the trash icon next to any key
- **Endpoint:** `GET/POST/DELETE /api/user/mcp-keys`

Keys are hashed with SHA-256 before storage. The raw key is only shown once at creation time.

---

## Authentication

The MCP server supports two authentication methods. Both use the `Authorization: Bearer <token>` header.

### OAuth 2.1 (for claude.ai and OAuth-capable clients)

The server implements the [MCP Authorization specification](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization) with OAuth 2.1 and PKCE (S256). OAuth-capable clients like claude.ai handle the flow automatically.

**OAuth endpoints:**

| Endpoint                                      | Description                              |
| --------------------------------------------- | ---------------------------------------- |
| `GET /.well-known/oauth-protected-resource`   | Resource metadata (RFC 9728)             |
| `GET /.well-known/oauth-authorization-server` | Authorization server metadata (RFC 8414) |
| `POST /api/mcp/oauth/register`                | Dynamic client registration (RFC 7591)   |
| `GET /api/mcp/oauth/authorize`                | Authorization endpoint                   |
| `POST /api/mcp/oauth/token`                   | Token exchange and refresh               |
| `POST /api/mcp/oauth/revoke`                  | Token revocation (RFC 7009)              |

**How it works:**

1. The client discovers the OAuth endpoints via the well-known metadata
2. The client registers itself using dynamic client registration
3. The user is redirected to sign in with their BlackTwist account (uses the existing BlackTwist login)
4. After sign-in, an authorization code is issued and exchanged for access + refresh tokens
5. Access tokens expire after 1 hour and are refreshed automatically using the refresh token (30-day lifetime)

### API Keys (for desktop clients and CLI tools)

API keys are static tokens prefixed with `bt_mcp_`. They don't expire by default and are suitable for clients that don't support OAuth (Claude Desktop, Cursor, Claude Code CLI).

See [API Key Management](#api-key-management) for how to create and manage keys.

---

## Technical Details

- **Endpoint:** `POST /api/mcp` (JSON-RPC)
- **Transport:** `WebStandardStreamableHTTPServerTransport` from `@modelcontextprotocol/sdk`
- **Mode:** Stateless (no session management)
- **Auth:** OAuth 2.1 with PKCE (S256) or API key via `Authorization: Bearer <key>` header
- **Unauthenticated discovery:** `initialize` and `tools/list` can be called without authentication, so MCP registries (e.g. Glama) can inspect available tools
- **Rate limiting:** Authenticated requests are rate-limited to 60 req/min per IP. Unauthenticated discovery requests are limited to 10 req/min per IP.
- **Source code:** `src/libs/mcp/` (server, auth, oauth, tools), `src/app/api/mcp/` (route handler, OAuth endpoints)

# Posting What You Generate

Generate an image (or video/audio) with this skill, then publish it as a post on Civitai. The whole thing is two steps: run `generate.mjs`, then hand the image's remote URL to the Civitai MCP `create_post` tool. No manual upload, no base64, no file reading.

> **The one rule that saves you:** the `generate` output already gives you a remote `https` URL for every image. Pass that URL straight to `create_post`. Do NOT read the local PNG, do NOT base64-encode it, do NOT call `upload_image`. Encoding a local file is what makes a small model hit the "argument list too long" wall — skip it entirely.

## Step 1 — Generate

```bash
node generate.mjs wait --prompt "a single red apple on a white table" -n 1 -o ./out
```

When it finishes, the JSON printed to stdout includes a `remoteUrls` array. Each entry is a real download URL on Civitai's CDN, paired with the local file path it was saved to:

```json
{
  "workflowId": "1-20260612022340165",
  "status": "succeeded",
  "images": ["C:\\...\\out\\step_0-0.png"],
  "remoteUrls": [
    {
      "url": "https://orchestration-new.civitai.com/v2/consumer/blobs/XWEQ...JPEG?sig=...&exp=...",
      "type": "image",
      "path": "C:\\...\\out\\step_0-0.png"
    }
  ],
  "cost": { "total": 4 }
}
```

The field you want is **`remoteUrls[].url`**. That is the value you give to `create_post`. (The `images` array next to it is just local file paths on disk — those are NOT what you post.)

> The same `remoteUrls` array is also emitted by `node generate.mjs download --workflow-id <id> -o ./out` if you generated earlier and only want the URLs now.

## Step 2 — Connect to the Civitai MCP

Posting goes through the Civitai MCP server's `create_post` tool. Two ways to reach it:

- **You already have an MCP client** (Claude Code, Cursor, etc.): call the `create_post` tool directly. If the Civitai MCP isn't added yet: `claude mcp add --transport http civitai https://mcp.civitai.com/mcp`. Posting is a signed-in action, so set `CIVITAI_API_KEY` for the server.
- **No way to add MCP config in your runtime:** pull the standalone CLI and call the tool over the shell. It is a zero-dependency Node 18+ script — run it from anywhere (e.g. next to `generate.mjs`):

```bash
curl -fsSL https://mcp.civitai.com/cli -o mcp-cli.mjs
```

The CLI reads `CIVITAI_API_KEY` from the environment (the same key this skill uses). `create_post` requires it.

## Step 3 — Create the post

Pass the download URL from Step 1 as `images[].url`, and `publish: true` to make it live immediately (omit it or set `false` to leave a draft):

```bash
node mcp-cli.mjs call create_post '{"title":"My apple","images":[{"url":"<remoteUrls[0].url from generate output>"}],"publish":true}'
```

The MCP server fetches that URL and uploads it for you server-side, so a single `{url}` is all an image entry needs. On success it returns the post id:

```text
Post created and published.
ID: 29151695
Images attached: 1
```

The public post URL is:

```
https://civitai.com/posts/29151695
```

i.e. `https://civitai.com/posts/<id>`.

## End-to-end (copy-paste)

```bash
# 1. Generate one image and capture the JSON
node generate.mjs wait --prompt "a single red apple on a white table" -n 1 -o ./out > result.json

# 2. Pull the remote download URL out of the result (jq, or read the JSON yourself)
URL=$(jq -r '.remoteUrls[0].url' result.json)

# 3. Get the MCP CLI (skip if you already have it / use an MCP client instead)
curl -fsSL https://mcp.civitai.com/cli -o mcp-cli.mjs

# 4. Create + publish the post (CIVITAI_API_KEY must be set)
node mcp-cli.mjs call create_post "{\"title\":\"Red apple\",\"images\":[{\"url\":\"$URL\"}],\"publish\":true}"
# -> Post created and published. ID: <id>  ->  https://civitai.com/posts/<id>
```

If you don't have `jq`, just read `result.json`, copy the `remoteUrls[0].url` string, and paste it into the `create_post` call.

## `create_post` parameters

The fields you'll use most (see `node mcp-cli.mjs schema create_post` for the full list):

| Field | Required | Description |
|-------|----------|-------------|
| `images` | yes | Array of image entries, in order. Each entry is `{ "url": "<remote download URL>" }`. You can also use `{ "uuid": "..." }` for a pre-uploaded image, but with this skill you always have a `url`, so use that. |
| `title` | no | Post title |
| `detail` | no | Post description (HTML or plain text) |
| `publish` | no | `true` to publish immediately, otherwise it stays a draft (default `false`) |
| `tags` | no | Array of tag names |

Each image entry may also carry `width`, `height`, and `type` (`image` / `video` / `audio`), but they are optional — `{ "url": "..." }` is enough. For a video or audio post, set `"type": "video"` / `"type": "audio"` on the entry and use the matching `remoteUrls[].type`.

## Common mistakes

- **Base64-ing the local file** — never do this. Use `remoteUrls[].url`. Encoding the PNG into the command is what triggers "argument list too long" and burns tokens.
- **Posting the local path** — `images` is a local file path; it is not a URL and `create_post` can't use it. Use `remoteUrls[].url`.
- **Forgetting `CIVITAI_API_KEY`** — browsing MCP tools work without a key, but `create_post` is a signed-in write action and needs one.
- **Forgetting `publish`** — without `"publish": true` the post is created as a draft and won't be visible publicly.

# Posting What You Generate

Generate an image with this skill (it saves the file locally), then post it with one command:

```bash
node mcp-cli.mjs post-image ./out/step_0-0.png --title "My render"
```

That single call reads the local file, uploads it, creates the post, and publishes it. On success it prints the public post URL:

```text
Post published.
ID: 29151695
URL: https://civitai.com/posts/29151695
```

## End-to-end

```bash
# 1. Generate one image (saved to ./out)
node generate.mjs wait --prompt "a single red apple on a white table" -n 1 -o ./out

# 2. Post the saved PNG (pull the CLI first if you don't have it; CIVITAI_API_KEY required)
curl -fsSL https://mcp.civitai.com/cli -o mcp-cli.mjs   # skip if you already have it
node mcp-cli.mjs post-image ./out/step_0-0.png --title "Red apple"
# -> Post published. URL: https://civitai.com/posts/<id>
```

The local path is the `images[0]` entry from the generate output (e.g. `out/step_0-0.png`). `post-image` handles the upload and publish for you — no base64, no remote URL, no `upload_image` call.

## Flags

| Flag | Description |
|------|-------------|
| `--title "..."` | Post title (defaults to the filename) |
| `--detail "..."` | Post description (HTML or plain text) |
| `--nsfw <level>` | NSFW level |
| `--draft` | Leave the post as a draft instead of publishing |
| `--json` | Print the raw result JSON |

`CIVITAI_API_KEY` must be set (the same key this skill uses) — posting is a signed-in write action.

## Advanced

`post-image` works for images. For videos/audio, or to attach a pre-uploaded image by UUID, use `create_post` directly (`node mcp-cli.mjs schema create_post`); each image entry takes a `{ uuid }` or a remote `{ url }`. Note that the `remoteUrls[].url` in the generate output is a short-lived signed orchestration URL that expires quickly, so prefer posting the local file with `post-image`.

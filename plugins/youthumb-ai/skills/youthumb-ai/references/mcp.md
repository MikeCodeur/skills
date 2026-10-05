# YouThumb MCP server

Server URL: `https://www.youthumb.ai/api/mcp` (streamable HTTP, OAuth sign-in — no API key).
Docs: https://www.youthumb.ai/en/docs/agents/mcp

## Connect

| Client                       | How                                                                                                                  |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Claude (claude.ai / desktop) | **Settings → Connectors → Add custom connector**, paste the URL, **Connect**                                         |
| ChatGPT                      | Connector settings (developer mode): new connector with the URL and OAuth authentication                             |
| Claude Code                  | `claude mcp add --transport http youthumb https://www.youthumb.ai/api/mcp` then `/mcp` → youthumb → **Authenticate** |
| Codex                        | `codex mcp add youthumb --url https://www.youthumb.ai/api/mcp` then `codex mcp login youthumb`                       |

Equivalent JSON config for clients that read an `.mcp.json`:

```json
{
  "mcpServers": {
    "youthumb": {"type": "http", "url": "https://www.youthumb.ai/api/mcp"}
  }
}
```

Claude Code: list with `claude mcp list`, remove with `claude mcp remove youthumb` (add `-s local` if asked).

## Sign in and approve

1. The client opens a YouThumb page in the browser; the user signs in (email/password or Google; magic links are not available for this connection).
2. The consent page shows the app, where it will send the user back, and the rights requested
   (`mcp:read`, `mcp:generate`, `mcp:assets`).
3. The user chooses the **Organization used by this app** — the app only works there, with its credits, persons
   and images. To change it later: revoke and reconnect.
4. **Allow** → the client gets a 15-minute access token and refreshes it on its own; no new sign-in at each use.

The user manages apps in **Account → MCP**: connected apps (revoke), each right family on/off (read / generation /
images and persons — re-checked on every call), and a daily credit limit for agents (organization credits; empty = no limit).

## Tools

| Right    | Tool                       | Input                                                               | Does                                                                |
| -------- | -------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------- |
| read     | `get_credits`              | —                                                                   | balance, plan and usage of the organization                         |
| read     | `list_persons`             | —                                                                   | persons with their photos (`id`, `name`, `description`, `images[]`) |
| read     | `get_person`               | `personId`                                                          | one person with its photos                                          |
| read     | `list_assets`              | `type?` (face, template, bucket), `page?`, `limit?` ≤ 100           | **published** images of the organization (see notes)                |
| read     | `list_templates`           | `category?`, `isFeatured?`, `isPublished?`, `page?`, `limit?` ≤ 100 | published templates (source images)                                 |
| read     | `list_presets`             | —                                                                   | the 15 style presets (`key`, `label`, `description`)                |
| read     | `get_generation`           | `projectId`                                                         | project status and every generation with its result image URLs      |
| generate | `create_thumbnail_project` | same body as `POST /api/thumbnails` (`references/api.md`)           | create the project (free) — **once per thumbnail**                  |
| generate | `start_generation`         | `projectId`, `prompt?`, `advancedOptions?`                          | start a generation (1 credit per variation)                         |
| assets   | `upload_asset`             | `base64`, `type`, `filename?`, `mimeType?`, `description?`          | upload an image                                                     |
| assets   | `delete_asset`             | `assetId`                                                           | delete an uploaded image (destructive, irreversible)                |
| assets   | `create_person`            | `name`, `description?`, `imageIds?`                                 | create a person from face images                                    |
| assets   | `add_person_images`        | `personId`, `imageIds` (1–10)                                       | attach more face images to a person                                 |

All tools are scoped to the organization chosen at consent: an id from another organization is answered "not found".
Read tools are annotated read-only; `delete_asset` is the only destructive one; `start_generation` is the only one
that spends credits.

## Responses

- Each tool returns JSON as text, followed by a last line: `YouThumb organization: <name> (<slug>)`.
  The slug gives the project URL: `https://www.youthumb.ai/<locale>/team/<slug>/thumbnails/<projectId>`.
- `create_thumbnail_project` → `{projectId, status: "pending", projectName, createdAt}`.
- `start_generation` → `{jobId, projectId, status: "processing", creditsConsumed}`.
- `get_generation` →

```json
{
  "projectId": "…",
  "projectName": "…",
  "status": "processing",
  "activeJobs": 1,
  "jobs": [
    {
      "jobId": "…",
      "status": "completed",
      "type": "magic-thumbnail",
      "userPrompt": "…",
      "generatedFileUrl": "https://…",
      "errorMessage": null,
      "createdAt": "…",
      "results": [
        {"id": "…", "url": "https://…", "width": 1920, "height": 1080}
      ]
    }
  ]
}
```

Project `status`: `pending` (no generation yet), `processing` (a job is pending/processing), `completed` (every job
completed), `failed` (at least one job failed and none is running). Job `status`: `pending`, `processing`,
`completed`, `failed`. Job `type`: `magic-thumbnail` (no source image) or `thumbnail` (adapting a source image).
Judge an attempt by **its own job**, not by the project status.

## Errors (returned as readable text, `isError: true`)

| Message says                                                              | Do                                                                        |
| ------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Not enough credits … Nothing was generated                                | tell the user                                                             |
| More variations than this organization's plan allows                      | retry with fewer `variations`                                             |
| Too many generations are running                                          | wait for one to finish (poll `get_generation`), then retry once           |
| Read / Generation / Images and persons is turned off … (Account → MCP)    | ask the user to turn it back on; don't retry                              |
| Today's agent spending limit is reached: N credit(s) left today           | fewer variations, wait until tomorrow, or ask the user to raise the limit |
| This connection has no YouThumb organization / user is no longer a member | ask the user to reconnect and choose an organization                      |
| `Forbidden: …` or `… not found in the YouThumb organization …`            | wrong id or wrong organization — list again                               |
| A missing right (scope not granted)                                       | the client asks the user to re-authorize; don't loop                      |

## Notes

- `upload_asset` types: `face` (person photos — returns an `id` for `create_person` / `add_person_images`),
  `template` (stored source image, has an `id`), `bucket` (temporary content image: logo, screenshot, product —
  **`id` is empty**, use the returned `url` in `contentImages`). `base64` may include a `data:image/...;base64,` prefix;
  `mimeType` defaults to `image/jpeg` — set it to match the file.
- `list_assets` only returns images marked as published in the organization. Images just uploaded through MCP/API
  are not published, so they do not show up: keep the `id`/`url` from the upload response. `bucket` uploads are never listed.
- `start_generation` reserves the daily agent limit for the requested variations and releases it if the start fails.

## Typical session

```
get_credits
list_persons                         → pick the person id (or upload_asset face + create_person)
upload_asset {type:"bucket", …}      → keep the url of each logo/screenshot
create_thumbnail_project {prompt:"… @image1 (logo) …", personId, presetKey:"free",
                          contentImages:[{url}], advancedOptions:{variations:1}}
start_generation {projectId}
get_generation {projectId}           → poll every 10–20 s until the new job is completed / failed
start_generation {projectId, prompt:"…full adjusted prompt…"}   ← same project for every new attempt
```

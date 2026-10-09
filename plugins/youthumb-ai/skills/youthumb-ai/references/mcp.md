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

| Right    | Tool                       | Input                                                                              | Does                                                                |
| -------- | -------------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| read     | `get_credits`              | —                                                                                  | balance, plan and usage of the organization                         |
| read     | `list_persons`             | —                                                                                  | persons with their photos (`id`, `name`, `description`, `images[]`) |
| read     | `get_person`               | `personId`                                                                         | one person with its photos                                          |
| read     | `list_assets`              | `type?`, `search?`, `isPublished?`, `page?`, `limit?` ≤ 100                        | owned/shared images, including unpublished uploads                  |
| read     | `list_templates`           | `scope?`, `search?`, `category?`, `isFeatured?`, `isPublished?`, `page?`, `limit?` | published templates (source images)                                 |
| read     | `list_presets`             | —                                                                                  | the 15 style presets (`key`, `label`, `description`)                |
| read     | `get_generation`           | `projectId`                                                                        | project status and every generation with its result image URLs      |
| generate | `create_thumbnail_project` | same body as `POST /api/thumbnails` (`references/api.md`)                          | create the project (free) — **once per thumbnail**                  |
| generate | `start_generation`         | `projectId` and per-attempt overrides (below)                                      | start a generation (1 credit per variation)                         |
| assets   | `upload_asset`             | `base64`, `type`, `filename?`, `mimeType?`, `description?`                         | upload an image                                                     |
| assets   | `delete_asset`             | `assetId`                                                                          | delete an uploaded image (destructive, irreversible)                |
| assets   | `create_person`            | `name`, `description?`, `imageIds?`                                                | create a person from face images                                    |
| assets   | `add_person_images`        | `personId`, `imageIds` (1–10)                                                      | attach more face images to a person                                 |

Organization resources stay in the organization chosen at consent. Temporary files belong to the caller;
the template catalog is public or caller-owned. Read tools are annotated read-only; delete/trash tools
are destructive. Generation/retry spends one credit per variation; prompt assistance spends 0.5 credit.
Management writes spend no credits.

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
`completed`, `failed`. Job `type`: `magic-thumbnail` (no source image) , `thumbnail` (adapting a source image) or `iterate` (a previous result).
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
- `list_assets` includes authorized unpublished imports. `type=bucket` lists personal temporary uploads;
  use their `url`, not their empty `id`. `fileKey` identifies the storage file without inventing a UUID.
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

## Library, favorites and trash

Discover existing projects before creating another thumbnail. `GET /api/thumbnails` and `list_thumbnail_projects` accept `search`, `status`, `favorite` and `deleted`. Lists use `page` (starting at 1) and `limit` (1–100, default 20); filters apply before pagination. `favorite=false` selects non-favorites and `deleted=true` selects trashed projects.

Favorites are shared by the selected organization. Send `{"isFavorite":true}` or `{"isFavorite":false}` explicitly; retries do not toggle the state. Rename with `{"projectName":"New title"}`. Library management costs no credits. MCP reads require `mcp:read` and Read access; mutations require `mcp:generate` and Generation access, plus the account's normal permissions. Trashing an entire project is restricted to administrators.

Normal lists and detail hide deleted jobs and deleted parent projects. `list_generations` returns completed results across visible thumbnail projects, including every variation image in each job’s `results` array. Trash listing accepts `kind=projects` or `kind=jobs` (individually deleted jobs, including those under deleted parents). There is no permanent deletion. Restore the project first, then any individually deleted jobs. Project restoration sets its status to `completed` and does not restore its children. Project `deletedAt` is derived from `updatedAt`; it is not an immutable historical deletion timestamp. A job has its own stored `deletedAt`. Repeated trash/restore requests preserve timestamps when already in the requested state.

Active jobs and projects with active jobs cannot be trashed (409). Restoring a job under a deleted parent also returns 409. Invalid payloads return 400, insufficient permissions 403, and unknown or foreign-organization identifiers 404.

`GET /api/thumbnails/:projectId`, `/detailed-status`, `/status` and `get_generation` share details: project metadata, `isFavorite`, full-project summary, latest result, job metadata, grouped images and person IDs. `jobs[].results` and top-level `activeJobs` remain available. Without query options all visible jobs are returned; `page` or `limit` opts into bounded job pages. A comma-separated `status` alone filters all visible jobs without pagination, preserving existing clients. Summary and latest result always describe the whole visible project.

| REST                                       | MCP                         | Parameters                                       |
| ------------------------------------------ | --------------------------- | ------------------------------------------------ |
| `GET /api/thumbnails`                      | `list_thumbnail_projects`   | `page, limit, search, status, favorite, deleted` |
| `GET /api/thumbnails/:projectId`           | `get_generation`            | `page?, limit?, status?`                         |
| `PATCH /api/thumbnails/:projectId`         | `update_thumbnail_project`  | `projectName`                                    |
| `GET /api/thumbnails/results`              | `list_generations`          | `page, limit, favorite, projectId`               |
| `PUT /api/thumbnails/:projectId/favorite`  | `set_project_favorite`      | `isFavorite`                                     |
| `PUT /api/thumbnails/jobs/:jobId/favorite` | `set_generation_favorite`   | `isFavorite`                                     |
| `DELETE /api/thumbnails/:projectId`        | `trash_thumbnail_project`   | `projectId`                                      |
| `POST /api/thumbnails/:projectId/restore`  | `restore_thumbnail_project` | `projectId`                                      |
| `DELETE /api/thumbnails/jobs/:jobId`       | `trash_generation`          | `jobId`                                          |
| `POST /api/thumbnails/jobs/:jobId/restore` | `restore_generation`        | `jobId`                                          |
| `GET /api/thumbnails/trash`                | `list_trash`                | `kind=projects / jobs, page, limit`              |

## Attempts, iteration and retry

`POST /api/thumbnails/:projectId/start` and `start_generation` keep the same project. Optional overrides: `prompt`, `personId` (`null` clears), `presetKey`, `templateId`, `styleReferenceUrl`, `youtubeUrl`, `sourceResultId`, `contentImages` (`[]` clears), `title` (`""` clears), and `advancedOptions`. Omitted fields retain project defaults; options merge. Overrides affect only this attempt, never the saved project. Supply only one source selector (preset, template, URL, YouTube or result). Each content image contains exactly one `userImageId` or `url`; explicit image URLs must belong to your configured uploaded storage path. Upload external images first.

Read `jobs[].results[].id` from project detail; pass that ID as `sourceResultId` to iterate on that exact completed image. It must belong to a visible completed job in the same project. A reference selects normal generation; a preset selects magic generation. Deleted projects and results cannot be generated from.

```json
{
  "projectId": "<project-id>",
  "sourceResultId": "<result-image-id>",
  "prompt": "Keep this composition and brighten the background",
  "advancedOptions": {"variations": 1}
}
```

`POST /api/thumbnails/jobs/:jobId/retry` / `retry_generation({jobId})` accepts a failed Default job only. It creates a new billed attempt in the same project, preserving the failed prompt, input images, persons, title, preset and options; it never resets the failed job or uses changed project defaults. The new metadata records `retryOfJobId`. Every call is non-idempotent and consumes generation credits again. Success includes `jobId`, `projectId`, `status`, `creditsConsumed` only after launch acknowledgement. A synchronous launch failure refunds credits and fails the orphan job.

## Prompt assistance

`POST /api/thumbnails/prompts/improve` / `improve_prompt`: `prompt` plus optional `context` with `axes` (`composition`, `hook`, `readability`), `language` (`fr`, `en`, `es`), `personName`, `title`, `customInstruction`, `hasStyleReference`, `referenceTokens` (`@image1`...), `sketchTokens` (`@sketch1`...).

`POST /api/thumbnails/prompts/from-reference` / `prompt_from_reference`: exactly one of `assetId`, published or own-private `templateId`, `projectId` + `sourceResultId`, or `imageUrl` proven to be an upload owned by the caller; optional `language` (default `en`). Arbitrary URLs are refused; upload first. References are authorized in the selected organization before billing.

Both return `{prompt, creditsConsumed: 0.5}` without creating a project or image. Provider failure refunds once. MCP requires `mcp:generate` and Generation enabled. Its daily cap counts exact half credits; changing the tool does not evade the limit. Existing integer limits retain their meaning.

## Resources: persons, uploads and catalog

All resource operations use the connected identity. Stored assets must belong to the selected organization and be owned by you or shared with it. Imports stay unpublished and are still discoverable. Unknown or inaccessible IDs return 404; permission refusals return 403; invalid inputs return 400. MCP errors use `isError: true`. Management operations consume no credits.

| REST                                      | MCP                   | Input / effect                                                                           |
| ----------------------------------------- | --------------------- | ---------------------------------------------------------------------------------------- |
| `PATCH /api/persons/:id`                  | `update_person`       | `name?` (1–100), `description?` (max 500); `null` clears description; at least one field |
| `DELETE /api/persons/:id`                 | `delete_person`       | Deletes profile and photo associations; stored images remain                             |
| `DELETE /api/persons/:id/images/:imageId` | `remove_person_image` | Detaches only this existing association, without deleting the asset                      |
| `PUT /api/persons/:id/default`            | `set_default_person`  | Idempotent organization default; existing owner permissions apply                        |
| `GET /api/assets/:id`                     | `get_asset`           | Authorized stored asset UUID                                                             |
| `GET /api/templates/:id`                  | `get_template`        | Published catalog template or your own private template                                  |

MCP mutations require `mcp:assets` and active Images and persons access. Reads require `mcp:read` and active Read access. Tool arguments use `personId`, `imageId`, `assetId` or `templateId` instead of URL path parameters. Person reads/updates return the profile with `images` and `imageCount`; delete returns `{"deleted":true}`. Existing create/add-photo calls also validate every referenced image before writing.

### Asset discovery

`GET /api/assets` / `list_assets` accepts `type` (`face`, `template`, `bucket`), `search`, `isPublished`, `page` (default 1), `limit` (default 20, max 100). Without `type`, stored face and template images are listed. Search matches name/description, and visibility/publication filters apply before pagination and count. Stored results include `isPublished`, `ownerScope: "organization"`; pagination returns `page`, `limit`, `total`, `totalPages`, `hasMore`.

`type=bucket` lists the caller's personal temporary folder, across all organizations. It is explicitly **user scoped**, with `ownerScope: "user"`, `id: ""`, stable `fileKey` and `url`. Use the URL in generation; a file key is not an asset UUID. Storage pages are reachable through `page`/`limit`; `total` and `totalPages` are `null`, so follow `hasMore`. Non-image placeholders/folders are omitted and may shorten a page. `search` matches file names; `isPublished` is invalid for temporary files. Provider failures are errors, not empty successful lists. A temporary file’s `createdAt` is its storage timestamp, or `null` if unavailable or invalid; no creation date is invented. Temporary files cannot be deleted through `/api/assets/:id`.

### Three reference namespaces

`GET /api/templates` / `list_templates` searches catalog title, description and tags with `search`. `scope=public` (default) lists published catalog templates; `scope=mine` lists only templates you created, published or private. `isPublished=false` requires `scope=mine`. Existing `category`, `isFeatured`, `page`, `limit` filters remain. Responses contain `templates` and `pagination` (`page`, `limit`, `total`, `totalPages`).

A catalog UUID goes in `templateId`. An uploaded personal reference is found through `list_assets(type="template")`: use its URL as `styleReferenceUrl`, or its UUID as a content `userImageId` / prompt-assistance `assetId`. A temporary file uses its URL. Never interchange catalog IDs, asset IDs and file keys.

```bash
curl "$BASE/api/assets?type=template&search=studio&page=1&limit=20" -H "x-api-key: $YOUTHUMB_API_KEY"
curl "$BASE/api/templates?scope=mine&isPublished=false&search=studio" -H "x-api-key: $YOUTHUMB_API_KEY"
curl -X PATCH "$BASE/api/persons/$PERSON_ID" -H "x-api-key: $YOUTHUMB_API_KEY" -H "Content-Type: application/json" -d '{"description":null}'
```

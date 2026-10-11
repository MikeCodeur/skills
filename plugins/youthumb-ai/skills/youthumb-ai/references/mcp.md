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

| Right    | Tool                       | Input                                                                                                                        | Does                                                                |
| -------- | -------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| read     | `get_credits`              | —                                                                                                                            | balance, plan and usage of the organization                         |
| read     | `list_persons`             | —                                                                                                                            | persons with their photos (`id`, `name`, `description`, `images[]`) |
| read     | `get_person`               | `personId`                                                                                                                   | one person with its photos                                          |
| assets   | `index_bucket_assets`      | `page?`, `limit?` (1–100, default 100)                                                                                       | Persist personal bucket UUIDs, explicit/idempotent, 0 credits       |
| read     | `list_assets`              | `type?`, `search?`, `isPublished?`, `page?`, `limit?` ≤ 100; bucket: `favorite?`, `sortBy?`, `sortOrder?`; face: `attached?` | owned/shared images, including unpublished uploads                  |
| read     | `list_templates`           | `scope?`, `search?`, `category?`, `isFeatured?`, `isPublished?`, `page?`, `limit?`                                           | published templates (source images)                                 |
| read     | `list_presets`             | —                                                                                                                            | the 15 style presets (`key`, `label`, `description`)                |
| read     | `get_generation`           | `projectId, summary?, page?, limit?, status?`                                                                                | project status and every generation with its result image URLs      |
| generate | `create_thumbnail_project` | same body as `POST /api/thumbnails` (`references/api.md`)                                                                    | create the project (free) — **once per thumbnail**                  |
| generate | `start_generation`         | `projectId` and per-attempt overrides (below)                                                                                | start a generation (1 credit per variation)                         |
| assets   | `upload_asset`             | `base64`, `type`, `filename?`, `mimeType?`, `description?`                                                                   | upload an image                                                     |
| assets   | `delete_asset`             | `assetId`                                                                                                                    | delete an uploaded image (destructive, irreversible)                |
| assets   | `create_person`            | `name`, `description?`, `imageIds?`                                                                                          | create a person from face images                                    |
| assets   | `add_person_images`        | `personId`, `imageIds` (1–10)                                                                                                | attach more face images to a person                                 |

Organization resources stay in the organization chosen at consent. Temporary files belong to the caller;
the template catalog is public or caller-owned. Read tools are annotated read-only; delete/trash tools
are destructive. Generation/retry spends one credit per variation; prompt assistance spends 0.5 credit.
Management writes spend no credits.

## Responses

- Each tool returns JSON as text, followed by a last line: `YouThumb organization: <name> (<slug>)`.
  The slug gives the project URL: `https://www.youthumb.ai/<locale>/team/<slug>/thumbnails/<projectId>`.
- `create_thumbnail_project` → `{projectId, status: "pending", projectName, createdAt}`.
- `start_generation` → `{jobId, projectId, status: "processing", creditsConsumed}`.
- `get_generation` in detailed mode (`summary:false`; pass `page`/`limit` to bound jobs) →

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
  **persisted UUID in `id`**, usable as `userImageId` or use the returned `url`). `base64` may include a `data:image/...;base64,` prefix;
  `mimeType` defaults to `image/jpeg` — set it to match the file.
- `list_assets` includes authorized unpublished imports. `type=bucket` lists personal temporary uploads;
  use their persisted UUID or URL; index historical files whose `id` is null. Find favorite logos with `favorite:true` and `search:"logo"`.
  See the assets section below for sorting, exact totals and full-enumeration cost.
- `start_generation` reserves the daily agent limit for the requested variations and releases it if the start fails.

## Typical session

```
get_credits
list_persons                         → pick the person id (or upload_asset face + create_person)
upload_asset {type:"bucket", …}      → keep the url of each logo/screenshot
create_thumbnail_project {prompt:"… @image1 (logo) …", personId, presetKey:"free",
                          contentImages:[{url}], advancedOptions:{variations:1}}
start_generation {projectId}
get_generation {projectId, summary:true, page:1, limit:5} → poll every 10–20 s until the new job is completed / failed
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
| `GET /api/thumbnails/:projectId`           | `get_generation`            | `summary?, page?, limit?, status?`               |
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

`POST /api/thumbnails/prompts/from-reference` / `prompt_from_reference`: exactly one of `assetId`, published or own-private `templateId`, `projectId` + `sourceResultId`, or `imageUrl` proven to be an upload owned by the caller or an exact stored result URL from a completed, visible job in the selected organization; optional `language` (default `en`). Arbitrary URLs are refused; upload first. References are authorized in the selected organization before billing.

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

`type=bucket` lists personal files across the caller's organizations, with `ownerScope: "user"`, `fileKey`, `url`, `isFavorite`, and persisted `id` (UUID) or `id: null` / `requiresIndexing: true` for historical files. See the indexing procedure below. `favorite=true` selects starred files, `favorite=false` excludes them; omission includes both. `search` matches file names. Bucket-only `sortBy=name|createdAt` and `sortOrder=asc|desc` default to `createdAt` descending. Unknown or invalid dates remain `createdAt: null` and sort last in either direction; ties use `fileKey` ascending. These three bucket filters are rejected for other asset types; `isPublished` is invalid for bucket files.

The service enumerates matching storage files in batches of 100, deduplicates by path and excludes non-images/placeholders before favorite filtering, sorting and public pagination. `total` is the filtered image count, `totalPages = ceil(total / limit)` and `hasMore` signals another page. Empty results have totals of 0; an out-of-range page is empty. Orphan favorites do not create assets. Every request enumerates the matching folder, so large folders cost more storage requests; concurrent changes are not an atomic snapshot. Any provider page failure fails the request rather than returning partial success.

Example: `GET /api/assets?type=bucket&search=logo&favorite=true&sortBy=createdAt&sortOrder=desc&page=1&limit=20` or MCP `list_assets({"type":"bucket","search":"logo","favorite":true,"sortBy":"createdAt","sortOrder":"desc","page":1,"limit":20})`.

### Three reference namespaces

`GET /api/templates` / `list_templates` searches catalog title, description and tags with `search`. `scope=public` (default) lists published catalog templates; `scope=mine` lists only templates you created, published or private. `isPublished=false` requires `scope=mine`. Existing `category`, `isFeatured`, `page`, `limit` filters remain. Responses contain `templates` and `pagination` (`page`, `limit`, `total`, `totalPages`).

A catalog UUID goes in `templateId`. An uploaded personal reference is found through `list_assets(type="template")`: use its URL as `styleReferenceUrl`, or its UUID as a content `userImageId` / prompt-assistance `assetId`. An indexed bucket file uses its UUID or URL; an unindexed historical file still uses its URL. Never interchange catalog IDs, asset IDs and file keys.

```bash
curl "$BASE/api/assets?type=template&search=studio&page=1&limit=20" -H "x-api-key: $YOUTHUMB_API_KEY"
curl "$BASE/api/templates?scope=mine&isPublished=false&search=studio" -H "x-api-key: $YOUTHUMB_API_KEY"
curl -X PATCH "$BASE/api/persons/$PERSON_ID" -H "x-api-key: $YOUTHUMB_API_KEY" -H "Content-Type: application/json" -d '{"description":null}'
```

## Compact exploration and explicit counts

Prefer `get_generation({projectId, summary:true, page:1, limit:5})` or `GET /api/thumbnails/:projectId?summary=true&page=1&limit=5` for exploration and polling. `summary=true` defaults to page 1 and limit 20 (1–100); `status` still accepts comma-separated statuses. The response contains `projectId`, `projectName`, `status`, `isFavorite`, the full-project `summary`, `pagination`, and `jobs` with only `jobId`, `status`, `type`, `isFavorite`, `userPrompt`, `generatedFileUrl`, `createdAt`. It omits project/job metadata, embedded images and persons. Full prompts are retained, so there is no fixed byte-size guarantee. Follow `pagination.hasMore` for further pages. For result image IDs needed by `sourceResultId`, request a bounded detailed page with `summary:false`.

`summary` absent or false preserves the existing detailed contract, including unbounded status-only reads. `list_generations` / `GET /api/thumbnails/results` now includes the complete `userPrompt` (or null) per job; no detail call is required to read prompts.

`get_credits` / `GET /api/usage` adds `counts.activeGenerationProjects` (non-deleted projects, all generation types) and `counts.completedThumbnailJobs` (completed, non-deleted jobs in visible thumbnail projects; same scope as unfiltered `list_generations`). These are organization-scoped counts of projects and jobs, not images or credits consumed. Existing `generations` statistics still count projects; `generations.total` equals `counts.activeGenerationProjects`. Credits and historical counters are unchanged.

Project favorites (`generation_project`), job favorites (`generation_job`) and personal bucket file favorites are separate. `list_thumbnail_projects({favorite:true})` may correctly return zero even with many favorite jobs: it does not select projects merely containing favorite jobs. If tool parameters omit `summary`, `page`, `limit` or asset `search`, refresh/reconnect the MCP client and reload its tool list; updating the skill alone does not update cached tool schemas.

### Prompt fidelity and reference resolution

`improve_prompt` / `POST /api/thumbnails/prompts/improve` preserves `context.personName` as a name, never an image token. Face references do not consume `@imageN`: those tokens identify content images. Provide `referenceTokens` / `sketchTokens` for all available references; existing tokens in the prompt and custom instruction are also checked. Every required token must occur exactly once and no new token may appear.

Put exact on-image text in quotation marks, for example `Alex beside @image1 with the text "BYE !"`, or set `context.title`. The output is validated for exact person name, title, and quoted literals in `prompt` / `customInstruction`. If the model drops a required literal or changes token counts, the call fails explicitly and refunds the 0.5 credit; it does not retry automatically or fabricate a replacement. This is a literal-preservation check, not a guarantee of artistic quality, semantic token placement or preservation of every unquoted instruction.

`prompt_from_reference` resolves a published or caller-owned private `templateId` using the stored image path, never the title. Relative paths become absolute storage URLs. `imageUrl` accepts the caller's own uploaded files, or an exact URL saved on a completed, non-trashed thumbnail job/result in the selected organization, under the configured storage origin/bucket/environment and `generation/` path. Another tenant's results, missing/deleted/failed results, arbitrary URLs, credentials, query strings and ambiguous paths are refused before billing/provider access. Use `projectId` + `sourceResultId` when selecting a result by ID. These two prompt tools consume 0.5 credit on success; they are not free reading tools.

A rejected improvement returns HTTP 500 with `code: "PROMPT_FIDELITY_FAILED"` and a safe explanation of the lost literal or invalid tokens. This is a provider-output failure, not invalid user input; the existing refund applies. Other unexpected server errors retain a generic response.

### Asset counts and person associations

Without `type`, `list_assets` / `GET /api/assets` lists accessible stored faces and templates (selected organization AND owned or shared). It does not include the personal bucket. `get_credits.resources.assets` counts all stored assets in the organization, including files not visible to this caller; it is not the count of personal bucket files and need not equal the accessible listing total. Query `type=bucket` separately.

Face assets return `personIds` and `isAttached`. Use `type=face&attached=true` (or MCP `{type:"face",attached:true}`) for attached photos, and `attached=false` for independent files. This filter applies before count and pagination and is rejected without `type=face`. A file may belong to several persons. Removing a photo from a person detaches it and preserves the asset. Multiple files may share a name or even identical bytes: neither a name nor an absent association proves that deletion is safe. No automatic cleanup or deduplication of existing assets is performed.

### Template URLs and continuation

`get_template` and `list_templates` return absolute `imageUrl` and nullable `previewUrl`, alongside unchanged `imagePath` / `previewPath`. Use the returned URL, never construct it from `title`. `pagination.hasMore` is `page * limit < total` (false for empty, final or out-of-range pages). `prompt_from_reference({templateId})` resolves the stored path on the server.

New image uploads/imports, including public catalog originals and generated previews, use a safe filename slug and an extension matching decoded image bytes; the display name is retained. Existing objects are not renamed or moved, so old unusual filenames remain valid through their stored URLs. Do not append a guessed extension to an existing URL.

### Refresh an outdated tool schema

The server contract version is `2.3.0`. Check the server's `tools/list`: `list_assets` includes `search`, `favorite`, `sortBy`, `sortOrder`, and face-only `attached`; `get_generation` includes `summary`, `page`, `limit`, `status`. Prefer `{projectId,summary:true,page:1,limit:5}` for discovery. A skill update and a server version bump cannot force a third-party client to discard cached tool definitions. Refresh/reconnect the MCP connection or start a fresh client session if these arguments are missing, then inspect `tools/list` again. Do not conclude that a feature is absent from a stale client schema.

`improve_prompt` and `prompt_from_reference` are paid generation actions (0.5 credit each), not read-only audit calls. Use them only when prompt generation is requested.

## Persistent personal bucket assets (2.3.0)

New `bucket` uploads (web UI, multipart REST or MCP/base64) return a persisted UUID in `id`, plus `fileKey`, `url`, `ownerScope: "user"` and `requiresIndexing: false`. Use `get_asset({assetId: id})` or `contentImages: [{userImageId: id}]`; URL input still works. Only the exact owner can resolve/delete this personal asset, across their selected organizations. It cannot be attached as a person face. `delete_asset` / `DELETE /api/assets/:id` removes the storage object before its registry identity.

Historical files are not silently migrated: listing returns `id: null` and `requiresIndexing: true` until explicitly indexed. Their URLs and path-based favorites remain unchanged. `pagination.unindexedCount` counts unindexed files in the complete filtered bucket listing. Reads never create database rows.

To index an authorized user's existing files, call `index_bucket_assets({page:1,limit:100})` or `POST /api/assets/bucket/index` with JSON `{"page":1,"limit":100}`. `page` starts at 1, `limit` is 1–100 (default 100). Response: `page`, `limit`, `indexed`, `hasMore`, `nextPage`. Follow `nextPage` until `hasMore:false`, then re-list the bucket to retrieve UUIDs. Pages are raw storage pages, so skipped non-images can make `indexed` lower than `limit`; `indexed` includes already indexed files. Restarting or retrying keeps the same UUID. Storage changes during traversal may require another pass; no snapshot is promised.

This explicit, idempotent, non-destructive mutation requires the `assets` family and `mcp:assets` scope, consumes **0 credits**, and reads at most `limit + 1` storage entries per call. It creates only personal identity rows: no renaming, copying, automatic orphan deletion or favorite changes. There are **33 MCP tools**. Migration and deployment must precede use; these code changes do not assert that production's historical files have already been indexed.

Reference-image MIME is detected from the downloaded bytes, not the URL suffix. PNG/JPEG/WebP remain in their actual format; other decodable formats are converted to PNG before prompt assistance. Non-image data is rejected before the model call. Historical extensionless or misleading filenames do not require renaming or re-uploading stored objects.

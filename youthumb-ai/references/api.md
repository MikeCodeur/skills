# YouThumb REST API (API key)

Base URL: `https://www.youthumb.ai` · Header: `x-api-key: $YOUTHUMB_API_KEY` (`Authorization: Bearer <key>` also accepted)
Key: YouThumb → **Account → API Keys** → create (shown once). Store it as `YOUTHUMB_API_KEY`; never commit it or paste it in prompts.
Full docs: https://www.youthumb.ai/en/docs/developer-api

Successful resource responses are `{"success": true, "data": …}` (asset DELETE also retains top-level `message: "Asset deleted successfully"`); failures are `{"success": false, "error": "…", "details"?: {field: [messages]}}`.
API-key sessions have no selected organization: REST resolves the **first organization returned for the account**. Changing the web application's active organization does not retarget the key. Use MCP consent when an explicit organization choice is required.

## Endpoints

| Method       | Path                                                                       | Purpose                                                                                                                        |
| ------------ | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| GET          | `/api/usage`                                                               | credits, plan and usage                                                                                                        |
| GET · POST   | `/api/persons`                                                             | list persons (with photos) / create one (`name` 1–100, `description?` ≤ 500, `imageIds?`)                                      |
| GET          | `/api/persons/:id`                                                         | one person with photos                                                                                                         |
| POST         | `/api/persons/:id/images`                                                  | attach face images (`imageIds`: 1–10 asset ids of type `face`)                                                                 |
| GET          | `/api/assets?type=&page=&limit=`                                           | authorized published or unpublished assets (`type`: face, template, bucket; `limit` ≤ 100, default 20)                         |
| POST         | `/api/assets`                                                              | upload — JSON (`base64`, `type`, `filename?`, `mimeType?`, `description?` ≤ 500) or multipart (`file`, `type`, `description?`) |
| GET · DELETE | `/api/assets/:id`                                                          | get / delete an asset                                                                                                          |
| POST         | `/api/thumbnails`                                                          | **create a project** (free) → 201                                                                                              |
| POST         | `/api/thumbnails/:projectId/start`                                         | **start a generation** (1 credit per variation); per-attempt overrides (see Start)                                             |
| GET          | `/api/thumbnails/:projectId/status`                                        | light status; `?page=&limit=` (≤ 100) paginates, `?status=completed` or `pending,processing` filters jobs                      |
| GET          | `/api/thumbnails/:projectId/detailed-status`                               | every job with its images                                                                                                      |
| GET          | `/api/templates?category=&isFeatured=&page=&limit=` · `/api/templates/:id` | templates (published only by default)                                                                                          |
| GET          | `/api/presets`                                                             | the 15 style presets                                                                                                           |

## Project fields (`POST /api/thumbnails`)

| Field                     | Rules                                                                                                                                                                                                                                                                                            |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `prompt`                  | **required**, 3–2000 chars                                                                                                                                                                                                                                                                       |
| `presetKey`               | optional, default `free`; one of `free`, `mrbeast-viral`, `reaction-shocked`, `before-after`, `challenge-experiment`, `gaming-esports`, `tech-review`, `vlog-travel`, `tutorial-educational`, `cinematic-film`, `dramatic-intense`, `mystery-dark`, `luxury-premium`, `office`, `youtube-studio` |
| `personId` / `personName` | one or the other, never both (`personName` ≤ 100, exact name in the organization)                                                                                                                                                                                                                |
| `contentImages`           | max 6, each `{"url": "…"}` or `{"userImageId": "…"}`; reference them `@image1…@image6` in the prompt, in array order                                                                                                                                                                             |
| `styleReferenceUrl`       | source image to adapt (any reachable image URL)                                                                                                                                                                                                                                                  |
| `youtubeUrl`              | imports the video's thumbnail (`maxresdefault.jpg`) as source image                                                                                                                                                                                                                              |
| `templateId`              | uses the template's image as source image                                                                                                                                                                                                                                                        |
| `title`                   | words to render on the thumbnail, ≤ 200 chars                                                                                                                                                                                                                                                    |
| `projectName`             | ≤ 100 chars; default = `title`, else "API Thumbnail"                                                                                                                                                                                                                                             |
| `advancedOptions`         | see below — stored as the project's defaults                                                                                                                                                                                                                                                     |

Source image precedence: `styleReferenceUrl` > `youtubeUrl` > `templateId` (only one is used). With a source image the
job type is `thumbnail` (adapt the image); without, `magic-thumbnail` (new composition). External images
(source and content) are copied into YouThumb storage at creation; an unreachable or non-image URL is dropped silently.

`advancedOptions` (each one only adds an instruction when set):

| Option            | Values                                                                              | Effect                                                 |
| ----------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------ |
| `variations`      | 1–4 (default 1)                                                                     | images per generation, 1 credit each; the plan caps it |
| `faceExpression`  | `neutral`, `happy`, `surprised`, `excited`, `serious`, `confident`, `keep-original` | `neutral` and `keep-original` add nothing              |
| `clothingStyle`   | `casual`, `professional`, `sporty`, `elegant`, `streetwear`, `keep-original`        |                                                        |
| `faceEnhancement` | `subtle`, `normal`, `enhanced`, `keep-original`                                     | retouching strength                                    |
| `textPosition`    | `top`, `center`, `bottom`, `none`, `keep-original`                                  | where any text goes                                    |
| `negativePrompt`  | ≤ 500 chars                                                                         | sent as "Avoid: …"                                     |
| `backgroundBlur`  | 0–100                                                                               | blur percentage                                        |

## Start (`POST /api/thumbnails/:projectId/start`)

Body (all optional): `prompt`, `personId`, `presetKey`, `templateId`, `styleReferenceUrl`, `youtubeUrl`, `sourceResultId`, `contentImages`, `title`, `advancedOptions` → `200 {jobId, projectId, status: "processing", creditsConsumed}`.
The override applies to **this generation only**: without `prompt`, the project's original prompt is used;
`advancedOptions` are merged onto the project's original ones. Omitted fields come from the project; supplied fields override this attempt (clearing rules below).

## Statuses and responses

- Create → `201 {projectId, status: "pending", projectName, createdAt}`.
- Project `status`: `pending` (no generation yet) · `processing` (a job pending/processing) · `completed` (every job
  completed) · `failed` (at least one job failed, none running). One failed attempt keeps the project `failed`: read the newest job.
- Job `status`: `pending` · `processing` · `completed` · `failed`. Job `type`: `magic-thumbnail`, `thumbnail` or `iterate`.
- `status` → `{projectId, status, projectName, jobs: [{jobId, status, generatedFileUrl, createdAt}], pagination?}`.
- `detailed-status` →

```json
{
  "projectId": "…", "projectName": "…", "status": "completed",
  "summary": {"totalJobs": 2, "activeJobs": 0, "completedJobs": 2, "failedJobs": 0, "totalVariations": 3},
  "latestGeneratedFileUrl": "https://…",
  "jobs": [{
    "jobId": "…", "status": "completed", "type": "magic-thumbnail", "userPrompt": "…",
    "generatedFileUrl": "https://…", "errorMessage": null, "createdAt": "…", "updatedAt": "…", "metadata": {…},
    "images": {
      "styleReference": [{"id": "…", "url": "…", "name": "…"}],
      "content": [{"id": "…", "url": "…", "name": "@image1"}],
      "faces": [{"id": "…", "url": "…", "name": "…"}],
      "results": [{"id": "…", "url": "https://…", "name": "…", "width": 1920, "height": 1080}]
    },
    "personIds": ["…"]
  }],
  "projectMetadata": {"presetKey": "free", "personId": "…", "contentImages": [{"url": "…"}], "title": "…", "advancedOptions": {…}},
  "createdAt": "…", "updatedAt": "…"
}
```

Results of a generation = `jobs[i].images.results[]`. Before the first start, `projectMetadata` shows what the project
holds: check `contentImages` contains real URLs, not `{}`.

- Persons → `{id, name, description, isDefault, imageCount, createdAt, images?: [{id, url, name}]}` (create returns no `images`).
- Assets → `{id, type, url, name, description, mimeType, size, width, height, createdAt}`; list → `{assets, pagination: {page, limit, total, totalPages, hasMore}}`.
- Templates → `{id, title, description, category, imagePath, previewPath, aspectRatio, tags, isPublished, isFeatured, createdAt}`;
  `category`: `gaming`, `tech`, `lifestyle`, `business`, `education`, `entertainment`, `user_uploaded_youtube`, `other`.
- Presets → `[{key, label, emoji, description}]`.

## Assets

| `type`     | Use                                                            | Stored                                                                                                |
| ---------- | -------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `face`     | person photos → `imageIds` of a person                         | yes, has `id`                                                                                         |
| `template` | a reusable source image                                        | yes, has `id`                                                                                         |
| `bucket`   | content images (logo, screenshot, product) for `contentImages` | temporary file, **`id` is empty** — use `url`; listed with `type=bucket`; not deletable by asset UUID |

`GET /api/assets` lists both published and unpublished authorized images in the selected organization:
owned by the caller or shared with that organization. It never publishes an upload. Use `type=bucket`
for personal temporary files. Check every upload is a real image rather than HTML saved as `.png`.

## Recipes

```bash
H=(-H "x-api-key: $YOUTHUMB_API_KEY" -H "Content-Type: application/json")
B=https://www.youthumb.ai

# Person: upload a face image (type face), then create the person with its id and an appearance description
FACE=$(base64 < face.jpg | tr -d '\n')
curl -s -X POST $B/api/assets "${H[@]}" \
  -d "{\"base64\":\"$FACE\",\"type\":\"face\",\"filename\":\"face.jpg\",\"mimeType\":\"image/jpeg\"}"
curl -s -X POST $B/api/persons "${H[@]}" \
  -d '{"name":"Alex","description":"30-year-old creator, short dark hair, no beard","imageIds":["<face-asset-id>"]}'

# Content image (logo, screenshot): type bucket, keep the returned url (multipart works too)
curl -s -X POST $B/api/assets -H "x-api-key: $YOUTHUMB_API_KEY" -F "file=@logo.png;type=image/png" -F "type=bucket"

# Project (ONCE per thumbnail)
curl -s -X POST $B/api/thumbnails "${H[@]}" -d '{
  "prompt":"Alex on the left third, confident smirk. @image1 (app logo) floating right with a soft blue glow. Clean dark background, teal key light from upper left. No text.",
  "presetKey":"free","personId":"<person-id>","contentImages":[{"url":"<asset-url>"}],
  "advancedOptions":{"variations":1}}'

# Generation (and every later attempt, SAME projectId, full prompt each time)
curl -s -X POST $B/api/thumbnails/<projectId>/start "${H[@]}" \
  -d '{"prompt":"…full adjusted prompt…","advancedOptions":{"faceExpression":"serious","variations":2}}'

# Result (poll every 10–20 s)
curl -s $B/api/thumbnails/<projectId>/detailed-status -H "x-api-key: $YOUTHUMB_API_KEY"
```

## Errors

| Code | Meaning                                                                                     | Do                                               |
| ---- | ------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| 400  | validation error (`details` per field), or no active organization                           | fix the fields (limits above)                    |
| 401  | missing/invalid/revoked key — or the key's rate limit (100 requests/min) was hit            | check the key; after many calls, wait 60 s       |
| 402  | `Insufficient credits`                                                                      | tell the user                                    |
| 403  | `Variations limit exceeded for your plan`, or project of another organization               | fewer variations / check ids                     |
| 404  | project, person, asset or template not found                                                | check ids                                        |
| 429  | `Too many parallel jobs running`                                                            | wait for running jobs to finish, then retry once |
| 500  | server error — also some not-found cases (e.g. unknown `personId`/`personName` at creation) | read `error`; retry once later                   |

One request at a time; never retry in a loop.

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

`type=bucket` lists the caller's personal temporary folder, across all organizations. It is explicitly **user scoped**, with `ownerScope: "user"`, `id: ""`, stable `fileKey` and `url`. Use the URL in generation; a file key is not an asset UUID. Storage pages are reachable through `page`/`limit`; `total` and `totalPages` are `null`, so follow `hasMore`. Non-image placeholders/folders are omitted and may shorten a page. `search` matches file names; `isPublished` is invalid for temporary files. Provider failures are errors, not empty successful lists. Temporary files cannot be deleted through `/api/assets/:id`.

### Three reference namespaces

`GET /api/templates` / `list_templates` searches catalog title, description and tags with `search`. `scope=public` (default) lists published catalog templates; `scope=mine` lists only templates you created, published or private. `isPublished=false` requires `scope=mine`. Existing `category`, `isFeatured`, `page`, `limit` filters remain. Responses contain `templates` and `pagination` (`page`, `limit`, `total`, `totalPages`).

A catalog UUID goes in `templateId`. An uploaded personal reference is found through `list_assets(type="template")`: use its URL as `styleReferenceUrl`, or its UUID as a content `userImageId` / prompt-assistance `assetId`. A temporary file uses its URL. Never interchange catalog IDs, asset IDs and file keys.

```bash
curl "$BASE/api/assets?type=template&search=studio&page=1&limit=20" -H "x-api-key: $YOUTHUMB_API_KEY"
curl "$BASE/api/templates?scope=mine&isPublished=false&search=studio" -H "x-api-key: $YOUTHUMB_API_KEY"
curl -X PATCH "$BASE/api/persons/$PERSON_ID" -H "x-api-key: $YOUTHUMB_API_KEY" -H "Content-Type: application/json" -d '{"description":null}'
```

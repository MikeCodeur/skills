# YouThumb REST API (API key)

Base URL: `https://www.youthumb.ai` · Header: `x-api-key: $YOUTHUMB_API_KEY` (`Authorization: Bearer <key>` also accepted)
Key: YouThumb → **Account → API Keys** → create (shown once). Store it as `YOUTHUMB_API_KEY`; never commit it or paste it in prompts.
Full docs: https://www.youthumb.ai/en/docs/developer-api

Every response is `{"success": true, "data": …}` or `{"success": false, "error": "…", "details"?: {field: [messages]}}`.
The key acts in the account's **active organization** (switch it in the app before calling).

## Endpoints

| Method       | Path                                                                       | Purpose                                                                                                                        |
| ------------ | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| GET          | `/api/usage`                                                               | credits, plan and usage                                                                                                        |
| GET · POST   | `/api/persons`                                                             | list persons (with photos) / create one (`name` 1–100, `description?` ≤ 500, `imageIds?`)                                      |
| GET          | `/api/persons/:id`                                                         | one person with photos                                                                                                         |
| POST         | `/api/persons/:id/images`                                                  | attach face images (`imageIds`: 1–10 asset ids of type `face`)                                                                 |
| GET          | `/api/assets?type=&page=&limit=`                                           | published assets (`type`: face, template, bucket; `limit` ≤ 100, default 20)                                                   |
| POST         | `/api/assets`                                                              | upload — JSON (`base64`, `type`, `filename?`, `mimeType?`, `description?` ≤ 500) or multipart (`file`, `type`, `description?`) |
| GET · DELETE | `/api/assets/:id`                                                          | get / delete an asset                                                                                                          |
| POST         | `/api/thumbnails`                                                          | **create a project** (free) → 201                                                                                              |
| POST         | `/api/thumbnails/:projectId/start`                                         | **start a generation** (1 credit per variation); body optional `prompt`, `advancedOptions`                                     |
| GET          | `/api/thumbnails/:projectId/status`                                        | light status; `?page=&limit=` (≤ 100) paginates, `?status=completed` or `pending,processing` filters jobs                      |
| GET          | `/api/thumbnails/:projectId/detailed-status`                               | every job with its images                                                                                                      |
| GET          | `/api/templates?category=&isFeatured=&page=&limit=` · `/api/templates/:id` | templates (published only by default)                                                                                          |
| GET          | `/api/presets`                                                             | the 15 style presets                                                                                                           |

## Project fields (`POST /api/thumbnails`)

| Field                     | Rules                                                                                                                                                                                                                                                                                                                                                                                       |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `prompt`                  | **required**, 3–2000 chars                                                                                                                                                                                                                                                                                                                                                                  |
| `presetKey`               | optional, default `free`; one of `free`, `mrbeast-viral`, `reaction-shocked`, `before-after`, `challenge-experiment`, `gaming-esports`, `tech-review`, `vlog-travel`, `tutorial-educational`, `cinematic-film`, `dramatic-intense`, `mystery-dark`, `luxury-premium`, `office`, `youtube-studio` — named presets are stored but currently not applied to API generations (they run as Free) |
| `personId` / `personName` | one or the other, never both (`personName` ≤ 100, exact name in the organization)                                                                                                                                                                                                                                                                                                           |
| `contentImages`           | max 6, each `{"url": "…"}`; reference them `@image1…@image6` in the prompt, in array order. `{"userImageId"}` is accepted but ignored at generation                                                                                                                                                                                                                                         |
| `styleReferenceUrl`       | source image to adapt (any reachable image URL)                                                                                                                                                                                                                                                                                                                                             |
| `youtubeUrl`              | imports the video's thumbnail (`maxresdefault.jpg`) as source image                                                                                                                                                                                                                                                                                                                         |
| `templateId`              | uses the template's image as source image                                                                                                                                                                                                                                                                                                                                                   |
| `title`                   | words to render on the thumbnail, ≤ 200 chars                                                                                                                                                                                                                                                                                                                                               |
| `projectName`             | ≤ 100 chars; default = `title`, else "API Thumbnail"                                                                                                                                                                                                                                                                                                                                        |
| `advancedOptions`         | see below — stored as the project's defaults                                                                                                                                                                                                                                                                                                                                                |

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

Body (all optional): `{"prompt": "…", "advancedOptions": {…}}` → `200 {jobId, projectId, status: "processing", creditsConsumed}`.
The override applies to **this generation only**: without `prompt`, the project's original prompt is used;
`advancedOptions` are merged onto the project's original ones. Person, images, source, title and preset come from the project.

## Statuses and responses

- Create → `201 {projectId, status: "pending", projectName, createdAt}`.
- Project `status`: `pending` (no generation yet) · `processing` (a job pending/processing) · `completed` (every job
  completed) · `failed` (at least one job failed, none running). One failed attempt keeps the project `failed`: read the newest job.
- Job `status`: `pending` · `processing` · `completed` · `failed`. Job `type`: `magic-thumbnail` or `thumbnail`.
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
- Assets → `{id, type, url, name, description, mimeType, size, width, height, createdAt}`; list → `{assets, pagination: {page, limit}}`.
- Templates → `{id, title, description, category, imagePath, previewPath, aspectRatio, tags, isPublished, isFeatured, createdAt}`;
  `category`: `gaming`, `tech`, `lifestyle`, `business`, `education`, `entertainment`, `user_uploaded_youtube`, `other`.
- Presets → `[{key, label, emoji, description}]`.

## Assets

| `type`     | Use                                                            | Stored                                                                   |
| ---------- | -------------------------------------------------------------- | ------------------------------------------------------------------------ |
| `face`     | person photos → `imageIds` of a person                         | yes, has `id`                                                            |
| `template` | a reusable source image                                        | yes, has `id`                                                            |
| `bucket`   | content images (logo, screenshot, product) for `contentImages` | temporary file, **`id` is empty** — use `url`; not listed, not deletable |

`GET /api/assets` lists only published images: assets uploaded through the API are unpublished and do not appear —
keep the upload response. Check every file is a real image before uploading (`file logo.png`): an HTML page saved as `.png` fails.

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

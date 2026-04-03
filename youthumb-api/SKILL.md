---
name: youthumb-api
description: >
  Interact with the YouThumb.ai API to generate AI-powered YouTube thumbnails.
  Upload assets, create persons, create projects, launch generation, and retrieve results.
  Use when the user says "YouThumb API", "generate thumbnail via API",
  "upload asset to YouThumb", "create YouThumb project", "thumbnail generation",
  "YouThumb person", or when automating YouTube thumbnail creation programmatically.
license: MIT
compatibility: Requires curl or any HTTP client. API key from YouThumb.ai required.
metadata:
  author: mikecodeur
  version: "1.0"
  website: https://youthumb.ai
---

# YouThumb API — Thumbnail Generation Skill

Automate YouTube thumbnail creation via the [YouThumb.ai](https://youthumb.ai) developer API.

## Authentication

**Get your API key:** Log in to [YouThumb.ai](https://youthumb.ai) → User menu → Account → API Key.

Set it as an environment variable:

```bash
export YOUTHUMB_API_KEY="your-api-key-here"
```

All API calls require the header:
```
x-api-key: $YOUTHUMB_API_KEY
```

---

## API Reference

**Base URL:** `https://www.youthumb.ai`

### Endpoints Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| **Assets** | | |
| POST | `/api/assets` | Upload an image |
| GET | `/api/assets` | List all assets |
| GET | `/api/assets/:id` | Get asset details |
| DELETE | `/api/assets/:id` | Delete an asset |
| **Thumbnails** | | |
| POST | `/api/thumbnails` | Create a project |
| POST | `/api/thumbnails/:projectId/start` | Start generation (costs credits) |
| GET | `/api/thumbnails/:projectId/status` | Simple status check |
| GET | `/api/thumbnails/:projectId/detailed-status` | Status with image URLs |
| **Persons** | | |
| GET | `/api/persons` | List all persons |
| POST | `/api/persons` | Create a person |
| GET | `/api/persons/:id` | Get person details |
| POST | `/api/persons/:id/images` | Add face images to a person |
| **Other** | | |
| GET | `/api/templates` | List available templates |
| GET | `/api/presets` | List available presets |

**Full API docs:** https://www.youthumb.ai/en/docs/developer-api

---

## Workflow

### Step 1 — Set Up a Person (one-time)

A Person is a face identity used across thumbnails. Upload 3-5 clear photos of the subject.

```bash
# Create a person
curl -s -X POST "https://www.youthumb.ai/api/persons" \
  -H "x-api-key: $YOUTHUMB_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "John Creator"}'
```

Response:
```json
{"success": true, "data": {"id": "uuid-of-person", "name": "John Creator"}}
```

```bash
# Add face images (base64 encoded)
FACE_B64=$(base64 -w0 /path/to/face-photo.jpg)
curl -s -X POST "https://www.youthumb.ai/api/persons/PERSON_ID/images" \
  -H "x-api-key: $YOUTHUMB_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"base64\":\"$FACE_B64\",\"filename\":\"face1.jpg\",\"mimeType\":\"image/jpeg\"}"
```

Upload 3-5 images with different angles and lighting for best results.

### Step 2 — Upload Assets (logos, screenshots, icons)

Assets are visual elements that appear on the thumbnail (logos, screenshots, icons).

```bash
# Upload an image asset
ICON_B64=$(base64 -w0 /path/to/logo.png)
curl -s -X POST "https://www.youthumb.ai/api/assets" \
  -H "x-api-key: $YOUTHUMB_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"base64\":\"$ICON_B64\",\"type\":\"bucket\",\"filename\":\"logo.png\",\"mimeType\":\"image/png\"}"
```

Response:
```json
{"success": true, "data": {"id": "uuid", "url": "https://...supabase.co/..."}}
```

⚠️ **Always use `"type": "bucket"`** for content assets (logos, screenshots, etc.).

⚠️ **Verify the file is a real image** before uploading — check with `file /path/to/image.png`. If it returns "HTML document" or anything other than image data, do NOT upload.

### Step 3 — Create a Thumbnail Project

```bash
curl -s -X POST "https://www.youthumb.ai/api/thumbnails" \
  -H "x-api-key: $YOUTHUMB_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Person on the left with a confident smirk. @image1 (React logo) floating on the right with blue glow. Clean dark background with subtle teal lighting. No text.",
    "presetKey": "free",
    "projectName": "React Tutorial Thumb",
    "personId": "your-person-uuid",
    "contentImages": [
      {"url": "https://...your-uploaded-asset-url..."}
    ],
    "advancedOptions": {
      "variations": 1,
      "faceExpression": "confident",
      "textPosition": "none"
    }
  }'
```

Response:
```json
{"success": true, "data": {"id": "project-uuid", "status": "draft"}}
```

#### Asset References in Prompts (Critical)

Reference uploaded assets in your prompt using `@image1`, `@image2`, `@image3` etc. They map to the `contentImages` array in order.

```
@image1 → contentImages[0]
@image2 → contentImages[1]
@image3 → contentImages[2]
```

**Always include `@imageN (description)` in the prompt text.** Without this reference, the uploaded image will NOT appear on the thumbnail.

**Example:**
> Subject on the left, confident expression. @image1 (Claude Code logo) prominently displayed center-right with orange glow. @image2 (code screenshot) in the background with glass overlay. Dark cinematic atmosphere.

#### Content Images Format

```json
{"url": "https://..."}          // By direct URL (preferred)
{"userImageId": "asset-uuid"}   // By uploaded asset ID
```

Max 5 content images per project.

### Step 4 — Start Generation

⚠️ **This consumes credits.** Always verify the project looks correct before starting.

```bash
# Verify the project first
curl -s "https://www.youthumb.ai/api/thumbnails/$PROJECT_ID/detailed-status" \
  -H "x-api-key: $YOUTHUMB_API_KEY"
```

Check that `contentImages` contains the correct URLs (not empty objects `{}`).

```bash
# Start generation
curl -s -X POST "https://www.youthumb.ai/api/thumbnails/$PROJECT_ID/start" \
  -H "x-api-key: $YOUTHUMB_API_KEY"
```

The `start` endpoint optionally accepts `prompt` and `advancedOptions` to override the project config:

```bash
curl -s -X POST "https://www.youthumb.ai/api/thumbnails/$PROJECT_ID/start" \
  -H "x-api-key: $YOUTHUMB_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Alternative prompt for this generation run"}'
```

### Step 5 — Retrieve Results

Poll the status until generation completes:

```bash
curl -s "https://www.youthumb.ai/api/thumbnails/$PROJECT_ID/detailed-status" \
  -H "x-api-key: $YOUTHUMB_API_KEY"
```

When `status: "completed"`, download the results from `jobs[].images.results[]`.

---

## Field Validation Reference

### Thumbnail Project

| Field | Required | Rules |
|-------|----------|-------|
| `prompt` | ✅ Yes | 3-2000 characters |
| `presetKey` | ✅ Yes | Use `"free"` |
| `personId` | No | UUID — do not combine with `personName` |
| `personName` | No | Max 100 chars — do not combine with `personId` |
| `projectName` | No | Max 100 chars |
| `contentImages` | No | Max 5 items, each `{url?}` or `{userImageId?}` |
| `title` | No | Max 200 chars |
| `templateId` | No | UUID |
| `styleReferenceUrl` | No | Valid URL |
| `youtubeUrl` | No | Valid URL |

### Advanced Options

| Field | Values | Default |
|-------|--------|---------|
| `variations` | 1, 2, 3, or 4 | 1 |
| `faceExpression` | neutral, happy, surprised, excited, serious, confident, keep-original | confident |
| `textPosition` | top, center, bottom, none, keep-original | none |
| `negativePrompt` | Max 500 chars | — |
| `clothingStyle` | casual, professional, sporty, elegant, streetwear, keep-original | keep-original |
| `faceEnhancement` | subtle, normal, enhanced, keep-original | normal |
| `backgroundBlur` | 0-100 | 0 |

---

## Rate Limits

The API has **aggressive rate limiting**. Important:

- Wait **30-45 seconds** between calls when you hit a rate limit
- A `401 Unauthorized` response is **often a rate limit in disguise**, not a real auth error — wait and retry
- Do NOT spam retries — use exponential backoff
- One request at a time is safest

## Error Codes

| Code | Meaning | Action |
|------|---------|--------|
| 400 | Bad request | Check required fields and validation rules |
| 401 | Invalid/missing API key | Verify your key — or wait (could be rate limit) |
| 402 | Insufficient credits | Top up credits at youthumb.ai |
| 403 | Access denied | Check account permissions |
| 404 | Resource not found | Verify project/person/asset ID |
| 429 | Rate limited | Wait 30-45 seconds and retry |
| 500 | Server error | Retry after a delay |

---

## Best Practices

1. **Always verify before generating** — check `detailed-status` to confirm assets are correctly linked
2. **Start with 1 variation** — each variation costs credits, scale up only when needed
3. **Use `presetKey: "free"`** — gives full control over the prompt
4. **Validate images before upload** — verify files are actual images (PNG, JPEG), not HTML or SVG
5. **Reference every asset** — use `@image1 (description)` in prompts or the image won't appear
6. **Respect rate limits** — the API is strict, space your calls out

For prompt writing tips and examples, see the companion skill: [youthumb-prompts](../youthumb-prompts/).

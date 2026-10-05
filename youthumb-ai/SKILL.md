---
name: youthumb-ai
description: >
  Everything to create YouTube thumbnails with YouThumb.ai from an agent: connect through the MCP server
  (Claude, Claude Code, Codex, ChatGPT) or the REST API with an API key, set up persons and assets, write
  strong thumbnail prompts, generate and iterate inside ONE project per thumbnail. Use when the user says
  "YouThumb", "miniature YouTube", "thumbnail", "génère une miniature", "prompt de miniature", "YouThumb MCP",
  "YouThumb API", "create a thumbnail project", "relance une génération", or works on YouTube thumbnails.
license: MIT
compatibility: MCP client (Claude, Claude Code, Codex, ChatGPT) or any HTTP client with a YouThumb API key.
metadata:
  version: '2.0'
  website: https://www.youthumb.ai
  replaces: youthumb-api, youthumb-prompts
---

# YouThumb.ai — thumbnails from an agent

YouThumb.ai generates YouTube thumbnails from a prompt, a person (face identity), reference images and a style.
This skill covers the two ways in, the workflow, and how to write prompts that work.

## 1. Pick the connection

| You are in…                         | Use                              | How                 |
| ----------------------------------- | -------------------------------- | ------------------- |
| Claude, ChatGPT, Claude Code, Codex | **MCP server** (sign in, no key) | `references/mcp.md` |
| n8n, scripts, CI, any HTTP client   | **REST API** with an API key     | `references/api.md` |

If both are possible, prefer **MCP**: same capabilities, no key to store, the user approves rights and chooses the organization.

## 2. Golden rule — one project per thumbnail

A **project** (`thumbnails/<projectId>`) is ONE thumbnail being worked on. Every attempt, retry, variant or
tweak for that thumbnail is a **new generation inside the same project**, never a new project.

- **First time** for a video/thumbnail: create the project once (`create_thumbnail_project` / `POST /api/thumbnails`),
  then start a generation.
- **Every next attempt** on the same thumbnail: call `start_generation` / `POST /api/thumbnails/<projectId>/start`
  on the **same `projectId`**, optionally with a new `prompt` and `advancedOptions` (e.g. another expression).
- Want several options at once? Use `advancedOptions.variations` (1–4) in ONE generation — not one project per variation.
- Create a new project only when the user starts a **different thumbnail** (another video, or explicitly asks for a new one).
- Keep the `projectId` in the conversation (and tell the user its URL: `https://www.youthumb.ai/<locale>/team/<org>/thumbnails/<projectId>`
  when known) so later requests reuse it. If unsure which project the user means, list or ask — don't create a duplicate.

## 3. Workflow

1. **Check credits** (`get_credits` / `GET /api/usage`). Each variation costs 1 credit; creating a project is free.
2. **Person** — the face on the thumbnail. Reuse an existing one (`list_persons`); create one only if missing:
   upload 1–2 sharp, front-facing face photos (`type: "face"`), then create the person with their ids and a short
   appearance description (age, beard, glasses…). The generator uses the first two photos of a person.
3. **Assets** — logos, screenshots, products to place on the thumbnail. Upload real images only (PNG/JPEG/WebP/GIF),
   keep the returned **`url`**, pass them as `contentImages: [{"url": …}]` (max 6, in order) and reference each as
   `@image1`, `@image2`… in the prompt.
4. **Prompt** — write it with `references/prompts.md` (7 blocks: subject, expression, assets, background,
   lighting, materials, composition). Use `presetKey: "free"` and describe the style in the prompt
   (see pitfalls about named presets). To adapt an existing image, pass it as `styleReferenceUrl`,
   `youtubeUrl` or `templateId` and describe what changes and what stays.
5. **Show the user** the prompt and settings, then **create the project** (once) and **start a generation** with 1 variation.
6. **Poll** `get_generation` / `GET /api/thumbnails/<projectId>/detailed-status` every 10–20 s until the **newest job**
   is `completed` (images in its results) or `failed` (read `errorMessage`). Show the image URLs.
7. **Iterate in the same project** (rule 2): adjust the prompt or options, start a new generation.

## 4. Things that bite

- **`@imageN` is mandatory** for every content image, with a short description: `@image1 (product logo)`.
  A content image whose token is not in the prompt is dropped before generation.
- **Content images must be passed by `url`.** `{ "userImageId": … }` passes validation but is ignored at generation.
  Upload results of type `bucket` have an empty `id` — use their `url`. External URLs are downloaded at project
  creation; an unreachable or non-image URL is silently removed (check `projectMetadata.contentImages` in `detailed-status`).
- **Fixed at project creation**: person, content images, source image (`styleReferenceUrl` / `youtubeUrl` / `templateId`),
  title and preset. A later `start` only changes `prompt` and `advancedOptions`.
- **A `start` override is not saved.** A `start` without `prompt` reuses the project's ORIGINAL prompt, and
  `advancedOptions` merge onto the project's original ones — resend the full adjusted prompt every time.
- **Named presets**: `presetKey` accepts every key from `list_presets`, but in the current code generations started
  through the API/MCP run as **Free** (the preset is stored, not applied). Describe the look in the prompt; named
  presets work in the app.
- **Project status ≠ last attempt**: the project becomes `failed` as soon as any of its jobs failed, even if a later
  one completed. Read the status of the newest job.
- **Prompt limit: 2000 characters** (3 minimum). Aim for 80–250 words.
- **Generation is asynchronous**: `start` returns `processing`; failures show up later in the job's `errorMessage`.
- **Errors are explicit**: not enough credits, too many variations for the plan, too many generations running
  at once, a right turned off by the user (Account → MCP), the daily agent spending limit. Report them, don't loop.
- **Organization**: via MCP, the agent works in the organization the user chose when approving; every tool
  response ends with it. Via API key, it is the account's **active organization** (switched in the app).
- **Never** spam retries; on a rate limit wait 30–60 s and back off.
- **Privacy**: never put personal data (real names, e-mails, faces of people who did not consent) in prompts,
  examples or logs. Use the person registered in YouThumb by the user.
- **App only** (not reachable through API/MCP): sketch, Improve the prompt, prompt extraction from an image,
  Add Light, Add Title styling, Use a result as source, Reuse run, compare, favorites, trash, preview tools.
  Point the user to the app for these — see `references/prompts.md` § 12.

## 5. References

- `references/mcp.md` — connect from Claude, Claude Code, Codex, ChatGPT; tools, rights, responses.
- `references/api.md` — REST endpoints, fields, enums, statuses, response shapes, curl recipes, errors.
- `references/prompts.md` — how the generator reads a prompt, starting points, prompt method and libraries,
  options, iteration, what is app-only.
- `references/prompt-examples.md` — full example prompts by video type.

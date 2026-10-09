---
name: youthumb-ai
description: >
  Create and manage YouTube thumbnails with YouThumb.ai from an agent: connect through the MCP server
  (Claude, Claude Code, Codex, ChatGPT) or the REST API with an API key, set up persons and assets, write
  strong thumbnail prompts, generate and iterate inside ONE project per thumbnail. Use when the user says
  "YouThumb", "miniature YouTube", "thumbnail", "génère une miniature", "prompt de miniature", "YouThumb MCP",
  "YouThumb API", "create a thumbnail project", "relance une génération", or works on YouTube thumbnails.
license: MIT
compatibility: MCP client (Claude, Claude Code, Codex, ChatGPT) or any HTTP client with a YouThumb API key.
metadata:
  version: '2.2.1'
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

## Library management

Find existing projects with `list_thumbnail_projects` before creating duplicates. Use `favorite: true` for organization favorites and `list_generations` for completed results across projects. Set favorites explicitly; never simulate a toggle. Use `list_trash` to inspect removed items, restore the parent project before an individually deleted generation. Reads require Read access; these management writes require Generation access but consume no credits. Project trash remains admin-only. Contracts and examples: `references/api.md` and `references/mcp.md`.

## 3. Workflow

1. **Check credits** (`get_credits` / `GET /api/usage`). Each variation costs 1 credit; creating a project is free.
2. **Person** — the face on the thumbnail. Reuse an existing one (`list_persons`); create one only if missing:
   upload 1–2 sharp, front-facing face photos (`type: "face"`), then create the person with their ids and a short
   appearance description (age, beard, glasses…). The generator uses the first two photos of a person.
3. **Assets** — logos, screenshots, products to place on the thumbnail. Upload real images only (PNG/JPEG/WebP/GIF),
   keep the returned **`url`**, pass them as `contentImages: [{"url": …}]` (max 6, in order) and reference each as
   `@image1`, `@image2`… in the prompt.
4. **Prompt** — write it with `references/prompts.md` (7 blocks: subject, expression, assets, background,
   lighting, materials, composition). Use `presetKey: "free"` for full control, or a named
   preset from `list_presets` and keep the prompt on subject and story. To adapt an existing image, pass it as `styleReferenceUrl`,
   `youtubeUrl` or `templateId` and describe what changes and what stays.
5. **Show the user** the prompt and settings, then **create the project** (once) and **start a generation** with 1 variation.
6. **Poll** `get_generation` / `GET /api/thumbnails/<projectId>/detailed-status` every 10–20 s until the **newest job**
   is `completed` (images in its results) or `failed` (read `errorMessage`). Show the image URLs.
7. **Iterate in the same project** (rule 2): adjust the prompt or options, start a new generation.

## 4. Things that bite

- **`@imageN` is mandatory** for every content image, with a short description: `@image1 (product logo)`.
  A content image whose token is not in the prompt is dropped before generation.
- **Content images**: pass `{ "url": … }` or `{ "userImageId": … }` (an uploaded asset id).
  Upload results of type `bucket` have an empty `id` — use their `url`. External URLs are downloaded at project
  creation; an unreachable or non-image URL is silently removed (check `projectMetadata.contentImages` in `detailed-status`).
- **Per-attempt overrides**: change person, content images, source, title, preset, prompt and options on
  `start_generation` without making another project. Use `sourceResultId` to refine a completed result.
- **A `start` override is not saved.** A `start` without `prompt` reuses the project's ORIGINAL prompt, and
  `advancedOptions` merge onto the project's original ones — resend the full adjusted prompt every time.
- **Project status ≠ last attempt**: the project becomes `failed` as soon as any of its jobs failed, even if a later
  one completed. Read the status of the newest job.
- **Prompt limit: 2000 characters** (3 minimum). Aim for 80–250 words.
- **Generation is asynchronous**: `start` returns `processing`; failures show up later in the job's `errorMessage`.
- **Errors are explicit**: not enough credits, too many variations for the plan, too many generations running
  at once, a right turned off by the user (Account → MCP), the daily agent spending limit. Report them, don't loop.
- **Organization**: via MCP, the agent works in the organization the user chose when approving; every tool
  response ends with it. API-key sessions have no selected organization: REST uses the **first organization
  returned for the account**. Changing the web app's active organization does not retarget an API key;
  use MCP consent for an explicit organization choice.
- **Never** spam retries; on a rate limit wait 30–60 s and back off.
- **Privacy**: never put personal data (real names, e-mails, faces of people who did not consent) in prompts,
  examples or logs. Use the person registered in YouThumb by the user.
- **App only**: sketch creation, Add Light, Add Title styling, visual comparison and preview tools.
  Library favorites/trash, prompt assistance and result iteration are available through API/MCP.
- **Three image namespaces**: stored assets use `assetId`/`userImageId`; catalog templates use `templateId`;
  temporary bucket files have `id: ""`, `fileKey`, `ownerScope: "user"` and are referenced by URL.
  List assets to find unpublished uploads; use `scope: "mine"` for your private catalog templates.
- **Persons**: update name/description (`null` clears), detach a photo, delete a person or set the organization
  default with the resource tools. Deleting a person or detaching a photo keeps the underlying asset.

## 5. References

- `references/mcp.md` — connect from Claude, Claude Code, Codex, ChatGPT; tools, rights, responses.
- `references/api.md` — REST endpoints, fields, enums, statuses, response shapes, curl recipes, errors.
- `references/prompts.md` — how the generator reads a prompt, starting points, prompt method and libraries,
  options, iteration, what is app-only.
- `references/prompt-examples.md` — full example prompts by video type.

## Same-project attempts and prompt assistance

Use `start_generation` overrides for another attempt, `sourceResultId` from `get_generation.jobs[].results[].id` for true iteration, and `retry_generation` only for failed Default jobs. Retry is a new billed, non-idempotent attempt from its saved inputs. Never create another project merely to change a person, preset, content, title or prompt. Explicit null/empty values clear fields; omission preserves defaults. `improve_prompt` and `prompt_from_reference` cost 0.5 credit each and count toward the MCP daily cap. Upload external references first. Read the exact contracts in references/api.md and references/mcp.md.

## Compatibility

This is the canonical successor to `youthumb-api` and `youthumb-prompts`. Old installed copies may
lack management tools and still claim generation settings are fixed at creation. Read this skill and
its references for current contracts. Do not create duplicate projects to work around old instructions.

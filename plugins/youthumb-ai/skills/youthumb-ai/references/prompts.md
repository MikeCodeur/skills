# Writing YouThumb prompts

How the generator reads a request, how to choose a starting point, the 7-block prompt method with its
libraries, the options, and how to iterate in the same project. Sections marked **App only** cannot be done
through the API or MCP — tell the user to do them in the YouThumb editor, on the same project.

## 1. What the generator receives

Your prompt is sent first and only once. YouThumb then adds, only when they apply:

- the mode directive (named preset style, or "Add text" with the `title`);
- **PERSONS** — the person's name and description, linked to their face photos. If the prompt does not say what
  to do with the person, they are shown as a main subject;
- the list of images: `style_reference` (source image), `@image1…@image6` (content images), face photos;
- **FACE** — keep the exact identity (face, hair, beard, glasses, age, skin tone). Face lighting comes from the
  prompt first, then the source image, otherwise a bright YouTube studio look exposed for the face;
- the options you set (`faceExpression`, `clothingStyle`, `faceEnhancement`, `backgroundBlur`, `textPosition`,
  `negativePrompt` as "Avoid: …");
- output constraints (16:9).

Consequences: describe the scene, not the tool ("generate a thumbnail of…" adds nothing); call the person by the
name registered in YouThumb; any lighting you write on the face wins; a content image whose `@imageN` token is
absent from the prompt is dropped.

## 2. Pick the starting point

| Starting point                                               | Through API / MCP                                                   | Prompt focus                                                          |
| ------------------------------------------------------------ | ------------------------------------------------------------------- | --------------------------------------------------------------------- |
| **Free** — prompt-led composition                            | `presetKey: "free"` (default)                                       | the full scene: subject, background, placement, style                 |
| **Named preset** — a style direction                         | `presetKey` accepted, but API/MCP generations currently run as Free | in the app: subject and story, not a look that contradicts the preset |
| **Source image** — adapt an existing layout                  | `styleReferenceUrl`                                                 | what to change and what must stay                                     |
| **YouTube thumbnail** — import a video's thumbnail as source | `youtubeUrl`                                                        | same as source image; only the thumbnail is imported                  |
| **Template** — a library image as source                     | `templateId` (from `list_templates`)                                | same as source image                                                  |

A source image guides the AI; it does not lock pixels. "Free" is a style name, not free of credits.

Presets (`list_presets`): `free` (no style constraints, prompt only) · `mrbeast-viral` (high energy, bright colors,
expressive faces) · `reaction-shocked` (dramatic reactions, bold text) · `before-after` (split comparison,
transformation) · `challenge-experiment` (action, suspense, outcome teasing) · `gaming-esports` (neon, dynamic action) ·
`tech-review` (clean product shots) · `vlog-travel` (scenic, personal) · `tutorial-educational` (clear visuals, step
indicators) · `cinematic-film` (movie poster, dramatic light) · `dramatic-intense` (high contrast, intense emotion) ·
`mystery-dark` (dark tones, suspense) · `luxury-premium` (elegant, premium) · `office` (tidy desk, warm light,
room for text) · `youtube-studio` (creator studio, ring light). Through the API, borrow these words in a Free prompt.

## 3. People

- One person per project (`personId` or `personName`). Create the profile once, reuse it everywhere.
- Photos: 1–2 sharp, well-lit, front-facing photos of one person, no glasses or cap if possible. The generator uses
  the first two photos; the app allows up to five but more is not better. Avoid group shots, tiny faces, filters.
- Put stable appearance details in the person's **description** ("30-year-old, short beard, green eyes"): it is sent
  with every generation and reduces surprises.
- In the prompt, name the person and give position, framing, pose and expression.

## 4. Reference images (`@image1` … `@image6`)

- Max **6** per project (in the app the sketch counts toward the 6). Tokens follow the order of `contentImages`.
- Write every token with a short description and a role: `@image1 (product logo) top-right, small`.
- Use a real image of a logo or product instead of describing it: shapes and colors come out closer. Still check fine
  logo details and screenshot text in the result — the AI redraws them.
- Fewer elements = each one bigger and clearer. One dominant element + the person usually beats five assets.

## 5. Sketch — **App only**

In the editor, **Sketch** draws a rough 16:9 layout (shapes, arrows, text, face or image stickers) and attaches it as a
reference. It is sent as placement zones (percentages), never as pixels: it gives positions and sizes, the prompt gives
the style. Through the API, write positions in words ("left third", "top-right corner, about a quarter of the width").

## 6. The prompt — 7 blocks

Every prompt describes one complete scene with all seven blocks:

1. **Subject** — person name, position (left / center / right third), framing (close-up, medium, waist up), pose.
2. **Expression** — with micro-details (eyes, eyebrows, mouth).
3. **Assets** — every `@imageN (description)` with placement, size and treatment (glow, angle, glass panel…).
4. **Background** — colors, gradient, glow, depth; one or two accent colors maximum.
5. **Lighting** — key light direction, color temperature, rim light, shadows.
6. **Materials & effects** — glass, chrome, neon, particles, smoke… one or two, not all.
7. **Composition** — layout, camera angle, depth of field, visual hierarchy, space left for a title.

Rules:

- Always: one clear focal subject; every asset placed; lighting stated; cinematic / photographic language;
  80–250 words and **under 2000 characters** (API limit).
- Never: "generate a thumbnail of…"; unplaced assets; busy patterns, geometric fractals, lines everywhere;
  more than two accent colors; a wide-open screaming mouth by default; contradictions between prompt, title and options.
- With a source image, write the change and what stays: "Keep the split composition. Replace the product on the left
  with @image1 and change the background to blue."
- Be concrete: "make it viral" says less than subject, placement and background.

## 7. Five distinct proposals

When the user asks for prompts, collect first: the person's registered name, the assets (map each to `@imageN`), the
video topic, angle (tutorial, comparison, news, opinion, reveal…), emotion and style preference. Then write **5
proposals**, each with its own identity:

| #   | Approach             | Mood                                                 |
| --- | -------------------- | ---------------------------------------------------- |
| 1   | Clean & professional | dark minimal, confident, subtle glow                 |
| 2   | Dramatic & cinematic | strong contrast, dramatic pose, particles            |
| 3   | Bold & colorful      | vibrant accents, energetic composition               |
| 4   | Mysterious / insider | dark mood, secretive expression, smoke, shadows      |
| 5   | Pattern break        | unusual angle, unexpected element, curiosity trigger |

At most one of the five uses "surprised"; default to confident / serious / impressed.
Output each one as:

```
━━━ PROMPT 1 — Clean & Professional ━━━
<full prompt, ready to paste or send>
Expression: confident | Background: dark + teal glow | Assets: @image1 center, @image2 floating
```

Proposals are not `variations`: `variations` (1–4) are several images from ONE prompt. To test several proposals, run
them one after another as generations of the same project (`start` with each `prompt`), one variation each.

## 8. Libraries

### Expressions

| Expression        | When                 | Prompt keywords                                                                | Closest `faceExpression`                    |
| ----------------- | -------------------- | ------------------------------------------------------------------------------ | ------------------------------------------- |
| Confident smirk   | default              | `confident smirk, knowing look, slight head tilt, one eyebrow slightly raised` | `confident`                                 |
| Impressed         | discovery, review    | `impressed but composed, raised eyebrow, subtle smirk, hint of admiration`     | —                                           |
| Serious / focused | deep tech, security  | `focused expression, intense gaze, determined look, sharp eyes`                | `serious`                                   |
| Finger on lips    | secret, insider      | `finger on lips, knowing smirk, secretive look`                                | —                                           |
| Pointing          | call to action       | `pointing at [element], confident expression, direct eye contact`              | `confident`                                 |
| Arms crossed      | authority, opinion   | `arms crossed, confident stance, authoritative look, slight smirk`             | `confident`                                 |
| Warm smile        | lifestyle, good news | `warm genuine smile, relaxed eyes`                                             | `happy`                                     |
| High energy       | challenge, launch    | `energetic, enthusiastic, big smile, dynamic pose`                             | `excited`                                   |
| Subtle surprise   | real wow (rare)      | `subtly surprised, eyebrows slightly raised, mouth slightly open`              | `surprised` (strong: eyes wide, mouth open) |

### Backgrounds

| Mood        | Colors                  | Keywords                                                                   |
| ----------- | ----------------------- | -------------------------------------------------------------------------- |
| Tech / dark | black + orange or teal  | `clean dark background, subtle orange glow, soft ambient light`            |
| Dramatic    | purple + electric blue  | `dark cinematic background, deep purple gradient, electric blue rim light` |
| Minimal     | pure black + one accent | `pure black background, single [color] light source, ultra minimal`        |
| Warm        | dark + amber / gold     | `warm dark background, golden ambient light, subtle bokeh`                 |
| Cold        | dark blue / steel       | `cold steel blue background, desaturated tones, sharp shadows`             |
| Energetic   | vibrant gradient        | `vibrant [color] to [color] gradient, dynamic energy, subtle particles`    |

### Materials & effects

| Effect      | Keywords                                                    | Best for           |
| ----------- | ----------------------------------------------------------- | ------------------ |
| Glass       | `frosted glass overlay, glass morphism, transparent panels` | modern tech        |
| Chrome      | `chrome reflections, metallic surface, brushed steel`       | premium, authority |
| Neon        | `neon glow effect, light trails, luminous edges`            | AI, futuristic     |
| Holographic | `holographic display, iridescent shimmer, prismatic light`  | innovation         |
| Particles   | `floating particles, dust motes in light, subtle sparkles`  | cinematic          |
| Glitch      | `subtle glitch effect, digital distortion, scan lines`      | security, hacking  |
| Smoke       | `wisps of smoke, atmospheric fog, volumetric haze`          | mystery            |
| Embers      | `floating embers, fire particles, warm sparks`              | hot takes          |
| Bokeh       | `bokeh light orbs, out-of-focus lights, dreamy depth`       | personal, vlog     |
| Light rays  | `volumetric light rays, god rays, dramatic light beams`     | announcements      |

### Text & typography

| Element      | Keywords                                                           |
| ------------ | ------------------------------------------------------------------ |
| Bold text    | `bold white text "[TEXT]" in large sans-serif font, high contrast` |
| Glowing text | `glowing [color] text "[TEXT]", neon style lettering`              |
| 3D text      | `3D extruded text "[TEXT]", metallic finish, dramatic shadow`      |
| No text      | `no text, no typography, clean visual only`                        |
| Code snippet | `floating code snippet, monospace font, syntax highlighted`        |

### Composition & camera

| Style           | Keywords                                                                      |
| --------------- | ----------------------------------------------------------------------------- |
| Classic YouTube | `medium shot, person on left third, element on right, shallow depth of field` |
| Centered power  | `centered subject, symmetrical composition, straight-on camera angle`         |
| Dynamic angle   | `slight low angle, dynamic perspective, subject looking down at camera`       |
| Split frame     | `split frame, person on one side, visual element on other, strong contrast`   |
| Close-up        | `close-up portrait, tight crop, intense eye contact, blurred background`      |

## 9. Options (API / MCP `advancedOptions`)

| Option            | Use                                                                                                                                                         |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `faceExpression`  | `happy`, `surprised`, `excited`, `serious`, `confident` add an expression line; `neutral` / `keep-original` add nothing. Keep the prompt consistent with it |
| `faceEnhancement` | `subtle` → `normal` → `enhanced` retouching. Lower it if the face drifts from the person                                                                    |
| `clothingStyle`   | `casual`, `professional`, `sporty`, `elegant`, `streetwear`                                                                                                 |
| `backgroundBlur`  | 0–100 %                                                                                                                                                     |
| `textPosition`    | `top`, `center`, `bottom` for any text; `none` / `keep-original` add nothing                                                                                |
| `negativePrompt`  | ≤ 500 chars, a short list: "extra people, extra logos, cluttered background". A guide, not a filter. Never exclude "text" while asking for a title          |
| `variations`      | 1–4 images from the same request, 1 credit each; start with 1, raise once the composition works                                                             |

## 10. Title and text

- Put the words in `title` (≤ 200 chars) and leave space for them in the prompt ("clear empty area above the laptop for a
  short title"); the prompt describes the scene, the title supplies the words. Or write the text directly in the prompt
  with the typography keywords. Not both with different words.
- Keep it short (2–4 words) to stay readable in a feed; proofread every letter of the result — AI text can misspell.
- `title` and the source image are fixed at project creation.
- **App only — Add Title**: a text-focused pass on a source image with style (e.g. Boxed), color and placement grid
  (Auto, Behind…).

## 11. Lighting

- Write it in the prompt: key light direction, color, rim light, contrast. It overrides the default studio face light.
- Start simple: one key light and one accent; conflicting sources flatten the scene.
- **App only — Add Light**: relight a source image with presets (Studio Pro, Neon Gamer, Cinematic, Golden Hour) or
  custom lights (up to four sources: direction around the subject, high / eye level / low, color; rim light, backlight).

## 12. Iterate in the same project

Every attempt is a new generation (a "run") in the same project; earlier runs and their results stay.

| Goal                                                             | App                                             | API / MCP                                              |
| ---------------------------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------ |
| Same settings, fresh images                                      | **Regenerate** on the run (charges immediately) | `start` with the same `prompt` and options             |
| Change prompt or options, then retry                             | **Reuse this run**, edit, Generate              | `start` with the full new `prompt` / `advancedOptions` |
| Several images of one setup                                      | variation selector 1–4                          | `advancedOptions.variations` 1–4                       |
| Refine ONE finished image (darker background, keep the product…) | **Use as source** on the result                 | **App only**                                           |
| Change person, images, source, title or preset                   | edit the composer                               | **App only** on the same project (fixed at creation)   |

Iteration tips: change one thing per attempt; when the composition is right, keep the prompt and change only the
expression, the lighting words or the title; compare results before choosing (the app compares up to three side by side).

**App only, summary**: Sketch · Improve the prompt (rewrite with axes Composition / Visual hook / Mobile readability,
its own credit cost) · Prompt from the reference image (extraction; accepting it switches the source to Free) ·
Add Light · Add Title · Use as source · Reuse run · Regenerate button · Compare · Favorites · Trash · Download
(through the API, use the result URLs) · Thumbnail preview, YouTube card and avatar tools.
An agent can still do its own prompt rewrite with this file before sending it.

## 13. Example walkthrough

**Person:** "Alex" (registered in YouThumb). **Assets:** `@image1` → frontend framework logo (blue atom icon),
`@image2` → AI coding assistant logo (purple terminal icon). **Video:** building a web app with and without AI —
productivity boost, impressive, tech-forward.

> Alex on the left third of frame, medium shot from waist up, confident smirk with one eyebrow slightly raised, arms
> crossed. @image1 (blue atom framework logo) floating at mid-right, softly glowing with a blue aura. @image2 (purple
> terminal icon of the AI assistant) hovering below and behind, emitting subtle orange light trails. Clean dark
> background with a deep navy-to-black gradient, single teal light source from upper left casting soft directional
> shadows. Subtle floating particles catching the light. Chrome-like reflective floor adding depth. Shallow depth of
> field with background elements slightly soft. No text.

`Expression: confident | Background: dark navy + teal | Assets: @image1 mid-right glowing, @image2 behind lower`

More by video type: `prompt-examples.md`.

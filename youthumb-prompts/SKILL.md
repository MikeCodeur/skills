---
name: youthumb-prompts
description: >
  Generate optimized prompts for YouThumb.ai YouTube thumbnails.
  Guided 4-step workflow: collect person name, map visual assets,
  describe the video, then generate 5 distinct ready-to-paste prompts.
  Use when the user says "thumbnail prompt", "YouThumb prompt",
  "generate thumbnail", "miniature YouTube", "prompt for my thumbnail",
  "help me with YouThumb", or when preparing YouTube thumbnail prompts.
license: MIT
compatibility: Works with any AI coding agent. No tools or scripts required.
metadata:
  author: mikecodeur
  version: "1.0"
  website: https://youthumb.ai
---

# YouThumb Prompts — Thumbnail Prompt Generator

Generate 5 optimized, ready-to-paste prompts for [YouThumb.ai](https://youthumb.ai) thumbnails.

**Follow the 4 steps in order. Do not skip ahead or generate prompts before collecting all info.**

---

## Step 1 — Person

Ask the user:

> **Who will be on the thumbnail?**
> Give me the name exactly as registered in YouThumb.ai.

Store the name. This is the subject referenced in every prompt.

If the user hasn't created a Person yet:
> Go to YouThumb.ai → Persons → Create. Upload 3-5 clear face photos (different angles, good lighting). Name it exactly as you want to reference it.

---

## Step 2 — Assets

Ask the user:

> **What visual elements should appear on the thumbnail?**
> List each with:
> - **Name** — what it is (logo, screenshot, icon...)
> - **Description** — what it looks like or represents
>
> Examples:
> - "Claude Code logo — purple/orange terminal icon"
> - "Screenshot of my app — dark theme with charts"

Map each asset to `@image1`, `@image2`, `@image3` etc. (max 5, in order given).

Display the mapping back:
```
@image1 → Claude Code logo (purple/orange terminal icon)
@image2 → VS Code logo (blue code editor icon)
```

⚠️ **Every asset MUST be referenced as `@imageN (description)` in the prompts.** Without this, YouThumb will NOT include the image.

---

## Step 3 — Video Description

Ask the user:

> **Describe your video:**
> - What's the topic?
> - What's the angle/hook? (comparison, tutorial, news, opinion, reveal...)
> - What emotion should the thumbnail convey?
> - Any style preference? (dark, colorful, minimal, dramatic...)

---

## Step 4 — Generate 5 Prompts

Using Steps 1-3, generate **5 distinct prompt variations**, each 100-300 words.

### Prompt Structure (all 7 blocks required in each prompt)

Every prompt must describe a complete scene including ALL of these elements:

1. **Subject** — Person name, position (left/center/right), pose, body framing
2. **Expression** — Face expression with micro-details (eyes, mouth, eyebrows)
3. **Assets** — Every `@imageN (description)` with placement, size, and visual treatment
4. **Background** — Color scheme, gradient, glow, particles, depth
5. **Lighting** — Light source direction, intensity, color temperature, shadows
6. **Materials & Effects** — Textures, reflections, distortions, glitch, glass, chrome, particles
7. **Composition** — Layout, depth of field, camera angle, visual hierarchy

### The 5 Variations

Each prompt must have a **distinct visual identity**:

| # | Approach | Mood |
|---|----------|------|
| 1 | **Clean & Professional** | Dark minimal, confident expression, subtle glow |
| 2 | **Dramatic & Cinematic** | Strong contrast, dramatic pose, particles/effects |
| 3 | **Bold & Colorful** | Vibrant accents, energetic composition, eye-catching |
| 4 | **Mysterious / Insider** | Dark mood, secretive expression, smoke/shadows |
| 5 | **Pattern Break** | Unusual angle, unexpected element, curiosity trigger |

---

## Reference Libraries

Use these to build rich, specific prompts. Mix and match across prompts.

### Expressions

| Expression | When to use | Prompt keywords |
|-----------|-------------|-----------------|
| **Confident smirk** | Default for most | `confident smirk, knowing look, slight head tilt, one eyebrow slightly raised` |
| **Impressed** | Discovery, review | `impressed but composed, raised eyebrow, subtle smirk, hint of admiration` |
| **Serious/focused** | Deep tech, security | `focused expression, intense gaze, determined look, sharp eyes` |
| **Finger on lips** | Secret, insider | `finger on lips, knowing smirk, secretive look` |
| **Pointing** | Call to action | `pointing at [element], confident expression, direct eye contact` |
| **Arms crossed** | Authority, opinion | `arms crossed, confident stance, authoritative look, slight smirk` |
| **Subtle surprise** | Genuine wow (rare) | `subtly surprised, eyebrows slightly raised, mouth slightly open` |

⚠️ **Rule: max 1 out of 5 prompts can use "surprised". Default = confident/serious/impressed.**

### Backgrounds

| Mood | Colors | Keywords |
|------|--------|----------|
| Tech/dark | Black + orange/teal | `clean dark background, subtle orange glow, soft ambient light` |
| Dramatic | Purple + electric blue | `dark cinematic background, deep purple gradient, electric blue rim light` |
| Minimal | Pure black + one accent | `pure black background, single [color] light source, ultra minimal` |
| Warm | Dark + amber/gold | `warm dark background, golden ambient light, subtle bokeh` |
| Cold | Dark blue/steel | `cold steel blue background, desaturated tones, sharp shadows` |
| Energetic | Vibrant gradient | `vibrant [color] to [color] gradient, dynamic energy, subtle particles` |

⚠️ **Never: busy patterns, geometric fractals, lines everywhere, more than 2 accent colors.**

### Materials & Effects

| Effect | Keywords | Best for |
|--------|----------|----------|
| Glass | `frosted glass overlay, glass morphism, transparent panels` | Modern tech |
| Chrome | `chrome reflections, metallic surface, brushed steel` | Premium/authority |
| Neon | `neon glow effect, light trails, luminous edges` | AI/futuristic |
| Holographic | `holographic display, iridescent shimmer, prismatic light` | Innovation |
| Particles | `floating particles, dust motes in light, subtle sparkles` | Cinematic |
| Glitch | `subtle glitch effect, digital distortion, scan lines` | Security/hacking |
| Smoke | `wisps of smoke, atmospheric fog, volumetric haze` | Mystery |
| Fire/embers | `floating embers, fire particles, warm sparks` | Hot takes |
| Bokeh | `bokeh light orbs, out-of-focus lights, dreamy depth` | Personal/vlog |
| Light rays | `volumetric light rays, god rays, dramatic light beams` | Announcements |

### Text & Typography

| Element | Keywords |
|---------|----------|
| Bold text | `bold white text "[TEXT]" in large sans-serif font, high contrast` |
| Glowing text | `glowing [color] text "[TEXT]", neon style lettering` |
| 3D text | `3D extruded text "[TEXT]", metallic finish, dramatic shadow` |
| No text | `no text, no typography, clean visual only` |
| Code snippet | `floating code snippet, monospace font, syntax highlighted` |

### Composition & Camera

| Style | Keywords |
|-------|----------|
| Classic YouTube | `medium shot, person on left third, element on right, shallow depth of field` |
| Centered power | `centered subject, symmetrical composition, straight-on camera angle` |
| Dynamic angle | `slight low angle, dynamic perspective, subject looking down at camera` |
| Split frame | `split frame, person on one side, visual element on other, strong contrast` |
| Close-up | `close-up portrait, tight crop, intense eye contact, blurred background` |

---

## Output Format

Present each prompt like this:

```
━━━ PROMPT 1 — Clean & Professional ━━━

[Full prompt text ready to paste into YouThumb.ai]

Expression: confident | Background: dark + teal glow | Assets: @image1 center, @image2 floating

━━━ PROMPT 2 — Dramatic & Cinematic ━━━

[Full prompt text...]

Expression: serious | Background: purple + blue rim | Assets: @image1 large left

...etc for all 5
```

Add a **one-line summary** under each prompt (expression + background + asset placement).

---

## Prompt Writing Rules

### ✅ Always
- Reference EVERY asset with `@imageN (description)` — or it won't appear
- Specify person position (left/center/right) and framing (medium/close-up)
- Include lighting direction and color
- Describe the scene as a whole, not a list of disconnected elements
- Use cinematic/photographic language
- Keep each prompt between 100-300 words

### ❌ Never
- Don't write "generate a thumbnail of..." — describe the scene directly
- Don't leave assets unplaced — every `@imageN` needs position + visual treatment
- Don't use more than 2 accent colors per prompt
- Don't describe cluttered backgrounds (lines, geometric patterns, fractals)
- Don't default to "mouth wide open surprised" expression
- Don't forget lighting — it makes or breaks the mood

---

## YouThumb.ai Advanced Options Reference

When pasting prompts into YouThumb.ai, users can also set:

| Option | Values | Default |
|--------|--------|---------|
| Face Expression | neutral, happy, surprised, excited, serious, confident, keep-original | **confident** |
| Text Position | top, center, bottom, none, keep-original | **none** |
| Variations | 1-4 | **1** (each costs credits) |
| Clothing Style | casual, professional, sporty, elegant, streetwear, keep-original | keep-original |
| Negative Prompt | Max 500 chars — exclude unwanted elements | — |
| Background Blur | 0-100 | 0 |
| Face Enhancement | subtle, normal, enhanced, keep-original | **normal** |

---

## Example Walkthrough

**Step 1 — Person:** "Alex Dev"

**Step 2 — Assets:**
- @image1 → React logo (blue atom icon)
- @image2 → Claude Code logo (purple/orange terminal icon)

**Step 3 — Video:** "Comparing React development with and without AI. Angle: productivity boost. Mood: impressive, tech-forward."

**One of the 5 generated prompts:**

> Alex Dev on the left third of frame, medium shot from waist up, confident smirk with one eyebrow slightly raised, arms crossed. @image1 (React blue atom logo) floating at mid-right, softly glowing with a blue aura. @image2 (Claude Code purple terminal icon) hovering below and behind, emitting subtle orange light trails. Clean dark background with a deep navy-to-black gradient, single teal light source from upper left casting soft directional shadows. Subtle floating particles catching the light. Chrome-like reflective floor adding depth. Shallow depth of field with background elements slightly soft. No text.
>
> Expression: confident | Background: dark navy + teal | Assets: @image1 mid-right glowing, @image2 behind lower

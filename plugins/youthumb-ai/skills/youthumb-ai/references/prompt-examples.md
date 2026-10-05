# Prompt examples by video type

Full prompts for common YouTube video categories. "Alex" stands for the person registered in YouThumb; replace it
with the user's own registered name. Products and logos are generic: use the user's real assets as `@imageN`.
Adapt the structure and style to the video — don't copy them as is.

---

## Tutorial / how-to

**Context:** a developer showing how to automate tests with an AI coding assistant.

**Assets:**

- @image1 → AI coding assistant logo (orange terminal icon)
- @image2 → screenshot of test results (green checkmarks on a dark terminal)

**Prompt:**

> Alex standing on the left third of frame, medium shot from waist up, confident smirk with a knowing look and slight head tilt, right hand pointing toward the floating elements on the right side. @image1 (orange terminal icon of the AI assistant) displayed prominently at center-right, glowing with a soft orange aura and subtle light rays emanating outward. @image2 (test results screenshot with green checkmarks) floating behind and slightly below the logo, angled at 15 degrees with a frosted glass overlay effect. Clean dark background with a deep charcoal-to-black gradient, warm orange glow emanating from the right side where the assets are placed. Single key light from upper-left casting soft shadows on the face. Floating code particles dissolving in the background, very subtle. Shallow depth of field keeping the subject and main logo sharp while the screenshot softens. No text.

`Expression: confident | Background: dark + warm orange glow | Assets: @image1 center-right glowing, @image2 behind angled`

---

## Comparison / VS

**Context:** comparing two code editors for a daily development workflow.

**Assets:**

- @image1 → code editor A logo (dark angular icon)
- @image2 → code editor B logo (orange terminal icon)

**Prompt:**

> Alex centered in frame, close-up from chest up, arms crossed with an authoritative stance, serious focused expression with sharp eyes and a subtle evaluating look, slight head tilt to the right. @image1 (dark angular logo of editor A) floating on the left side at shoulder height, surrounded by a cool blue neon glow and faint electric particles. @image2 (orange terminal logo of editor B) floating on the right side at matching height, radiating warm orange light trails. A visible divide running down the center of the frame — left side tinted cold steel blue, right side tinted warm amber. Dark cinematic background with subtle smoke wisps at the bottom. Strong rim light from both sides creating a dramatic silhouette edge. Chrome-like reflective surface below adding depth. Tight symmetrical composition emphasizing the face-off between the two tools. No text.

`Expression: serious / evaluating | Background: split cold blue vs warm amber | Assets: @image1 left with blue glow, @image2 right with orange glow`

---

## News / announcement

**Context:** a major tech company just released a new AI design tool.

**Assets:**

- @image1 → the company's multicolor logo
- @image2 → screenshot of the new tool's interface (colorful UI builder)

**Prompt:**

> Alex on the right third of frame, medium-close shot, impressed expression with raised eyebrow and subtle smirk hinting at admiration, one hand slightly raised in a presenting gesture toward the left. @image1 (multicolor company logo) large and prominent on the upper left, casting colorful light reflections — red, blue, yellow, green — onto the surrounding area. @image2 (UI builder screenshot) displayed below the logo as a floating holographic panel, slight 3D perspective tilt, edges glowing with a soft white luminous border. Dark background with a deep navy gradient, volumetric light rays streaming from behind the logo creating a dramatic announcement feel. Subtle prismatic light refractions scattered across the scene. Soft fill light on the face from the front, colored light bouncing from the logo creating subtle multicolor highlights on the skin. Wide composition giving space to the visual elements. No text.

`Expression: impressed | Background: dark navy + multicolor light rays | Assets: @image1 upper-left large, @image2 holographic panel below`

---

## Opinion / hot take

**Context:** a video arguing that tool fatigue is real and nobody talks about it.

**Assets:** none — pure face-driven thumbnail.

**Prompt:**

> Alex centered in frame, tight close-up portrait from shoulders up, slightly tired but determined expression — eyes intense and direct, jaw slightly clenched, subtle bags under the eyes suggesting long hours, a knowing half-smile that says "I've been through it." Dark moody background with a deep desaturated blue-to-black gradient, single warm amber key light from the left side creating strong shadows on the right half of the face. Wisps of smoke drifting slowly at the bottom of the frame, atmospheric and cinematic. Just three or four floating embers drifting upward near the edges, barely visible; no other effects — intentionally stripped down and raw. Very shallow depth of field, background completely blurred. The mood is intimate, serious, personal. No text, no logos, no visual clutter.

`Expression: tired but determined | Background: dark desaturated blue + amber key light | Assets: none — face only`

---

## Product review / demo

**Context:** reviewing a new web app starter kit with a live demo.

**Assets:**

- @image1 → product logo (modern gradient icon)
- @image2 → screenshot of the dashboard (dark theme, data panels)
- @image3 → tech stack icons grouped together

**Prompt:**

> Alex on the left third, medium shot, confident smirk with finger on lips in a "shh, I found something good" gesture, playful knowing look in the eyes. @image1 (product gradient logo) floating at center frame, slightly above eye level, with a holographic shimmer effect and iridescent edges catching the light. @image2 (dark dashboard screenshot) displayed as a large floating panel behind the logo, slight perspective rotation, frosted glass edges, subtle glow from the data elements within. @image3 (tech stack icons) arranged in a small cluster at bottom-right, each with a faint glow in its own color. Clean dark background with a rich purple-to-black gradient. Soft neon purple glow from behind the main elements creating depth. Key light from upper-right, soft and diffused. Bokeh light orbs scattered in the deep background, very subtle. No text.

`Expression: confident "shh" gesture | Background: dark purple gradient + neon glow | Assets: @image1 center holographic, @image2 floating panel behind, @image3 cluster bottom-right`

---

## Security / exposé

**Context:** auditing leaked source code and finding hidden secrets.

**Assets:**

- @image1 → terminal screenshot (code with highlighted secrets)

**Prompt:**

> Alex on the right side of frame, medium-close shot, serious focused expression with intense gaze and slightly narrowed eyes, one hand raised near the chin in a thoughtful analyzing pose. @image1 (terminal screenshot with highlighted code) displayed as a large glitchy floating panel on the left side, subtle digital distortion around the edges — scan lines, chromatic aberration, pixel scatter — as if the data is unstable or corrupted. Pure black background, deep red warning glow emanating from behind the code panel. Subtle glitch fragments scattered across the scene. Single cold white key light from the right creating harsh shadows, contrasting with the warm red glow from the left. Thin green scan line crossing the image horizontally at eye level. Atmospheric fog at the bottom. The mood is investigative, tense, serious. No text.

`Expression: serious / analytical | Background: black + red warning glow | Assets: @image1 large left with glitch distortion`

---

## Adapting a source image (styleReferenceUrl / youtubeUrl / templateId)

**Context:** reusing the layout of an existing review thumbnail for a new product.

**Assets:**

- @image1 → the new product photo

**Prompt:**

> Keep the split composition and the face position on the right. Replace the product on the left with @image1, larger and sharply lit. Change the background to a deep blue gradient with a soft glow behind the product. Keep the same bold, high-contrast look.

`Starting point: source image | Focus: what changes, what stays`

---

## Tips for adapting examples

- **Swap the expression** to match the video's energy — don't default to the same one.
- **Adjust asset placement** to the number of elements — fewer assets = bigger and more prominent.
- **Match the background mood** to the topic — warm for positive, cold for serious, red for danger.
- **Keep it simple** — one dominant visual element + the person usually beats cramming everything in.
- **Stay under 2000 characters** — the API rejects longer prompts.

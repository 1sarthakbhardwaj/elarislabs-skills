---
name: static-to-video
description: >-
  Turn a reference image or a plain brief into an on-brand stop-motion video using the ElarisLabs
  Creative Studio (MCP tools). Breaks the look into an editable spec, generates on-brand stills with
  gpt-image-2 or nano-banana-2, waits for approval, animates the approved frames into stop-motion with
  Seedance 2.0 (Kling fallback when a human is in frame, Gemini Omni for edits), and appends a branded
  end screen with the user's logo. Use when the user asks to make a video from a still/reference image,
  a "static to video", stop-motion, product animation, or an on-brand promo from a brief.
disable-model-invocation: true
---

# Static → Video

Turn a **reference image or a brief** into an **on-brand stop-motion video**, then cap it with a
**branded end screen**. Runs on the ElarisLabs MCP tools (see `reference.md` for exact calls).

## Golden rules

- **Never skip the approval gate.** Generate stills, show them, and wait for the user to pick before
  spending credits on video.
- **Ask for brand assets up front** (logo, name, colors) — the deliverable is branded, and the end
  screen needs them.
- **Everything is billed to the brand** the API key is bound to. Check `get_credits` before a large run
  and estimate cost before animating.
- **The look spec is editable.** Show it, let the user tweak it, then generate from the final spec.

## Workflow

Copy this checklist and track progress:

```
- [ ] 1. Intake — collect reference/brief + brand assets
- [ ] 2. Look Spec — derive an editable spec, get sign-off
- [ ] 2b. Aspect reframe — if source AR ≠ target, nano-banana-2 first
- [ ] 3. Stills — generate on-brand frames (nano-banana-2 / gpt-image-2)
- [ ] 4. Approval gate — user selects keepers
- [ ] 5. Animate — stop-motion per approved still (Seedance / Kling / Gemini)
- [ ] 6. End screen — render branded outro from the template
- [ ] 7. Assemble & deliver — stitch clips + end screen
```

### 1. Intake

Accept **either** a reference image URL **or** a text brief. Then ask for the brand kit if not given:

- **Brand logo** (transparent PNG or SVG) — required for the end screen. Ask explicitly.
- **Brand name** + optional **tagline / CTA**.
- **Primary + secondary colors** (hex). Fall back to palette pulled from the reference image.
- **Aspect ratio** (default `9:16` for social, `16:9` for landscape) and **rough length**.

If a reference image was given, run `score_creative` (optional) or just read it to seed the spec.

### 2. Look Spec (editable)

Break the look down into this structured spec and **show it to the user for edits** before generating:

```yaml
subject:        # what's in frame (product, character, scene)
has_human:      # true | false  ← drives the video model routing in step 5
style:          # e.g. tactile clay stop-motion, paper-craft, product macro
palette:        # 3–5 hex colors (brand + accents)
lighting:       # e.g. soft key + warm rim
composition:    # framing, camera angle, negative space for text
mood:           # 2–3 adjectives
motion_feel:    # stop-motion cadence: snappy | dreamy | mechanical
frames:         # 3–6 beats, each one sentence (these become the stills)
aspect_ratio:   # 9:16 | 16:9 | 1:1
```

Each entry in `frames` becomes one still in step 3. Keep 3–6 frames for a tight stop-motion loop.

### 2b. Aspect reframe (required when AR differs)

After look-spec sign-off, compare the **source image aspect ratio** to `aspect_ratio` in the spec.

- If they **match** → skip; use the original as the master reference.
- If they **differ** → call `generate_image` with `model: "nano-banana-2"`, the original as
  `referenceImages`, and a reframe prompt that asks to **recompose / outpaint into the target ratio**
  while keeping product, lighting, typography style, and brand look. Use the returned image as the
  **master reference** for every later still. Show the reframed canvas once before generating frames.

Do **not** stretch or letterbox with naive crop alone — Nano Banana 2 should invent plausible
surrounding scene so the composition still reads as a designed ad at the target ratio.

See `reference.md` for the exact MCP call and prompt pattern.

### 3. Generate stills

For each frame in the spec, call `generate_image`. Pick the model by need:

| Need | Model (`model` arg) |
|------|---------------------|
| Aspect reframe when source AR ≠ target AR | `nano-banana-2` |
| Default on-brand stills, character/product consistency | `nano-banana-2` |
| Text baked into the image, precise multi-reference edits, premium fidelity | `gpt-image-2` |
| Fast/cheap draft passes | `flux-2-flash` |

- Pass the **master reference** (original or reframed) and the brand logo when it should appear as
  `referenceImages` to keep frames consistent and on-brand.
- Generate 1–2 variants per frame. Keep resolution `1K`–`2K` for stills that will be animated.

### 4. Approval gate — REQUIRED

Present every still. Ask the user to select which ones to animate. **Do not proceed to video until they
confirm.** Only approved stills move to step 5.

### 5. Animate into stop-motion

Animate **each approved still** as a short image-to-video clip (2–4s) with stop-motion cadence. Route
the model by the `has_human` flag and the task — see the routing table:

| Situation | Model | Notes |
|-----------|-------|-------|
| **Default (no human in frame)** | **Seedance 2.0** (`seedance-2`, or `seedance-2-fast`) | Best for object/product/scene stop-motion |
| **A human / person is in frame** | **Kling** (`kling-v3`, or `kling-o3`) | Better human motion; use when the still has people |
| **Editing / restyling an existing clip or frame** | **Gemini Omni** (`gemini-omni-flash`) | Use for edits, not first-pass animation |

Stop-motion prompt guidance: describe **incremental, snappy motion** ("frame-by-frame stop-motion,
slight pose shifts, subtle handheld jitter, 12fps feel"), short duration, and hold the subject stable.

> **MCP caveat:** the MCP `generate_video` tool **auto-routes** and does not take a `model` argument.
> To *pin* Seedance / Kling / Gemini you have two options — see `reference.md`:
> **(A)** call `generate_video` with the still as `imageUrl` and state the intended model + motion in the
> prompt (router usually honors the still's content), or
> **(B)** use the Studio video node / the internal `video-studio` router where the model id is explicit.

Before animating, estimate credits and confirm with the user for larger batches.

### 6. Branded end screen

Render the outro from the template at **`assets/end-screen.html`**. Fill the placeholders:

- `{{BRAND_LOGO}}` → user's logo URL (transparent)
- `{{BRAND_NAME}}`, `{{TAGLINE}}`, `{{CTA}}`
- `{{PRIMARY_COLOR}}`, `{{SECONDARY_COLOR}}`, `{{ACCENT_COLOR}}`, `{{TEXT_COLOR}}`

Match the aspect ratio to the video. Render the HTML to a 2–3s clip (via the HyperFrames pipeline / the
app's render route) or export a final frame and hold it. See `reference.md` for rendering options.

### 7. Assemble & deliver

Concatenate the approved clips in `frames` order, then append the end-screen clip. Deliver the final
video plus a one-line summary (frames used, models used, credits spent). Offer to `schedule_post` it to a
connected social account if the user wants.

## Additional resources

- Exact MCP calls, model IDs, fallbacks, and rendering options: [reference.md](reference.md)
- Branded outro template: [assets/end-screen.html](assets/end-screen.html)

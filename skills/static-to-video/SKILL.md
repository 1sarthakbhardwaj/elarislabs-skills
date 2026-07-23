---
name: static-to-video
description: >-
  Turn a reference image or a plain brief into an on-brand stop-motion video using the ElarisLabs
  Creative Studio (MCP tools). Breaks the look into an editable spec, generates on-brand stills with
  gpt-image-2 or nano-banana-2, waits for approval, animates the approved frames with Seedance 2.0
  (Kling O3 Pro fallback when a human/character is in frame, Veo 3.1 as last resort), and assembles the
  clips + a branded end card + BGM in HyperFrames. Use when the user asks to make a video from a still/reference image,
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
- [ ] 5. Animate — per approved still (Seedance 2.0 → Kling O3 Pro → Veo 3.1, pinned via `model`)
- [ ] 6. Assemble in HyperFrames — clips + branded end card + BGM (Asian Paints style)
- [ ] 7. Deliver — final video + summary (frames, model used, credits)
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

### 5. Animate — pin the model (video guard)

Animate **each approved still** as a short image-to-video clip (2–5s). **Decide the model upfront from
the brief/reference**, then **pin it** with the `generate_video` `model` argument (the MCP tool now
accepts `model` — pass an id from `list_models`). Follow this fallback chain:

| Priority | Situation | Model (`model` arg) | Why |
|----------|-----------|---------------------|-----|
| **1 — primary** | No human / character in frame (product, food, scene, abstract) | **Seedance 2.0** → `seedance2-std-i2v` (or `seedance2fast-fast-i2v`) | Top choice for object/product/scene motion |
| **2 — fallback** | **A human or character is in frame** (Seedance won't reliably pass people) | **Kling O3 Pro** → `kling-o3-pro-i2v` | Best human/character motion |
| **3 — last resort** | Neither works for the shot (Seedance + Kling both fail/refuse) | **Veo 3.1** → `veo31-std-i2v` | Final fallback only |

Rules:
- **Choose upfront:** if the reference/spec has people or characters, go **straight to Kling O3 Pro** —
  don't waste a Seedance pass. Otherwise Seedance 2.0.
- **Always pass `model`.** Never rely on the server default (it may pick Veo). After each call, read
  the result's `modelUsed` (or `get_generation_history`) to confirm the right model ran; regenerate if not.
- **Never Veo 3.1 as a default** — it's only the last-resort fallback.

Motion prompt guidance: describe **incremental, snappy motion** ("frame-by-frame stop-motion, slight
pose shifts, subtle handheld jitter, 12fps feel"), short duration, hold the subject stable. Estimate
credits and confirm before large batches.

### 6. Assemble in HyperFrames (Asian Paints style)

**Assembly is done in HyperFrames, not by chaining another video model.** Mirror the Asian Paints /
product-launch build: bring the Seedance clip in as a track, then compose a **branded end card** and layer
**BGM** (no voiceover unless asked).

- End card treatment (matches Asian Paints): dark ground, brand color as the single accent, Inter, a
  short **kinetic-type barrage** resolving on the **wordmark + CTA pill**. Source placeholders from the
  brand kit; fall back to the template at **`assets/end-screen.html`** for tokens.
- **The logo on the end card is MANDATORY.** Use the user's brand logo if given. Otherwise **crop the
  logo/seal from the master reference** (isolate the mark, mask its background to transparent). If the
  crop is unusable, **regenerate a clean transparent logo with `nano-banana-2`**, passing the cropped
  mark as `referenceImages`. Place it as the hero element above the eyebrow, animated in first.
- Set the HyperFrames canvas to the target AR (16:9 → 1920×1080). BGM with a hit on the end-card reveal.
- Render via the HyperFrames pipeline / the app's render route. See `reference.md` for the calls.

### 7. Deliver

Deliver the final rendered video plus a one-line summary (frames used, **model that actually ran**,
credits spent). Offer to `schedule_post` it to a connected social account if the user wants.

## Additional resources

- Exact MCP calls, model IDs, fallbacks, and rendering options: [reference.md](reference.md)
- Branded outro template: [assets/end-screen.html](assets/end-screen.html)

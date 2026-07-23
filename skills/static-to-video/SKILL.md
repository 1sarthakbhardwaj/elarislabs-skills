---
name: static-to-video
description: >-
  Turn a reference image or a plain brief into an on-brand stop-motion video using the ElarisLabs
  Creative Studio (MCP tools). Always confirms the brief first (aspect ratio, total length, style, brand
  assets — never assumed) AND whether the input is a finished creative or a brief: a finished, on-brand
  creative is animated directly as the hero seed (no new stills); only a brief/rough reference goes
  through still generation. Breaks the look into an editable spec, generates on-brand stills with
  gpt-image-2 or nano-banana-2, waits for approval, animates the approved frames with an AR-capable model
  (Seedance 2.0 default, Kling O3 Pro when a human/character is in frame, Veo 3.1 last resort), and
  assembles the clips + a mandatory-logo branded end card + BGM in HyperFrames (AI clip + ~3s end card =
  total). Use when the user asks to make a video from a still/reference image, a "static to video",
  stop-motion, product animation, or an on-brand promo from a brief.
disable-model-invocation: true
---

# Static → Video

Turn a **reference image or a brief** into an **on-brand stop-motion video**, then cap it with a
**branded end screen**. Runs on the ElarisLabs MCP tools (see `reference.md` for exact calls).

## Golden rules

- **Decide the input type FIRST: finished creative vs. brief.** This is the most important fork.
  - **Finished creative** (the user hands over a complete, on-brand ad/still that already reads as
    designed — e.g. a product hero with the logo/type/layout already in it): treat it as the **hero
    seed**. **Do NOT regenerate stills.** Reframe only if AR differs, then animate the hero *directly*.
    Only use `generate_image` to extract/rebuild the transparent logo for the end card.
  - **Brief or rough/loose reference** (a text prompt, mood image, or "make me an ad"): run the **full
    spec path** — derive frames, generate stills, approval gate, then animate.
  - When unsure, **ask** ("Should I animate this creative as-is, or generate new frames from it?").
    Regenerating a finished creative wastes credits and drifts from the brand's real artwork.
- **Ask what the user actually needs FIRST, then start.** Never assume aspect ratio, length, or style.
  Confirm the brief (Step 0) before generating anything. Defaults are only a *fallback* when the user
  explicitly says "you decide."
- **Never assume aspect ratio.** Use what the user gives; if unspecified, ask (`9:16` / `16:9` / `1:1` /
  `4:5`). **The chosen video model MUST support the target AR** — if the primary model can't do it,
  switch to a model that does (see the AR→model rule in step 5), never stretch or letterbox.
- **Budget total length as AI clip + end screen.** When the user gives a total (e.g. "15s"), the AI
  animation is `total − end_screen` and the end screen is ~3s. So 15s → **~12s AI + ~3s end card**.
  Keep this split unless the user overrides either number.
- **Never skip the approval gate.** On the brief path, generate stills, show them, and wait for the user
  to pick before spending credits on video. On the hero-seed path there are no new stills — instead
  confirm the (reframed) hero is the frame to animate before spending video credits.
- **Ask for brand assets up front** (logo, name, colors) — the deliverable is branded, and the end
  screen needs them.
- **Everything is billed to the brand** the API key is bound to. Check `get_credits` before a large run
  and estimate cost before animating.
- **The look spec is editable.** Show it, let the user tweak it, then generate from the final spec.

## Workflow

Copy this checklist and track progress:

```
- [ ] 0. Confirm needs — ASK first: input type (finished creative vs brief), aspect ratio, length, style, brand assets.
- [ ] 1. Intake — collect reference/brief + brand assets
- [ ] 2. Look Spec — derive an editable spec (with confirmed AR + length budget), get sign-off
        (brief path only; for a finished creative, just read its look — don't invent new frames)
- [ ] 2b. Aspect reframe — if source AR ≠ target, nano-banana-2 first
- [ ] 3. Frame to animate:
        • HERO-SEED (finished creative): use the (reframed) creative AS-IS. Skip still generation.
        • BRIEF path: generate on-brand stills from the spec (nano-banana-2 / gpt-image-2).
- [ ] 4. Approval gate — brief path: user selects keepers. hero path: confirm the hero frame.
- [ ] 5. Animate — the approved frame(s) (AR-capable model; Seedance 2.0 → Kling O3 Pro → Veo 3.1, pinned via `model`)
- [ ] 6. Assemble in HyperFrames — clips + branded end card + BGM (Asian Paints style)
- [ ] 7. Deliver — final video + summary (frames, model used, credits)
```

### 0. Confirm needs (do this before anything)

**Always ask the user what they actually need before starting.** Do not assume. Confirm at minimum:

- **Input type (decide this first).** Is the provided image a **finished creative** (a complete, on-brand
  ad/hero that already reads as designed) or a **brief/rough reference**? A finished creative → **hero-seed
  path** (animate it directly, no new stills). A brief → **full spec path** (generate stills). If it's
  ambiguous, ask: "Animate this creative as-is, or generate fresh frames from it?" Default for a polished
  hero is animate-as-is — never silently regenerate someone's finished artwork.
- **Aspect ratio** — `9:16`, `16:9`, `1:1`, or `4:5`. If they give one, use it. If not, ask. Only fall
  back to a default if they say "you choose."
- **Total length** — then split it: **AI clip = total − ~3s end screen**. E.g. "15s" → ~12s AI + ~3s end
  card. Confirm the split.
- **Style / mood**, **brand assets** (logo, name, colors, CTA), and whether they have a reference image.

Echo the confirmed AR + length split back to the user, then proceed. The rest of the workflow inherits
these values.

### 1. Intake

Accept **either** a reference image URL **or** a text brief. Then ask for the brand kit if not given:

- **Brand logo** (transparent PNG or SVG) — required for the end screen. Ask explicitly.
- **Brand name** + optional **tagline / CTA**.
- **Primary + secondary colors** (hex). Fall back to palette pulled from the reference image.
- **Aspect ratio** — whatever the user specified in Step 0 (no silent default). **Rough length** — the
  total the user wants, already split into AI clip + ~3s end card.

If a reference image was given, run `score_creative` (optional) or just read it to seed the spec.

**Local files → get a public URL first.** Every generation tool needs a public https URL; the server can't
read your disk. **Never declare yourself blocked** — pick the path for your client:
- **MCP-only client (Claude Desktop/web, no shell):** if the user **attached the image**, call
  `upload_asset` and pass that attachment as `source`. The client supplies the bytes — do **not** retype
  base64 or refuse because it "looks too big." Returns a hosted `url`.
- **Client with a shell (Cursor/local):** call `create_upload_url` and use the `relay` curl (POST bytes to
  the allowlisted MCP host; server forwards to storage) → `file_url`.
- **Image already at a reachable URL:** `upload_asset` with `rehost: true`.
- **Last resort:** ask the user to upload it in Creative Studio and paste the asset URL.

All upload tools are free. Full details + curl in `reference.md` → "Getting a local file into storage".

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
aspect_ratio:   # 9:16 | 16:9 | 1:1 | 4:5  ← from the user (Step 0), never assumed
total_seconds:  # total video length the user asked for
ai_seconds:     # = total_seconds − end_seconds  (the animated clip[s])
end_seconds:    # ~3 (the branded end card)
```

Each entry in `frames` becomes one still in step 3. Keep 3–6 frames for a tight stop-motion loop.
`ai_seconds` is the budget for the animated clip(s); `end_seconds` (~3) is the end card. They must sum to
`total_seconds`.

### 2b. Aspect reframe (required when AR differs)

After look-spec sign-off, compare the **source image aspect ratio** to the **user-confirmed
`aspect_ratio`** in the spec (from Step 0 — never a silent default).

- If they **match** → skip; use the original as the master reference.
- If they **differ** → call `generate_image` with `model: "nano-banana-2"`, the original as
  `referenceImages`, and a reframe prompt that asks to **recompose / outpaint into the target ratio**
  while keeping product, lighting, typography style, and brand look. Use the returned image as the
  **master reference** for every later still. Show the reframed canvas once before generating frames.

Do **not** stretch or letterbox with naive crop alone — Nano Banana 2 should invent plausible
surrounding scene so the composition still reads as a designed ad at the target ratio.

See `reference.md` for the exact MCP call and prompt pattern.

### 3. Frame(s) to animate — fork by input type

**Hero-seed path (finished creative — the common case for a supplied ad/hero image):**

- **Do NOT generate stills.** The user's creative (or its reframed version from Step 2b) **is** the frame
  to animate. Regenerating it re-invents the brand's own artwork and wastes credits.
- The only `generate_image` call on this path is to **build the transparent logo** for the end card
  (crop the mark from the creative, mask to transparent; if unusable, rebuild with `nano-banana-2` using
  the crop as `referenceImages` — see Step 6).
- Go straight to the approval gate (confirm the hero frame), then animate.

**Brief path (text brief / rough or mood reference — no finished creative):**

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

- **Brief path:** present every still and ask the user to select which ones to animate.
- **Hero-seed path:** confirm the (reframed) hero creative is the frame to animate.

**Do not proceed to video until they confirm.** Only approved frames move to step 5.

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
- **The model MUST support the target aspect ratio.** Before picking, check the model's `aspectRatios`
  via `list_models` / `reference.md`. If the priority model can't do the requested AR (e.g. user wants
  `1:1` but the chosen i2v variant is 16:9-only), **switch to a model that supports that AR** rather than
  changing the user's AR. AR-capability beats the priority order — pick the highest-priority model that
  *both* fits the human/no-human situation *and* supports the AR.
- **Duration:** pass `durationSeconds` = the clip's share of `ai_seconds`. For a single hero beat that's
  the whole `ai_seconds` (cap per the model's max; split into multiple clips if longer). The end card
  covers `end_seconds` in HyperFrames — never pad the AI clip to fake total length.
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
- Set the HyperFrames canvas to the **confirmed AR**: `16:9`→1920×1080, `9:16`→1080×1920, `1:1`→1080×1080,
  `4:5`→1080×1350. Root `data-duration` = `total_seconds`; clip track = `ai_seconds`; end card = `end_seconds`.
- BGM with a hit on the end-card reveal.
- Render via the HyperFrames pipeline / the app's render route. See `reference.md` for the calls.

### 7. Deliver

Deliver the final rendered video plus a one-line summary (frames used, **model that actually ran**,
credits spent). Offer to `schedule_post` it to a connected social account if the user wants.

## Additional resources

- Exact MCP calls, model IDs, fallbacks, and rendering options: [reference.md](reference.md)
- Branded outro template: [assets/end-screen.html](assets/end-screen.html)

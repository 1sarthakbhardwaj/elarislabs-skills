# static-to-video — reference

Exact MCP tool calls, model IDs, fallbacks, and rendering options. Server: **ElarisLabs MCP**
(`user-elarislabs` in Cursor / `elarislabs` via `https://studio.elarislabs.ai/api/mcp`).

## Getting a local file into storage (do this FIRST for any local file)

Every generation tool needs a **public https URL**; the server can't read your disk. Pick the path that
matches your client — **never conclude you're blocked, and never refuse because base64 "looks too big."**

| Your client | Use | How |
|-------------|-----|-----|
| **MCP-only** (Claude Desktop / web — no shell, no filesystem) | **`upload_asset`** | If the user **attached the image**, pass that attachment as `source`. The client supplies the bytes — you do **not** transcribe base64. Returns `{ url }`. This is the reliable MCP-only path. |
| Image already lives at a URL you can reach | `upload_asset` | `source: "<url>"`, `rehost: true` → server fetches + re-hosts, returns a stable `{ url }`. |
| **Client with a shell** (Cursor, local agent) | **`create_upload_url`** → `relay` | POST the raw bytes to the relay endpoint on the MCP host (see below). |

**`upload_asset` (MCP-only / attachments):**
```
upload_asset
  source: <the attached image>     # data URL / base64 — client fills bytes; DO NOT retype
  kind:   "image"
# → { url }  ← use as referenceImages / imageUrl
```

**`create_upload_url` → relay (shell clients):** returns `{ relay, direct }`.
```
# relay (preferred — hits the allowlisted MCP host, server forwards to storage):
curl -X POST 'https://studio.elarislabs.ai/api/mcp/upload?filename=lumina.png' \
  -H 'Authorization: Bearer <YOUR_ELX_KEY>' -H 'Content-Type: image/png' \
  --data-binary @/path/to/lumina.png
# → { file_url }   (direct = fal presigned PUT; only if you can reach v3b.fal.media)
```

> All upload tools are **free**. If none of the paths work from your environment, ask the user to upload
> the file in Creative Studio and paste the asset URL — then pass it straight into `referenceImages` /
> `imageUrl`. Do not spend generation credits inventing the asset from scratch.

## Model IDs (grounded to the repo)

### Image (`generate_image` accepts a `model`)
| Model | `model` id | Use |
|-------|-----------|-----|
| Nano Banana 2 | `nano-banana-2` | Aspect reframe (source AR ≠ target) + default on-brand stills / product consistency |
| GPT Image 2 | `gpt-image-2` | Text-in-image, precise multi-ref edits, premium fidelity |
| Flux 2 Flash | `flux-2-flash` | Fast/cheap drafts |
| Seedream v5 Lite | `seedream-v5-lite` | Cost-effective alternative |

### Video (image-to-video) — pass as the `generate_video` `model` arg
IDs are the `list_models` ids (verified live on the ElarisLabs MCP). **Always pass `model`.**

| Priority | Model | `model` id | Route |
|----------|-------|-----------|-------|
| **1 primary** | Seedance 2.0 | `seedance2-std-i2v` | **Default** — no human/character in frame |
| 1 (cheaper) | Seedance 2.0 Fast | `seedance2fast-fast-i2v` | Faster/cheaper draft of the default |
| **2 fallback** | Kling O3 Pro | `kling-o3-pro-i2v` | **Human / character in frame** (decide upfront) |
| 2 (alt) | Kling O3 | `kling-o3-std-i2v` | Human, standard tier |
| **3 last resort** | Veo 3.1 | `veo31-std-i2v` | Only if Seedance + Kling both fail. Never a default. |

> The MCP `generate_video` / `regenerate_video` tools **now accept a `model` argument** (shipped in
> `lib/mcp/tools/{registry,executors}.ts`). The executor maps the id to the exact fal endpoint, applies
> the right duration format, and returns `modelUsed` so you can verify which model ran.

#### Aspect-ratio support (pick a model that fits the requested AR)

The model **must** support the target AR — never change the user's AR to fit a model. If the priority
model can't do the AR, drop to the next model that can.

| Model | 16:9 | 9:16 | 1:1 | 4:5 |
|-------|:----:|:----:|:---:|:---:|
| Seedance 2.0 (`seedance2-std-i2v`) | ✓ | ✓ | ✓ | ✗ |
| Kling O3 Pro (`kling-o3-pro-i2v`)  | ✓ | ✓ | ✓ | ✗ |
| Veo 3.1 (`veo31-std-i2v`)          | ✓ | ✓ | **✗** | ✗ |

- **1:1 requested → Veo 3.1 is NOT a valid fallback** (it can't do square). Use Seedance 2.0, or Kling O3
  Pro for people.
- **4:5 requested → no i2v model does it natively.** Animate in the nearest supported AR (usually 9:16),
  then set the **HyperFrames canvas to 1080×1350** and fit the clip (the reframe/outpaint step can also
  build a true 4:5 master still). Confirm the approach with the user.
- Always verify the model's `aspectRatios` via `list_models` before committing.

## Tool calls

### Check budget (free)
```
get_credits
```

### Aspect reframe (when source AR ≠ target AR) — do this before stills
Skip if source already matches `aspect_ratio` in the look spec.

```
generate_image
  prompt:          "Recompose this ad into a clean <TARGET_RATIO> canvas. Keep the same product,
                    props, lighting, palette, and typography style. Outpaint / extend the scene
                    naturally to fill the new frame — do not stretch, squash, or letterbox.
                    Preserve brand hierarchy and leave readable negative space where copy lives."
  model:           "nano-banana-2"
  referenceImages: ["<original reference url or data URL>"]
  resolution:      "2K"
```

Use the returned URL as the **master reference** for all subsequent stills.

Detect source AR from image dimensions (e.g. 1024×1024 → `1:1`, 1080×1920 → `9:16`).
Normalize common targets: `1:1`, `9:16`, `16:9`, `4:5`.

### Generate a still
```
generate_image
  prompt:          "<frame sentence from the look spec, on-brand>"
  model:           "nano-banana-2"          # or "gpt-image-2"
  referenceImages: ["<master reference url>", "<brand logo url if it should appear>"]
  resolution:      "2K"
  quality:         "high"                    # gpt-image models only
```

### Animate a still (pin the model)
```
generate_video
  prompt:          "frame-by-frame stop-motion of <subject>, slight pose shifts, subtle
                    handheld jitter, ~12fps cadence, hold subject stable"
  imageUrl:        "<approved still url>"
  durationSeconds: 3
  model:           "seedance2-std-i2v"   # <- ALWAYS pass. Kling: kling-o3-pro-i2v. Veo: veo31-std-i2v.
```
- After the call, check the returned `modelUsed` (or `get_generation_history`) to confirm the right model
  ran. If it fell back to something else, `regenerate_video` with the correct `model`.
- `regenerate_video` takes the same `model` arg plus a `variationHint` to reroll a clip.

Routing logic (decide **upfront** from `has_human` + the confirmed AR):
```
# 1) pick by content
base = kling-o3-pro-i2v  if has_human/character in frame  else seedance2-std-i2v
# 2) enforce AR support (must fit the user's AR; do NOT change the AR)
if base does not support target_AR:  base = first model that fits (has_human? kling : seedance)
# 3) last resort only if both above fail — and only if it supports the AR
fallback = veo31-std-i2v   only if target_AR in {16:9, 9:16}   (Veo has no 1:1)
```
Duration: `durationSeconds` = this clip's share of `ai_seconds` (= total − ~3s end card). Never pad the
AI clip to fake the total; the end card fills the remainder in HyperFrames.

### Stitch frames only (no per-frame animation)
For a pure timelapse of stills (not stop-motion), `create_hyperlapse` stitches image URLs locally:
```
create_hyperlapse
  images: ["<url1>", "<url2>", ...]   # ordered
  fps:    8                           # low fps = stop-motion feel
```

## Assemble in HyperFrames (Asian Paints style)

Assembly happens in **HyperFrames**, not by chaining another video model. Bring the Seedance clip in as a
track, add a **branded end card**, and layer **BGM** (no VO unless asked). Mirror the Asian Paints /
product-launch build.

- **End card treatment (Asian Paints):** dark ground, brand color as the single accent, Inter, a short
  **kinetic-type barrage** resolving on the **wordmark + CTA pill**. The template
  `assets/end-screen.html` (`{{PLACEHOLDER}}` tokens) is the fallback source for brand tokens.
- **Logo is required on the end card.** Priority: (1) user-supplied logo; (2) crop the mark/seal from the
  master reference and mask its background to transparent — a solid-color disc/badge masks cleanly with a
  circular alpha; (3) if the crop is poor, regenerate a transparent logo with `generate_image`
  `model: nano-banana-2` passing the cropped mark as `referenceImages`. Copy into `assets/`, place as an
  `<img>` hero above the eyebrow, animate in first (`back.out` spring).
- **Canvas:** target AR — `16:9` → 1920×1080, `9:16` → 1080×1920, `1:1` → 1080×1080. Put the BGM hit on
  the end-card reveal.
- **Render:** the app renders HTML → video via `lib/hyperframes/pipeline.ts` +
  `app/api/studio/hyperframes/render/route.ts`. Or use the local HyperFrames CLI project
  (`npx hyperframes init` → build frames → `render`) as done for the Asian Paints / timeline-editor promo.

## Delivery

- Deliver the final HyperFrames render.
- Summarize: frames used, **video model that actually ran** (`modelUsed` / `get_generation_history`),
  image model, total credits.
- Optional: `list_scheduler_integrations` → `schedule_post` to publish.

## Cost discipline

- Stills are cheap; video is the expensive step. Confirm the approved set and estimate before animating.
- Prefer `seedance2fast-fast-i2v` / `flux-2-flash` for drafts, upgrade to full models for the final pass.

## Proven recipe (verified end-to-end)

This exact flow was run and validated (air-fryer "Healthy Made Simple", 16:9, no humans):

1. **Reframe 1:1 → 16:9** — `generate_image` `model: nano-banana-2`, original as `referenceImages`,
   `resolution: 2K`. Output = master reference (~15 credits).
2. **Animate — Seedance 2.0** — `generate_video` `model: "seedance2-std-i2v"`, `imageUrl` = master
   reference, `durationSeconds: 5`. Confirm the result's **`modelUsed` == `seedance2-std-i2v`** (proves it
   didn't fall back to Veo). ~42 credits for 5s.
3. **BGM** — the MCP `generate_audio` is **TTS, not music**. For instrumental BGM use the ElevenLabs MCP
   `compose_music` (`force_instrumental: true`, `music_length_ms`), not `generate_audio`.
4. **Assemble in HyperFrames (local CLI):**
   - `npx hyperframes init videos/<name> --non-interactive --example=blank`
   - Copy the Seedance mp4 + BGM into `assets/`.
   - Author `index.html`: `<video class="clip" data-track-index="0">` for the product beat, an inline
     `<section class="clip" data-track-index="1">` end card (kinetic verbs → wordmark → CTA pill),
     `<audio data-track-index="10">` for BGM. **Media must be a direct child of the root**; one paused
     GSAP timeline at `window.__timelines["main"]`; animate the end card at **global time** (local +
     the section's `data-start`). Mark the blurred aurora `data-layout-allow-overflow`.
   - `npx hyperframes check` → `snapshot --at <midpoints>` (video frames read black in snapshots; that's
     expected — they composite in the real render) → `render -q high`.
   - Output = final MP4 with the Seedance footage composited + branded end card + BGM.

Working reference project: `videos/airfryer-healthy-made-simple/` in the ELARIS-AI repo.

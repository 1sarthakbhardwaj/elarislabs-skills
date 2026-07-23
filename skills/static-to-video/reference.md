# static-to-video — reference

Exact MCP tool calls, model IDs, fallbacks, and rendering options. Server: **ElarisLabs MCP**
(`user-elarislabs` in Cursor / `elarislabs` via `https://studio.elarislabs.ai/api/mcp`).

## Model IDs (grounded to the repo)

### Image (`generate_image` accepts a `model`)
| Model | `model` id | Use |
|-------|-----------|-----|
| Nano Banana 2 | `nano-banana-2` | Aspect reframe (source AR ≠ target) + default on-brand stills / product consistency |
| GPT Image 2 | `gpt-image-2` | Text-in-image, precise multi-ref edits, premium fidelity |
| Flux 2 Flash | `flux-2-flash` | Fast/cheap drafts |
| Seedream v5 Lite | `seedream-v5-lite` | Cost-effective alternative |

### Video (image-to-video)
| Model | id | Route |
|-------|----|-------|
| Seedance 2.0 | `seedance-2` | **Default** — no human in frame |
| Seedance 2.0 Fast | `seedance-2-fast` | Faster/cheaper default |
| Kling 3.0 | `kling-v3` | **Human in frame** |
| Kling O3 | `kling-o3` | Human, alt |
| Gemini Omni Flash | `gemini-omni-flash` | **Editing** an existing clip/frame |
| Veo 3.1 | `veo3.1` | Extra fallback if Seedance/Kling unavailable |

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

### Animate a still (MCP — auto-routes, no model arg)
```
generate_video
  prompt:          "frame-by-frame stop-motion of <subject>, slight pose shifts, subtle
                    handheld jitter, ~12fps cadence, hold subject stable"
  imageUrl:        "<approved still url>"
  durationSeconds: 3
```
- `regenerate_video` with a `variationHint` to reroll a clip you don't like.
- The MCP tool does **not** take a `model`. For explicit Seedance/Kling/Gemini routing use option B.

### Option B — pin the video model
The MCP surface auto-routes. To force a specific model, drive the Studio instead:
- Open the project (`get_project`) and add/animate via the **Studio video node**, choosing the model id
  from the table above, **or**
- Use the internal video-studio router (`lib/video-studio/router.ts` +
  `lib/models/studio-video-models.ts`) where the fal endpoint per model is explicit, e.g.
  `fal-ai/bytedance/seedance/...`, `fal-ai/kling-video/...`.

Routing logic to apply (from the look spec `has_human` flag):
```
if editing an existing clip/frame      -> gemini-omni-flash
elif has_human == true                 -> kling-v3   (fallback kling-o3)
else                                   -> seedance-2 (fast: seedance-2-fast)
if the chosen model is unavailable     -> veo3.1
```

### Stitch frames only (no per-frame animation)
For a pure timelapse of stills (not stop-motion), `create_hyperlapse` stitches image URLs locally:
```
create_hyperlapse
  images: ["<url1>", "<url2>", ...]   # ordered
  fps:    8                           # low fps = stop-motion feel
```

## Branded end screen — rendering options

The template `assets/end-screen.html` uses `{{PLACEHOLDER}}` tokens. Fill them, then render:

1. **HyperFrames pipeline (in-repo):** the app already renders HTML → video
   (`lib/hyperframes/pipeline.ts`, `app/api/studio/hyperframes/render/route.ts`). Feed the filled HTML
   as a composition and export a 2–3s clip at the video's aspect ratio.
2. **Composer overlay:** generate a plain background still, then `composer_edit` `add_text` to stamp the
   brand name/CTA, and overlay the logo — cheap, no render server.
3. **Static hold:** screenshot the filled HTML (e.g. Playwright) and hold the frame for 2–3s in assembly.

Match the render dimensions to the clips: `9:16` → 1080×1920, `16:9` → 1920×1080, `1:1` → 1080×1080.

## Delivery

- Concatenate approved clips in look-spec `frames` order, append the end-screen clip.
- Summarize: frames used, image + video models used, total credits (`get_generation_history`).
- Optional: `list_scheduler_integrations` → `schedule_post` to publish.

## Cost discipline

- Stills are cheap; video is the expensive step. Confirm the approved set and estimate before animating.
- Prefer `seedance-2-fast` / `flux-2-flash` for drafts, upgrade to full models for the final pass.

# static-to-video — reference

ElarisLabs MCP (`user-elarislabs`). Seed-first: animate the user’s still; HyperFrames + BGM to finish.

## Models

### Image (sparingly)
| Model | id | When |
|-------|-----|------|
| Nano Banana 2 | `nano-banana-2` | Aspect reframe; optional product close-ups from seed |
| GPT Image 2 | `gpt-image-2` | Text-heavy one-off hero seed (only if no reference still) |
| Flux 2 Flash | `flux-2-flash` | Cheap drafts only |

### Video (I2V on the seed)
| Model | id | When |
|-------|-----|------|
| Seedance 2.0 | `seedance-2` / `seedance-2-fast` | Default, no human |
| Kling | `kling-v3` / `kling-o3` | Human in frame |
| Gemini Omni | `gemini-omni-flash` | Editing an existing clip |

## Tool calls

### Credits
```
get_credits
```

### Aspect reframe (only if seed AR ≠ target)
```
generate_image
  model: nano-banana-2
  referenceImages: ["<seed>"]
  prompt: "Recompose into <TARGET_RATIO>. Keep product, lighting, typography. Outpaint — do not stretch."
  resolution: 2K
```

### Optional close-up (0–2 max)
```
generate_image
  model: nano-banana-2
  referenceImages: ["<master seed>"]
  prompt: "Tight product/food close-up matching this ad; same lighting and materials."
```

### Animate a seed (core step)
```
generate_video
  imageUrl: "<seed or insert url>"
  durationSeconds: 4
  prompt: "Seedance-style stop-motion / subtle product motion from this still: snappy
           frame-by-frame feel ~12fps, hold product stable, gentle steam/light shift,
           no morphing brand text, no new objects, no humans."
```

MCP has **no `model` arg** on `generate_video` (auto-route). Pin Seedance/Kling via Studio when needed.

### BGM (no voiceover)
Use ElevenLabs MCP `compose_music`:
```
compose_music
  prompt: "warm upbeat instrumental kitchen/lifestyle bed, soft acoustic + light percussion, no vocals"
  force_instrumental: true
  music_length_ms: 15000
  output_directory: "<run tmp dir>"
```
Do **not** use `generate_audio` (TTS) for this skill.

## HyperFrames finish

Order: **first card → I2V clip(s) → end card**, under one BGM bed.

### First card
- 1:1 / 9:16 / 16:9 matching target.
- Headline + brand from look spec; sage/forest palette from seed if no brand kit.
- ~2s hold or light HyperFrames motion (fade / scale).

### End card
Fill [assets/end-screen.html](assets/end-screen.html) tokens. If no logo, text + seal only.

### Assembly
1. Preferred: HyperFrames project (`lib/hyperframes/pipeline.ts` +
   `app/api/studio/hyperframes/render/route.ts`) with clip URLs as media + BGM track.
2. Fallback: ffmpeg — concat card stills (tpad/loop) + mp4 clips, `-i bgm.mp3 -shortest`,
   AAC audio, H.264 video.

Dimensions: `1:1` → 1080×1080, `9:16` → 1080×1920, `16:9` → 1920×1080.

## Cost

- One seed animate ≈ main cost. Avoid still farms.
- Confirm seeds before `generate_video` / `compose_music`.

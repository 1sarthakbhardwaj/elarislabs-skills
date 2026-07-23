---
name: static-to-video
description: >-
  Animate a seed still (reference ad/product image) into a short branded video via ElarisLabs MCP:
  optional aspect reframe + optional close-ups from the seed, image-to-video (Seedance / Kling /
  Gemini Omni), then HyperFrames assembly with first card, end card, and instrumental BGM (no
  voiceover). Use when the user says static-to-video, animate this ad/still, stop-motion from a
  reference image, or finish a product still into a promo video.
disable-model-invocation: true
---

# Static → Video (seed-first)

The **reference image is the seed**. Do **not** regenerate a stack of alternate hero stills.
Animate the seed. Optionally add **one or two product close-ups** conditioned on that same seed.
Finish in **HyperFrames**: first card → motion clips → end card, with **instrumental BGM only**
(no voiceover).

## Golden rules

- **Seed first.** If the user already pasted an ad/still, that IS the master seed. Confirm it.
- **Do not invent 4–5 frame stills** as the core pipeline. Extra stills only when the user asks for
  close-ups / inserts, and always with the seed as `referenceImages`.
- **Ask which seed(s) to animate** before spending video credits (approval gate).
- **Video models animate the seed** — image-to-video. You are not “animating by generating more stills.”
- **No voiceover.** Use instrumental BGM. HyperFrames for first card, end card, and final stitch.
- **Brand kit** for cards (logo if available, name, colors). Fall back to palette from the seed.
- Check `get_credits` before video / music spend.

## Workflow

```
- [ ] 1. Intake — seed image (or brief) + brand kit + target AR
- [ ] 2. Look Spec — short motion plan (not a multi-still storyboard)
- [ ] 2b. Aspect reframe — ONLY if seed AR ≠ target AR → nano-banana-2 once
- [ ] 3. Optional inserts — 0–2 close-ups from seed (nano-banana-2 / gpt-image-2)
- [ ] 4. Approval — confirm which seed(s) to animate
- [ ] 5. Animate — generate_video on each approved seed (Seedance / Kling / Omni)
- [ ] 6. HyperFrames finish — first card + clips + end card + BGM (no VO)
- [ ] 7. Deliver — final URL + credit summary
```

### 1. Intake

- **Seed:** reference image URL / data URL. If only a brief, generate **one** hero seed
  (`gpt-image-2` if text-in-image matters, else `nano-banana-2`), then treat it as the seed.
- Brand: logo (ask), name, tagline/CTA, colors; else pull from seed.
- Target `aspect_ratio` (`1:1` | `9:16` | `16:9`).

### 2. Look Spec (editable, short)

```yaml
seed:           # "user reference" | url of generated hero
has_human:      # true | false
motion_feel:    # snappy | dreamy | mechanical  (stop-motion cadence)
aspect_ratio:   # 1:1 | 9:16 | 16:9
optional_inserts:  # 0–2 close-up beats, or []
  - product basket / food detail
motion_prompt:  # one paragraph for I2V on the seed
first_card:     # headline / CTA for open
end_card:       # brand + tagline
bgm:            # mood for instrumental bed (no lyrics)
```

### 2b. Aspect reframe (when AR differs)

If seed AR ≠ target → one `generate_image` with `nano-banana-2` + seed as reference → recompose /
outpaint to target. That output becomes the **master seed**. If ARs match → skip.

### 3. Optional inserts (not a still farm)

Only if useful (e.g. food/product macro). Max **0–2**. Always pass the master seed as
`referenceImages`. Prefer `nano-banana-2` for product lock; `gpt-image-2` if text must stay perfect.

### 4. Approval gate

Show seed (+ inserts). Ask: **which images should we animate?** Do not call `generate_video` until
confirmed (unless the user already said “animate the seed / run the full pipeline”).

### 5. Animate seeds (image-to-video)

For each approved image:

| Situation | Model |
|-----------|--------|
| No human | Seedance 2.0 (`seedance-2` / `seedance-2-fast`) |
| Human in frame | Kling (`kling-v3` / `kling-o3`) |
| Edit existing clip | Gemini Omni (`gemini-omni-flash`) |

MCP `generate_video` auto-routes (no `model` arg) — put stop-motion / Seedance intent in the prompt
and pass `imageUrl` = seed. Duration ~3–5s. See [reference.md](reference.md).

### 6. HyperFrames finish (required)

Assemble a polished piece — **not** a raw dump of clips:

1. **First card** (~2s) — brand / headline from look spec (template or HyperFrames HTML).
2. **Motion clips** — approved I2V results in order.
3. **End card** (~2–3s) — [assets/end-screen.html](assets/end-screen.html) (logo if given).
4. **BGM** — instrumental only (ElevenLabs `compose_music` or similar). **No TTS / voiceover.**
5. Mix BGM under the picture; soft duck under cards if needed.

Prefer the in-repo HyperFrames pipeline / render route when available; otherwise ffmpeg stitch of
card holds + clips + audio bed. Details in [reference.md](reference.md).

### 7. Deliver

Final video URL, list of seeds animated, models used, credits. Optional `schedule_post`.

## Anti-patterns

- Generating 4–6 “storyboard stills” that redraw the whole ad
- Treating still generation as the animation
- Adding voiceover by default
- Skipping first/end cards and shipping bare I2V clips

## Additional resources

- MCP calls, models, HyperFrames + BGM notes: [reference.md](reference.md)
- End card template: [assets/end-screen.html](assets/end-screen.html)

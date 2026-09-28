---
name: magnific-mcp
description: >
  Best-practices guide for using the Magnific MCP server inside Claude. Covers
  first-run setup (usage alerts, credit tracking, cost preferences), model
  selection with live credit costs, character/style consistency via the
  Library, image→video chaining, the video post-production toolbox (modify,
  extend, magic cut, relight, color grade, VFX, music/SFX, lip sync, dubbing),
  image editing tools, 3D, audio, budget management and approval rules for
  expensive operations. Use whenever the user wants to generate or edit
  images, videos, 3D models, audio or upscale media through Magnific —
  including phrases like "generate an image", "create a video", "Magnific",
  "Nano Banana", "Seedream", "Seedance", "Kling", "Veo", "GPT Image",
  "upscale this", "lip sync", "dub this video", "color grade", "make a
  character", "magnific library", "how many credits", or any request to
  produce visual/audio media through Magnific's MCP tools.
---

# Magnific MCP — Best Practices

This skill packages accumulated knowledge for using the Magnific MCP server
effectively. It assumes you have an active Magnific subscription and have
connected the Magnific MCP to your Claude client.

> **Catalog snapshot: late September 2026.** Magnific adds and rotates models
> every few weeks. The model lists and prices below are a starting point —
> the live catalogs (`*_models_list`) and `simulate_cost` are always the
> source of truth.

---

## 0. Detect which Magnific MCP is connected

The Magnific MCP can be registered under different names — common ones:

- `magnific`
- `magnific-personal`
- `magnific-work`

Use whichever is available. The tool names are identical across all of them
(`mcp__<server>__images_generate`, etc.) — only the server prefix changes.

If multiple are registered, ask the user once which to use as the default,
then store the choice in the skill config (below).

**Follow the `instruction` field.** Almost every tool response carries an
`instruction` telling you what to call next (usually `creations_show`, then
`creations_wait`). Follow it — it reflects the server's current contract
better than any static guide.

---

## 1. First-run setup — ALWAYS run this before generating

On the **first** invocation in any project, check whether
`~/.claude/skills/magnific-mcp/user_config.json` exists. If not, ask the user
the three setup questions below, then write the config. Do **not** skip this
step on the first run — the answers shape every subsequent decision.

### Question 1 — Usage alerts

> "Want me to warn you when monthly spend crosses a threshold? If yes, at what
> credit amount or percentage?"

Options:
- `none` — no alerts (just go).
- `percentage` — alert at e.g. 75% of monthly budget.
- `absolute` — alert when total spent crosses N credits.

### Question 2 — Credit tracking

> "Should I track credit spend across this skill, or do you prefer to check
> your balance manually?"

Options:
- `auto` — after every generation, call `account_balance` and update
  `budget.json`, log model + cost + timestamp.
- `manual` — never auto-check; user runs `account_balance` themselves.
- `summary-only` — log spend locally but don't hit the API after each call.

### Question 3 — Cost preference

> "How aggressive should I be with credits?"

| Mode | Behaviour |
|------|-----------|
| `quality-first` | Use top models freely. Don't ask before any spend. |
| `balanced` | Use sensible defaults. Confirm before single ops > 500 credits. |
| `careful` | Cheap models by default. Confirm before any spend > 100 credits. Always ask before video/3D. |

Persist the answers as JSON:

```json
{
  "mcp_name": "magnific",
  "alerts": { "mode": "percentage", "threshold": 75, "monthly_budget": 35000 },
  "tracking": "auto",
  "cost_mode": "balanced",
  "created": "YYYY-MM-DD"
}
```

After first-run setup, surface the three settings in one line at the start of
each new session — e.g. *"Magnific: balanced mode, alerts at 75%, tracking
on. Change anytime."*

---

## 2. Know the price before you spend — `simulate_cost`

`simulate_cost({ tool, arguments })` returns the **exact** credit cost of a
call without running it. It's read-only and never charges. Pass the same
arguments you're about to send to the real tool (`images_generate`,
`video_generate`, `video_modify`, `video_speak`, `models3d_*`, `audio_*`,
and most other paid tools). Pass `arguments: {}` to get the tool's
required/optional args back.

Use it:
- before any call your `cost_mode` says needs confirmation (§8),
- when comparing two candidate models for the user,
- whenever resolution / quality / duration changes — cost scales steeply
  with those, not just with the model.

`account_balance` now also returns the plan (`tier`, `isUnlimitedMode`,
`unlimitedAppliesHere`). If unlimited mode applies to the tool you're using,
`simulate_cost` reports `isUnlimited: true` — no need to ask about cost.
Don't assume a model is unlimited from memory: Magnific has been moving
individual models (e.g. Nano Banana, Flux.2, older GPT Image) from unlimited
to metered on some plans. Trust the live `isUnlimited` flag.

---

## 3. Image models

> **Live-truth principle**: run `images_models_list({ onlyRecommended: true })`
> before any non-trivial job. Prefer `agentRecommendation.tier: sota`, then
> the lowest `rank`. Entries flagged `private: true` are feature-flagged and
> not generally available — don't pick them.

### Recommended set (Sept 2026) — costs from `simulate_cost`

| Slug | Display name | Best for | Credits (1 image) | Notes |
|------|-------------|----------|-------------------|-------|
| `seedream-5-pro` | Seedream 5 Pro | **Magnific's new default.** Text→image, reference-guided edits, product, character, photoreal, general purpose | ~100 | 1.5k / 2k only; all 4 ref types |
| `recraft-v4-1` | Recraft V4.1 | **Pure text→image, first drafts**, illustration, typography, fast exploration (~14 s) | ~60 (`tier: mid`) / ~175 (`tier: pro`) | `style` refs only; `variant: standard \| utility` |
| `gpt-2-mini` | **GPT 2.5** ⚠️ | Text, infographics, UI, diagrams, layout. **Best Hebrew / Arabic / RTL.** Faster than GPT 2 (~31 s) | **~15 (low, 1k)** · ~100 (medium) · ~1,000 (max, 4k) | Qualities `low→max`; `variant: flare \| sunburst`; transparent BG |
| `gpt-2` | GPT 2 | Same use case as GPT 2.5, older/slower (~59 s) | ~15 (low, 1k) · ~100 (medium) · ~1,000 (high, 4k) | Transparent BG |
| `imagen-nano-banana-2-lite` | Nano Banana 2 Lite | **Cheap/fast drafts**, high-volume iteration (~9 s) | ~60 | New |
| `imagen-nano-banana-2-flash` | Nano Banana 2 | Edits / references when Pro fidelity isn't required | ~75 | Extreme ratios: 8:1, 4:1, 1:4, 1:8 |
| `imagen-nano-banana-2` | Nano Banana **Pro** | **Max fidelity / brand accuracy** on a final asset | ~75 (2k) · ~150 (4k) | Use only when fidelity is the point |

**Naming traps** (these catch everyone):
- `gpt-2-mini` is displayed as **"GPT 2.5"** — it's the newer model, not a
  smaller one.
- `imagen-nano-banana-2` = Nano Banana **Pro**. `imagen-nano-banana-2-flash`
  = Nano Banana 2.

**Quality/resolution drive cost more than model choice.** GPT 2.5 goes from
~15 credits (low, 1k) to ~1,000 (max, 4k) — a 66× spread on the same model.
Draft at low/1k, then re-run the winner at high quality.

### Decision tree (in this order)

1. **Output must contain rendered text** (any language, especially Hebrew /
   Arabic / RTL / Asian scripts) → `gpt-2-mini` (GPT 2.5). Draft at
   `quality: low`, finalize at `high`.
2. **Needs a transparent background** → `gpt-2-mini` / `gpt-2` with
   `transparentBackground: true`. Otherwise chain `images_remove_background`.
3. **Reference-guided work** (character, product, "make it look like this")
   → `seedream-5-pro`. Escalate to Nano Banana Pro only if the user needs
   maximum fidelity / brand accuracy.
4. **First-shot creation, no references** → `recraft-v4-1` (fastest) or
   `seedream-5-pro`.
5. **Many quick drafts / tight budget** → `imagen-nano-banana-2-lite`, or
   GPT 2.5 at `quality: low`.
6. When you switch model for a specific reason, say it briefly:
   *"Using GPT 2.5 because the prompt has Hebrew text."*

### The wider catalog

`images_models_list()` without filters returns ~50 models: the Flux.2 family
(Klein / Pro / Flex / Max), Ideogram 4, Luma Uni-1.1, Seedream 4/4.5/5
Flash/5 Lite, Qwen, Grok, Runway, Mystic 2.5, Z-Image, a `cinematic` mode
with camera controls (`images_models_settings({ mode: "cinematic" })`) and
more. They're valid, just not preferred — use one when the user names it.

### Useful `images_generate` parameters

- `count: 1..8` — N variants of one prompt = **one** call with `count: N`,
  never N separate calls.
- `seed` — same seed + same settings reproduce a result.
- `quality`, `resolution`, `tier`, `variant` — per model, see the catalog.
- `transparentBackground` — only on models that flag it.
- `@name` in the prompt — mention a Library character/product/location by
  its `library_list` name where it appears in the scene; the server rewrites
  the token (the entry must also be in `references[]`).
- `brandKitId` / `channel` / `templateSlug` — on-brand generation when the
  brand-kit tools are exposed to your client. If they're not in your tool
  list, ignore these.

---

## 4. Reference images — HARD rules

These are **stop-and-think** rules. Wrong handling here wastes credits and
breaks the user's intent.

### Rule 4.1 — If the user provides a reference image, you MUST use it

When the user shares an image (file path, URL, screenshot, inline upload,
or a previously generated creation) and asks for a related generation,
**that image must end up in `references[]` of the generation call** — either
as a creation identifier (upload it first via `creations_upload_image` /
`creations_request_upload`), or by routing through a Library entry.

Do not paraphrase the reference in words and hope for the best. Do not
silently drop it because the model "should know" what the user wants. The
reference is the user's specification.

### Rule 4.2 — If you can't access the reference, STOP and ask for access

If a reference is a local file you can't read, a private URL, or a creation
under a different account/MCP, **do not start the work**. Ask the user for
the file/URL/access first.

- *"I can't read the file you mentioned at `<path>`. Can you upload it or
  paste a public URL?"*
- *"That creation belongs to a Magnific account I'm not connected to —
  switch the MCP or share the asset URL."*

**Never** proceed with a generation that ignores the reference. Wasting
credits on a fallback is worse than waiting.

### Rule 4.3 — Reference types

`images_generate.references[]` takes up to 12 entries:

| Type | What to pass | Effect |
|------|-------------|--------|
| `image` | creation identifier | Any photo — incl. a person/product to keep consistent |
| `style` | creation identifier (style photo) **or** Library style LoRA id | Palette, texture, vibe |
| `character` | Library numeric `id` | Same person across generations |
| `product` | Library numeric `id` | Same branded object |
| `locations` | Library numeric `id` | Same place/set |

`character` / `product` / `locations` are **Library assets, never raw
creations** — for a one-off photo of a person, use `type: image`. Check
each model's `referenceTypes` in the catalog (Recraft takes `style` only).

### Rule 4.4 — Pick a representative subset

If the user hands over a folder of references, don't upload everything.
Pick the cover/key image plus 4–5 that span the range of style/colour.
More references rarely improve the result and cost upload time.

---

## 5. Video models

**Always start with `video_plan`** for any video request — it returns a
brief, open questions, the recommended slug and a prompt draft, and splits
anything longer than the model's max into clips + `video_concatenate`
instructions. Then `simulate_cost` before `video_generate`.

### Recommended set (Sept 2026) — costs for a 5 s, 16:9 clip

| Slug | Name | Best for | Durations | Resolutions | Audio | ~Credits (5 s) |
|------|------|---------|-----------|-------------|-------|---------------|
| `bytedance-seedance-pro-2.5` | **Seedance 2.5** | **Best overall.** Realism, complex motion, native audio + lipsync, directed camera (52 moves), multishot (6), up to 30 image / 10 video / 10 audio refs | **4–30 s** | Draft / 480p / 720p / 1080p | ✅ | Draft ~1,000 · 720p ~2,200 · 1080p ~3,950 |
| `bytedance-seedance-pro-2.0` | Seedance 2.0 | 4K output with Seedance controls | 4–15 s | 480p → **4K** | ✅ | 1080p ~3,500 |
| `bytedance-seedance-fast-2.0` | Seedance 2.0 Fast | Seedance controls/audio at draft quality | 4–15 s | 480p / 720p | ✅ | 720p ~1,175 |
| `kling-omni3` | Kling 3.0 Omni | **Good-value clip with sound**, up to 15 s, 4K option | 3–15 s | 720p / 1080p / 4K | ✅ | 1080p ~475 |
| `kling-25` | Kling 2.5 | **Cheapest quality silent clip**, start/end frames | 5 / 10 s | 720p / 1080p | ❌ | 1080p ~325 |
| `minimax-video-3_0-max` | MiniMax H3 Max | Fast drafts **with** image/video/audio refs, multishot | 5–15 s | 480p / 768p | ❌ | 768p ~400 |
| `minimax-video-3_0-max-turbo` | MiniMax H3 Max Turbo | **Cheapest draft** from text or start/end frames | 5–15 s | 480p / 768p | ❌ | 768p ~200 |

Also in the full catalog (`video_models_list()`, ~45 models): **Google Veo
3.1** / Fast / Lite (4–8 s, up to 4K, audio; Fast 8 s 1080p ≈ 800),
**Kling 3.0** / Turbo / Motion Control, **Gemini Omni 1.1** (3–10 s, up to
4K), **Wan 2.5 / 2.6 / 2.7**, **PixVerse 5.5 / 6**, **Grok Imagine 1.5**,
**Happy Horse 1 / 1.1 / Edit**, Runway Gen 4 / 4.5 / Act Two, Hailuo 2.3,
Wan 2.2 Animate. `private: true` entries (e.g. Wan 3.0, FLUX 3 video,
Seedance 2.0 Mini) are not generally available yet.

### Default video ladder

1. **Draft / explore motion** → `minimax-video-3_0-max-turbo` (~200) or
   `kling-25` 720p. Use H3 Max if the draft needs references.
2. **Final, needs sound** → `kling-omni3` (~475 for 5 s 1080p).
3. **Final, top quality / lipsync / long takes (>15 s) / heavy references**
   → `bytedance-seedance-pro-2.5` — **always confirm cost first**; it's
   5–10× the Kling price. Try `resolution: "Draft"` to validate staging.

Switch to Seedance when the brief mentions dialogue/lipsync, multishot,
directed camera moves, >15 s in one take, or audio references.

### Video gotchas

- **Keyframes vs references are exclusive.** Start/end frames go in
  `video_generate.keyframes.{start,end}` (`{ type: "image", url }`), and
  most models reject `references[]` in the same call. Choose: "animate this
  exact frame" (keyframes) or "use these as guidance" (references).
- **Audio references** go in `references[]` with `type: "audio"` — never
  `audioUrl`. On Seedance 2.0 / MiniMax an audio ref also needs a visual ref;
  Seedance 2.5 drops that requirement.
- **Aspect ratio auto-follows the start frame** on most models when one is
  provided.
- **Resolution × duration is the cost driver.** A 10 s Seedance 2.5 at
  1080p is ~8,000 credits. Check with `simulate_cost` every time either
  changes.
- **Chaining**: pass the creation `identifier` from the previous tool (or
  the `url` from `creations_get` / `creations_wait`) — never `webUrl`.

---

## 6. Post-production toolbox (existing footage and images)

Magnific now edits media, not just generates it. Before generating from
scratch, check whether one of these does the job on an existing asset.
All work on an owned **creation identifier** — upload external files first.

### Video editing

| Need | Tool | Notes |
|------|------|-------|
| Change content, keep the motion ("make it night", "replace the car") | `video_modify` | Models via `video_modify_models_list`: Seedance 2.5 / 2.0, Gemini Omni 1.1 (up to 4K), MiniMax H3 (2K), Runway Aleph 2 (can re-frame with `targetAspectRatio`), Grok. Length comes from the source. |
| Continue a clip past its end | `video_extend` | `durationSeconds` of new footage; optional prompt; Seedance 2.5 / Gemini Omni extenders |
| Join clips seamlessly | `video_magic_cut` | **Generates** blend frames at each join; `zones` cut parts out. Paid. |
| Join clips as a plain cut | `video_concatenate` | 2–10 clips. Butt-joint, no generated frames. |
| Same shot, different camera position | `video_camera_angle` | `rotate` 0–360, `vertical` −30…90, `closeup` 0–10. Regenerates — not a crop (that's `video_crop`). |
| Relight footage | `video_relight` | 1–4 lights or copy lighting from an image. Max 240 frames. |
| Colour grade | `video_color_grade` | LUT from `video_luts_list` and/or manual adjustments. Max 20 s. |
| Match the look of a still | `video_color_transfer` | Reference image → palette. Max 20 s. |
| Effects | `video_vfx` | Effect id from `video_vfx_list`. Max 20 s. |
| Handheld shake | `video_motion_shake` | Per-segment style (`organic`/`urgent`/`impact`) and intensity |
| Score with music | `video_music` | `mixMode`: keep / duck / remove original audio. 3–300 s. |
| Auto sound effects | `video_soundfx` | Reads the video; prompt optional |
| Upscale | `video_upscale` | Modes: `topaz` (+ frame interpolation / FPS boost), `magnific` (creative), `magnific_precision`. Read `video_upscale_models_list` first. |
| SDR → HDR | `video_hdr` | Topaz Hyperion. **Not** an upscale. |
| Trim / crop / speed / zoom / remove BG | `video_cut`, `video_crop`, `video_speed`, `video_zoom`, `video_remove_background` | |
| Extract frames or audio | `video_extract_frames`, `video_extract_audio`, `video_audio_isolate` | |
| QA a generated clip | `video_analyze` | **Free, read-only.** Ask about artifacts, continuity, counting; compare up to 10 clips; `mode: "shot_breakdown"` for pre-vis. Use it to check a result before showing it or before paying to fix it. |

### Talking heads, lip sync, dubbing

- **`video_speak`** — make a character talk. Pick from
  `video_speak_models_list`:
  - Image source: `veed-fabric-1.0` (default) / `-fast` (audio up to
    **300 s**), `omnihuman` (takes a prompt; best ≤ 15 s audio).
  - Video source: `lipsync-2.0` (default, ≤ 30 s), `veed-sync-2-v2v`
    (≤ 300 s), `latentsync`.
  - Audio comes from `audio_tts` or an upload.
- **`video_dubbing`** — translate/dub/subtitle a talking video (≤ 120 s).
  Opens an inline studio; the transcription preview is free and **nothing
  is charged until the user confirms**. Headless clients use
  `video_dubbing_preview` → `video_dubbing_preview_get` →
  `video_dubbing_confirm`. Target languages: en, es, pt, ar, ru, fr, de,
  ko, tr, it, id, ja, zh (check the enum for additions — e.g. Hebrew is
  not on the list as of this snapshot).

### Image editing

| Need | Tool |
|------|------|
| Edit one region, keep the rest | `images_retouch` (mask: white = change). Models in `retouch_models_list` — omit for Auto. `erase` mode removes objects. |
| Extend canvas / change ratio keeping content | `images_expand` |
| Relight a photo | `images_relight` (1–4 lights, neutral or coloured gel) |
| Same subject, new camera angle | `images_change_camera` |
| Skin retouch | `images_skin_enhancer` (`faithful` keeps identity) |
| Upscale | `images_upscale` — read `images_upscale_modes_list` first |
| Cut a grid / character sheet / storyboard into tiles | `images_split_grid` — **free** |
| Vector from text | `images_generate_svg` (Recraft V4 Pro Vector) |
| Trace raster → vector | `images_to_svg` |
| Animate an SVG (CSS/JS, optional hover/click presets) | `images_animate_svg` — billed by tokens after generation |
| PBR material maps | `images_pbr_maps` (basecolor/normal/roughness/metalness/height) |
| Depth map / segmentation / BG remove or replace | `images_depth_map`, `images_segment`, `images_remove_background`, `images_replace_background` |
| Restyle / reimagine / variations | `images_restyle`, `images_reimagine`, `images_variations` |
| Flat image → editable layered design | `design_auto_layers` |
| One design → many social sizes | `design_auto_resize` |

**Rule**: `images_generate` is **not** pixel-preserving. If the user wants
the rest of the image untouched, use `images_retouch` / `images_expand` /
`images_upscale` / `images_crop` instead of regenerating.

### Storyboard → shots pattern

Generate a storyboard or character sheet as **one grid image** (one call),
`images_split_grid` it into tiles for free, then use the tiles as keyframes
or references. Cheaper and more consistent than N separate generations.

---

## 7. Audio and 3D

### Audio

- `audio_tts` — voiceover. Models: `eleven_v3` (default), `eleven_turbo_v2_5`,
  `gemini_v2_5_pro`, `gemini_v3_1_flash_tts`, `seed_audio_1_0` (voiceless,
  styled from reference audio). Two-speaker dialogue via `speakers`.
  **Call `audio_tts_direction` before writing the text** — tags, pauses and
  scene notation differ per model and are passed through verbatim.
- `audio_voices_list` / `audio_voices_show` — pick a voice first.
- `audio_voice_change` — re-speak a recording in another voice, keeping
  the timing and emotion.
- `audio_music_generate`, `audio_sfx_generate` — standalone tracks.
- `audio_isolate`, `audio_clean_speech` — clean up recorded audio.

### 3D

- `models3d_generate` (image → GLB) — models: `tripo-p1` (default),
  `tripo-p2`, `tripo-v31` (up to 2M faces, detailed textures), `trellis-2`,
  `meshy-6`, `meshy-7.1`, `meshy-t2` (clean topology, separated parts).
  Multiview input via `views` (front/left/back/right).
- `models3d_text_generate` — text → 3D.
- Animation chain: `models3d_rig` (**provider `meshy`** if you'll animate)
  → `models3d_motion_generate` (text → motion clip, 2–10 s; apply within
  3 days) → `models3d_animate`.
- Clean-up: `models3d_remesh`, `models3d_retexture`, `models3d_uv_unwrap`,
  `models3d_segment`, `models3d_complete`, `models3d_resize`,
  `models3d_convert`.

---

## 8. Approval rules — when to ask before spending

Match these to the user's `cost_mode`. Numbers are `simulate_cost` values
at the time of writing — re-check live.

| Operation | ~Credits | quality-first | balanced | careful |
|-----------|---------|---------------|----------|---------|
| Draft image (GPT 2.5 low, NB2 Lite, Recraft mid) | 15–60 | ✅ go | ✅ go | ✅ go |
| Standard image (Seedream 5 Pro, NB2, NB Pro 2k, GPT medium) | 75–100 | ✅ go | ✅ go | ⚠️ confirm |
| High-end image (GPT high/max 4k, Recraft pro) | 175–1,000 | ✅ go | ⚠️ confirm | ❌ confirm |
| Batch ≥ 8 images | ×8 | ✅ go | ⚠️ confirm | ❌ confirm |
| Cheap video (H3 Turbo, Kling 2.5, Kling 3.0 720p, Kling Omni) | 200–500 | ✅ go | ⚠️ confirm | ❌ confirm |
| Mid video (Veo 3.1 Fast, Seedance Fast, Gemini Omni) | 800–1,200 | ⚠️ confirm | ❌ confirm | ❌ confirm |
| Seedance 2.5 / 2.0 at 720p+ | 2,000–8,000+ | ⚠️ confirm | ❌ confirm | ❌ confirm |
| Video editing (`video_modify`, `extend`, `magic_cut`, `camera_angle`, `relight`) | varies | ⚠️ confirm | ⚠️ confirm | ❌ confirm |
| Flow ≥ 1,000 credits | — | ⚠️ confirm | ❌ confirm | ❌ confirm |
| 3D generation / rig / animate | varies | ⚠️ confirm | ⚠️ confirm | ❌ confirm |
| Upscale 4K+ (image or video) | varies | ✅ go | ⚠️ confirm | ❌ confirm |
| Free tools (`simulate_cost`, `video_analyze`, `images_split_grid`, `video_concatenate`, catalogs, dubbing preview) | 0 | ✅ go | ✅ go | ✅ go |

⚠️ = say the cost in one line and proceed unless the user objects in the
same turn. ❌ = state the `simulate_cost` number and wait for a yes.

**Rule of thumb**: if a single call could cost > 500 credits, say the
number out loud before launching.

---

## 9. Flows and Spaces — check before building from scratch

**Flows** are pre-built multi-step workflows. Many common requests have a
Flow that's cheaper and more reliable than a custom chain.

1. `flows_list({ ownership: "public", query: "<keyword>" })` — narrow by topic.
2. `flows_get({ identifier })` — read inputs, outputs, live cost.
   (`simulate_flows` also estimates.)
3. Confirm: *"There's a Flow for this — `X` (~Y credits). Use it or build
   custom?"*
4. `flows_run`; `flows_wait` if you need the final URL for a chained call.

Typical public Flows: brand merch shots, product colour variants, icon
generator, room decorator, render-to-photoreal, dress a model, audience-
driven ads, detailed storyboard, mockup realizer, UGC scripted video, looped
motion, frame-to-frame, photo-to-motion, camera path. Costs range from ~75
to ~8,500 credits — always read `flows_get` first.

**Spaces** are node-based canvases (`spaces_create`, `spaces_edit`,
`spaces_run`, `simulate_spaces` for cost). Use them when the user wants a
reusable pipeline they can open and tweak in the Magnific UI. A reference
node uses its numeric `id` as `modifierId`.

---

## 10. The Library — reuse characters, styles, products, locations

This is the highest-leverage feature of Magnific and the most underused.

**When the user wants a recurring character, brand product, location or style**:
1. Create a library entry from 1–6 reference images: `library_create`.
2. Confirm with the user via `library_show` (inline picker).
3. From now on, pass the entry's numeric `id` in
   `images_generate.references[].type: character|product|locations|style`
   — the same entry works for `video_generate` references too.
4. Mention it in the prompt as `@<name>` where it appears in the scene.

Headless search → `library_list`. User-facing picker → `library_show`.
Share with teammates → `library_share`.

**Critical**: Library ids and creation identifiers are different types —
never swap them.

---

## 11. Standard generation flow

```
simulate_cost(tool, args)          ← when cost_mode requires it (§8)
       │
images_generate / video_generate / …
       │
       ├─ result.instruction tells you what to do next — follow it
       ▼
creations_show(identifiers)        ← UI-capable clients (inline preview)
   OR share webUrl                  ← text-only clients
       │
creations_wait(identifiers)        ← 1–8 per call; confirms + gives final url
```

- Never regenerate a creation that's still queued — wait for it.
- For chained tools, use the `url` from `creations_wait` / `creations_get`
  or the identifier — never `webUrl`.

### Organising output

- `folderReference` (from `folders_list`) files a result into a project.
  **It does not persist** — once the user names a target folder, pass it on
  **every** later generation/edit call.
- `project_report({ reference, days })` — markdown status report of a
  project's recent activity (1–30 days). Handy for client updates.
- Tags (`tags_*`), comments (`creations_comment`), likes and
  `creations_move` / `projects_move` for housekeeping.

### Uploading local files

For a local file (e.g. after ffmpeg processing) use
`creations_request_upload` → `curl -X PUT --data-binary @file <url>` →
`creations_finalize_upload` → identifier. The bytes go straight from disk,
never through the model context. Much cheaper than base64.

### Stock

`stock_search` / `stock_get` / `stock_download` / `stock_to_creation` —
licensed stock photos/videos from the same account. `stock_to_creation`
turns a stock asset into a creation you can edit with any tool above.

---

## 12. Budget tracking + post-job reporting

**Always report credit usage after every job that consumes credits.** This is
non-negotiable regardless of `tracking` mode — the user wants to see what
each operation cost. Tracking mode only controls whether you persist the log
to disk.

### After every paid call

1. Call `account_balance` to get the current balance.
2. Compute `cost = previous_balance - current_balance`. If no previous
   balance is known, use the `simulate_cost` number you checked, or report
   just the balance.
3. **Tell the user inline**, in one short line:

   > **"השתמשנו ב-X קרדיטים. נשארו Y."**
   > *(English: "Used X credits. Y remaining.")*

   If `tracking: auto` and `monthly_budget` is set, extend it:

   > **"השתמשנו ב-X קרדיטים. נשארו Y. החודש: Z מתוך W."**

4. If `tracking: auto` — persist to `budget.json`:
   - Append `{ ts, tool, model, cost }` to `log`.
   - Update `used_this_month += cost` and `last_known_balance = current`.
5. If `used_this_month / monthly_budget >= alerts.threshold/100`, append a
   warning: *"⚠️ עברת X% מהתקציב החודשי."*

When the month rolls over, reset `used_this_month` to 0 and stamp the new
`current_month`.

### `budget.json` shape

When `tracking: auto`, maintain a `budget.json` in the user's project root
(or fall back to `~/.claude/skills/magnific-mcp/budget.json`).

```json
{
  "monthly_budget": 35000,
  "current_month": "YYYY-MM",
  "used_this_month": 0,
  "last_known_balance": null,
  "last_checked": null,
  "log": [
    { "ts": "YYYY-MM-DDTHH:MM:SSZ", "tool": "images_generate", "model": "seedream-5-pro", "cost": 100 }
  ]
}
```

### When `tracking` is `manual` or `summary-only`

- `manual` — still call `account_balance` after each job to show the live
  number; just don't write to `budget.json`.
- `summary-only` — log to `budget.json` but skip the API call; use the
  `simulate_cost` number as the cost.

---

## 13. Identifier hygiene

When writing back to the user:
- **Use** names, titles, `webUrl`, and plain descriptions.
- **Don't quote** internal identifiers, UUIDs, folder references, session
  ids, or request ids unless the user explicitly asks.
- Internal identifiers are for your next tool call only.

---

## 14. Common gotchas

- **`creations_search` is data-only.** It returns results but no preview.
  To show them, call `creations_show(identifiers)`.
- **`creations_wait` takes 1–8 identifiers** — batch larger sets.
- **Library entries vs creations** — different identifier types, not
  interchangeable.
- **Model display names ≠ slugs.** GPT 2.5 = `gpt-2-mini`; Nano Banana Pro
  = `imagen-nano-banana-2`. Copy slugs verbatim from the catalog.
- **Quality/resolution/duration dominate cost.** Same model can vary 10–60×.
  Re-run `simulate_cost` when any of them changes.
- **Video editing tools cap at ~20 s / 200 MB** (`color_grade`, `vfx`,
  `color_transfer`, `motion_shake`); `video_relight` at 240 frames. Cut
  first with `video_cut`.
- **Aspect ratio with references** — Magnific keeps the requested output
  ratio even when references differ; don't pre-crop unless you want to
  constrain composition.
- **Generating images for programmatic post-processing** (chroma-key,
  slicing, masking, OCR) — image models add decorative borders, grid lines,
  frames and labels even when not asked. In the prompt, **explicitly forbid
  them**: *"no grid lines, no borders, no frames, no separators, no labels,
  no text, no watermark, no decorative elements — just <subject> on a solid
  <BG_COLOR> background filling edge-to-edge"*. A clean regeneration beats
  30 minutes of clean-up. (Exception: when you *want* a grid for
  `images_split_grid`, ask for clearly separated equal tiles.)
- **Transparent PNGs** — use `transparentBackground: true` on GPT 2 / 2.5
  instead of generating on white and removing the background.

---

## 15. Quick "give me good defaults" recipe

If the user just says *"make me an image of X"* with no other hints:

1. **Check `flows_list` first** with the user's keywords — common tasks
   (icon, mockup, room decor, merch) have cheaper pre-built flows.
2. If no flow fits, pick by §3:
   - Text inside the image / Hebrew / Arabic → `gpt-2-mini` (GPT 2.5)
   - References / editing / general purpose → `seedream-5-pro`
   - Pure text→image, fast → `recraft-v4-1`
3. For exploration, generate `count: 4` at a cheap setting, then re-run the
   winner at full quality (same `seed`).
4. Aspect ratio: `1:1` unless context implies wide/tall.
5. After the result lands, `creations_show` it inline, then `creations_wait`.
6. Offer one upgrade path only if it improves something the user cares
   about (e.g. *"want a 4K pass with Nano Banana Pro?"*).

For video: `video_plan` → draft with `minimax-video-3_0-max-turbo` or
`kling-25` using the generated image as start keyframe → final with
`kling-omni3` (sound) or `bytedance-seedance-pro-2.5` (top quality, confirm
cost). Run `video_analyze` on the result to catch artifacts before
presenting it.

For audio TTS, `audio_voices_list` first so the user can pick a voice, and
`audio_tts_direction` before writing the script.

---

## 16. Updating this skill

This is a living guide. If a user shares a workflow they like, or you find a
better default through trial, propose adding it to this file — but keep the
skill **provider-neutral** (don't bake in account-specific assumptions like
plan tier or renewal dates; those belong in the user's local `budget.json`).

To refresh the catalog section: run `images_models_list({ onlyRecommended:
true })`, `video_models_list({ onlyRecommended: true })`,
`video_modify_models_list`, `video_speak_models_list`, and `simulate_cost`
on the recommended slugs — all free.

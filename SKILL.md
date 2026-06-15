---
name: magnific-mcp
description: >
  Best-practices guide for using the Magnific MCP server inside Claude. Covers
  first-run setup (usage alerts, credit tracking, cost preferences), model
  selection, character/style consistency via the Library, image→video chaining,
  budget management, and approval rules for expensive operations. Use whenever
  the user wants to generate images, videos, 3D models, audio or upscale media
  through Magnific — including phrases like "generate an image", "create a
  video", "Magnific", "Nano Banana", "Seedance", "Kling", "upscale this",
  "make a character", "magnific library", "how many credits", or any request
  to produce visual/audio media through Magnific's MCP tools.
---

# Magnific MCP — Best Practices

This skill packages accumulated knowledge for using the Magnific MCP server
effectively. It assumes you have an active Magnific subscription and have
connected the Magnific MCP to your Claude client.

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

## 2. Model selection cheat sheet

> **Live-truth principle**: this list is curated, but the authoritative
> source is the MCP itself. Run `images_models_list({ onlyRecommended: true })`
> or `video_models_list({ onlyRecommended: true })` to fetch the current
> SOTA-ranked set before any non-trivial job — Magnific rotates models.

### Images (current SOTA, as of June 2026)

| Slug | Best for | Generation | References supported |
|------|---------|-----------|---------------------|
| `recraft-v4-1` | **First drafts**, photoreal, illustration, typography, creative exploration — pure text→image with no refs | ~14 s | `style` only |
| `gpt-2` | **Text, layout, infographics, UI mockups, diagrams, typography**, non-photoreal design. **Best Hebrew / Arabic / RTL / complex scripts.** | ~77 s | `style`, `character`, `product`, `image` |
| `imagen-nano-banana-2` (Nano Banana **Pro**) | **Image editing, composition, character/product/brand consistency**, final assets | ~60 s | `style`, `character`, `product`, `image`, **`composition`** ← unique |
| `imagen-nano-banana-2-flash` (Nano Banana **2**) | Faster cheaper edits when Pro fidelity isn't required | ~36 s | same as Pro |

**Decision tree (in this order):**
1. **Output must contain rendered text** (any language, but especially
   Hebrew/Arabic/RTL/Asian scripts) → `gpt-2`.
2. **Editing an existing image, matching a character, product, brand, or
   composition** → `imagen-nano-banana-2` (Pro). Use `-flash` for cheaper drafts.
3. **First-shot creation, no references** → `recraft-v4-1` (fastest + cleanest).
4. When you switch model for a specific reason, say it briefly:
   *"Using GPT 2 because the prompt has Hebrew text."*

### Video (current SOTA)

| Slug | Best for | Durations | Resolutions | Special |
|------|---------|-----------|-------------|---------|
| `bytedance-seedance-pro-2.0` (**Seedance 2.0**) | **Best overall.** Realistic motion, complex action, **native audio + lipsync**, directed camera, multishot up to 6 shots, 52 camera motions | 4–15 s | 480p / 720p / 1080p | Audio refs, multishot, scene direction |
| `kling-25` (**Kling 2.5**) | **Best value/price.** Fast iteration on silent clips, start/end-frame control | 5 / 10 s | 720p / 1080p | Keyframes |
| `bytedance-seedance-fast-2.0` (Seedance Fast) | Fast Seedance drafts when max quality isn't needed | 4–15 s | 480p / 720p | Audio, multishot |

**Default**: `kling-25` for silent value clips, `bytedance-seedance-pro-2.0`
when audio/lipsync/cinematic control matters. Always run `video_plan` first
to validate the slug + cost before `video_generate`.

### Reference types — what each ref slot does

| Type | Effect | Models that accept it |
|------|--------|----------------------|
| `style` | Visual style transfer (palette, texture, vibe) | All |
| `character` | Same person/figure across generations | GPT 2, Nano Banana Pro/2 |
| `product` | Same branded object across generations | GPT 2, Nano Banana Pro/2 |
| `composition` | Match framing/layout/pose | **Only Nano Banana Pro/2** |
| `image` | Generic visual reference | GPT 2, Nano Banana Pro/2 |

**Pro tip**: when the user says *"keep the same layout but change X"*, that's
a `composition` reference and you need Nano Banana Pro. Other models can't
respect composition.

### Audio

- TTS — `audio_tts` (preview voices via `audio_voices_list` first)
- Music generation — `audio_music_generate`

### Upscale

- Images — `images_upscale`
- Video — `video_upscale` (run `video_upscale_models_list` to see options)

---

## 3. Reference images — HARD rules

These are **stop-and-think** rules. Wrong handling here wastes credits and
breaks the user's intent.

### Rule 3.1 — If the user provides a reference image, you MUST use it

When the user shares an image (file path, URL, screenshot, inline upload,
or a previously generated creation) and asks for a related generation,
**that image must end up in `references[]` of the generation call** — either
directly as a URL, by uploading it first via `creations_upload_image`, or
by routing through a Library entry.

Do not paraphrase the reference in words and hope for the best. Do not
silently drop it because the model "should know" what the user wants. The
reference is the user's specification.

### Rule 3.2 — If you can't access the reference, STOP and ask for access

If a reference is a local file you can't read, a private URL, or a creation
under a different account/MCP, **do not start the work**. Ask the user for
the file/URL/access first.

Acceptable failure modes:

- *"I can't read the file you mentioned at `<path>`. Can you upload it or
  paste a public URL?"*
- *"That creation belongs to a Magnific account I'm not connected to —
  switch the MCP or share the asset URL."*

**Never** proceed with a generation that ignores the reference. Wasting
credits on a fallback is worse than waiting.

### Rule 3.3 — Multiple references at once

`images_generate.references[]` accepts several entries with different
`type`s (`character`, `product`, `style`, `locations`, plus raw image URLs).
Use them together when the user's intent spans more than one — e.g.
character + branded product + location all in one call.

---

## 4. Flows — check before building from scratch

**Magnific ships a catalog of pre-built workflows ("Flows") that chain
multiple steps into one call.** Many common requests have a Flow that's
cheaper, faster, and more reliable than rolling your own chain.

**Always run `flows_list({ ownership: "public" })` early in a task** and
check for a match before designing a custom multi-step generation.

### Examples from the public catalog

| Task | Flow | Approx. cost |
|------|------|--------------|
| Branded merch lineup from a logo | `Brand merch shots` | ~75 credits |
| Recolor a product | `Product color variants` | ~75 |
| Generate a clean icon in a chosen style | `Icon generator` | ~150 |
| Decorate an empty room photo | `Room decorator` | ~150 |
| Convert a 3D render to photorealistic | `Render to photoreal` | ~75 |
| Dress a model in a given outfit | `Dress a model in any outfit` | ~75 |
| Audience-segmented ad variants | `Audience-driven ads` | ~200 |
| Cinematic storyboard from a synopsis | `Detailed storyboard` | ~200 |
| Product mockup in a scene | `Mockup realizer` | ~200 |
| UGC scripted talking-head video | `UGC scripted video` | ~932 |
| Loop a still image into endless motion | `Looped motion` | ~1,000 |
| Smooth transition between two frames | `Frame to frame` | ~1,200 |
| Photo → motion with camera direction | `Photo to motion` | ~5,700 |
| Live-stream commerce demo with UI overlay | `UGC livestream demo` | ~7,100 |
| Camera path on a still image (text-defined) | `Create your camera path` | ~8,500 |

(Costs vary. Always trust `flows_get` for the current input/output spec and
the live `totalCost` before invoking `flows_run`.)

### Workflow

1. `flows_list({ ownership: "public", query: "<keyword>" })` — narrow by topic.
2. `flows_get({ identifier })` on a candidate — read inputs, outputs, cost.
3. Confirm with the user: *"There's a Flow for this — `X` (~Y credits).
   Use it or build custom?"*
4. `flows_run` with the inputs. Use `flows_wait` if you need the final URL
   for a downstream chained call.

---

## 5. The Library — reuse characters, styles, products

This is the highest-leverage feature of Magnific and the most underused.

**When the user wants a recurring character, brand product, or style**:
1. Create a library entry from 1–6 reference images: `library_create`.
2. Confirm with the user via `library_show` (inline picker).
3. From now on, pass the entry's `identifier` in
   `images_generate.references[].type: character|product|style|locations`
   — the same identifier flows into video generation too.

Headless search → `library_list`. User-facing picker → `library_show`.

**Critical**: pass the library `identifier` as-is to generation tools. Don't
convert it to a creation identifier — they're different.

---

## 6. Standard generation flow

```
images_generate(model, prompt, references?)
       │
       ├─ result.instruction tells you what to do next — follow it
       │
       ▼
creations_show(identifiers)        ← UI-capable clients (inline preview)
   OR share webUrl                  ← text-only clients
```

For image→video chaining:
- Pass the **`identifier`** from the previous tool (or the `url` from
  `creations_get`/`creations_wait`) into `video_generate.keyframes` /
  `references`.
- Never pass `webUrl` — it's for users, not tools.

For Spaces (`spaces_*`): the reference node uses numeric `id` as `modifierId`.

---

## 7. Approval rules — when to ask before spending

Match these to the user's `cost_mode`:

| Operation | quality-first | balanced | careful |
|-----------|---------------|----------|---------|
| Single image (`recraft-v4-1`, `imagen-nano-banana-2-flash`, `gpt-2`) | ✅ go | ✅ go | ✅ go |
| Single image, top model (`imagen-nano-banana-2`) | ✅ go | ✅ go | ⚠️ confirm |
| Batch ≥ 8 images | ✅ go | ⚠️ confirm | ⚠️ confirm |
| Video `kling-25` (5 s, 720p) | ✅ go | ⚠️ confirm | ⚠️ confirm |
| Video `bytedance-seedance-fast-2.0` | ✅ go | ⚠️ confirm | ⚠️ confirm |
| Video `bytedance-seedance-pro-2.0` (especially ≥10 s, 1080p, multishot) | ⚠️ confirm | ❌ confirm with `video_plan` cost | ❌ confirm with `video_plan` cost |
| Flow ≥ 1,000 credits (e.g. `Photo to motion`, `UGC livestream`, `Camera path`) | ⚠️ confirm | ❌ confirm | ❌ confirm |
| 3D model generation | ⚠️ confirm | ⚠️ confirm | ❌ confirm |
| Upscale 4K+ | ✅ go | ⚠️ confirm | ⚠️ confirm |

**Rule of thumb**: if the operation could cost > 500 credits in one call, say
the rough cost out loud before launching.

---

## 8. Budget tracking + post-job reporting

**Always report credit usage after every job that consumes credits.** This is
non-negotiable regardless of `tracking` mode — the user wants to see what
each operation cost. Tracking mode only controls whether you persist the log
to disk.

### After every generation / upscale / video / audio call:

1. Call `account_balance` to get the current balance.
2. Compute `cost = previous_balance - current_balance`. If no previous
   balance is known (first call), report just the current balance.
3. **Tell the user inline**, in one short line. Pattern:

   > **"השתמשנו ב-X קרדיטים. נשארו Y."**
   > *(English: "Used X credits. Y remaining.")*

   If `tracking: auto` and `monthly_budget` is set, extend it:

   > **"השתמשנו ב-X קרדיטים. נשארו Y. החודש: Z מתוך W."**

4. If `tracking: auto` — persist to `budget.json`:
   - Append `{ ts, tool, model, cost }` to `log`.
   - Update `used_this_month += cost` and `last_known_balance = current`.
5. If `used_this_month / monthly_budget >= alerts.threshold/100`, append a
   warning to the same message: *"⚠️ עברת X% מהתקציב החודשי."*

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
    { "ts": "YYYY-MM-DDTHH:MM:SSZ", "tool": "images_generate", "model": "imagen-nano-banana-2", "cost": 25 }
  ]
}
```

### When `tracking` is `manual` or `summary-only`

- `manual` — still call `account_balance` after each job to get the live
  number to show the user; just don't write to `budget.json`.
- `summary-only` — log to `budget.json` but skip the API call; estimate cost
  from documented model pricing if known.

---

## 9. Identifier hygiene

When writing back to the user:
- **Use** names, titles, `webUrl`, and plain descriptions.
- **Don't quote** internal identifiers, UUIDs, folder references, session
  ids, or request ids unless the user explicitly asks.
- Internal identifiers are for your next tool call only.

---

## 10. Common gotchas

- **`creations_search` is data-only.** It returns results but no `webUrl`.
  If you want the user to see them, call `creations_show(identifiers)`.
- **`creations_wait` only when you need the final asset URL** for a chained
  tool call — not every generation needs it.
- **Library entries vs creations** — different identifier types, not
  interchangeable.
- **Workflows that need character + product + style** — pass all three as
  separate references in one call; the model blends them.
- **Aspect-ratio in references** — Magnific keeps the requested output AR
  even when references differ; don't pre-crop unless you specifically want
  to constrain composition.
- **Generating images that will be processed programmatically** (chroma-key,
  slicing, masking, OCR pre-processing) — image models often add decorative
  borders, grid lines, frames, watermark-style labels, even when not asked.
  In the prompt, **explicitly forbid them**: *"no grid lines, no borders,
  no frames, no separators, no labels, no text, no watermark, no decorative
  elements — just <subject> on a solid <BG_COLOR> background filling
  edge-to-edge"*. Spending 75 credits on a clean regeneration beats 30 min
  of post-processing to remove them.

---

## 11. Quick "give me good defaults" recipe

If the user just says *"make me an image of X"* with no other hints:

1. **Check `flows_list` first** with the user's keywords — common tasks
   (icon, mockup, room decor, merch) have cheaper pre-built flows.
2. If no flow fits, pick model by the decision tree in §2:
   - Pure text→image, no refs → `recraft-v4-1` (fast, clean)
   - Text inside the image / Hebrew / Arabic → `gpt-2`
   - Editing or matching something → `imagen-nano-banana-2` (Pro)
3. Aspect ratio: `1:1` unless context implies wide/tall.
4. After result lands, `creations_show` it inline.
5. Offer one upgrade path: *"want a higher-fidelity pass with Nano Banana
   Pro?"* — only suggest upgrades that improve something the user cares
   about (don't upsell for the sake of it).

For video, default to `kling-25` (5 s, the generated image as start
keyframe). Switch to `bytedance-seedance-pro-2.0` when the brief mentions
audio, lipsync, multishot, or directed camera motion.

For audio TTS, `audio_voices_list` first so the user can pick a voice.

---

## 12. Updating this skill

This is a living guide. If a user shares a workflow they like, or you find a
better default through trial, propose adding it to this file — but keep the
skill **provider-neutral** (don't bake in account-specific assumptions like
plan tier or renewal dates; those belong in the user's local `budget.json`).

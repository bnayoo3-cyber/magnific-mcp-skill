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

Always prefer the smallest model that meets the task — bigger ≠ better, just
more expensive.

### Images

| Need | Model |
|------|-------|
| Fast iteration / mood boards | `imagen-nano-banana-2` |
| Character / product consistency | `imagen-nano-banana-2-pro` (reference-guided) |
| Photoreal output | `imagen-4-ultra` |
| Highest fidelity, willing to pay | `flux-1.1-pro-ultra` |
| SVG / vector | `images_generate_svg` |

### Video

| Need | Model | Cost class |
|------|-------|------------|
| Default text→video, image→video | `kling-2-5` | low/medium |
| Talking head from photo+script | `omni-3` (via `video_speak`) | medium |
| Top-tier motion realism | `seedance` | **HIGH — ~10× kling** |

### Audio

- TTS — `audio_tts` (pick voice via `audio_voices_list` first)
- Music generation — `audio_music_generate`

### Upscale

- Images — `images_upscale`
- Video — `video_upscale`

---

## 3. The Library — reuse characters, styles, products

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

## 4. Standard generation flow

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

## 5. Approval rules — when to ask before spending

Match these to the user's `cost_mode`:

| Operation | quality-first | balanced | careful |
|-----------|---------------|----------|---------|
| Single image, base model | ✅ go | ✅ go | ✅ go |
| Single image, top model (`flux-pro-ultra`, `imagen-4-ultra`) | ✅ go | ✅ go | ⚠️ confirm |
| Batch ≥ 8 images | ✅ go | ⚠️ confirm | ⚠️ confirm |
| Video (`kling-2-5`, `omni-3`) | ✅ go | ⚠️ confirm | ⚠️ confirm |
| Video (`seedance`) | ⚠️ confirm | ❌ confirm with cost estimate | ❌ confirm with cost estimate |
| 3D model generation | ⚠️ confirm | ⚠️ confirm | ❌ confirm |
| Upscale 4K+ | ✅ go | ⚠️ confirm | ⚠️ confirm |

**Rule of thumb**: if the operation could cost > 500 credits in one call, say
the rough cost out loud before launching.

---

## 6. Budget tracking helper

When `tracking: auto`, maintain a `budget.json` in the user's project root
(or fall back to `~/.claude/skills/magnific-mcp/budget.json`).

Minimal shape:

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

After every generation:
1. Call `account_balance`.
2. Diff against `last_known_balance` to derive `cost`.
3. Append to `log` and update `used_this_month`.
4. If `used_this_month / monthly_budget >= alerts.threshold/100`, warn the
   user inline: *"You've used 76% of your monthly Magnific credits."*

When the month rolls over, reset `used_this_month` and stamp the new month.

---

## 7. Identifier hygiene (per Magnific's own MCP instructions)

When writing back to the user:
- **Use** names, titles, `webUrl`, and plain descriptions.
- **Don't quote** internal identifiers, UUIDs, folder references, session
  ids, or request ids unless the user explicitly asks.
- Internal identifiers are for your next tool call only.

---

## 8. Common gotchas

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

---

## 9. Quick "give me good defaults" recipe

If the user just says *"make me an image of X"* with no other hints:

1. Model: `imagen-nano-banana-2` (cheap, fast, good).
2. Aspect ratio: `1:1` unless context implies wide/tall.
3. No references unless they referenced a prior character.
4. After result lands, `creations_show` it inline.
5. Offer one upgrade path: *"want a higher-fidelity pass with
   `imagen-4-ultra`?"*

For video, the equivalent default is `kling-2-5`, 5 seconds, the
first generated image as keyframe.

---

## 10. Updating this skill

This is a living guide. If a user shares a workflow they like, or you find a
better default through trial, propose adding it to this file — but keep the
skill **provider-neutral** (don't bake in account-specific assumptions like
plan tier or renewal dates; those belong in the user's local `budget.json`).

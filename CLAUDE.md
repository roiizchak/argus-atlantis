# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single scroll-scrubbed "fly through the world" landing page — **Argus**, a fan tribute to Atlantis. As the visitor scrolls, a pre-rendered camera flies from outside each scene into its interior, then flows into the next scene with no cuts: one continuous connected flight. Built with the `scroll-world` skill (installed plugin), which owns the whole approach — engine, asset pipeline, and gotchas.

Not a git repo. No build step, no package manager, no framework. The deliverable is `index.html` + `scrub-engine.js` + `assets/`.

## Serve / preview

Static files only. Serve the root and open the page. Prior sessions served on port **8123** (`http://127.0.0.1:8123/`) and previewed with the `playwright-cli` skill (logs in `.playwright-cli/`):

```bash
python -m http.server 8123     # then http://127.0.0.1:8123/
```

The `favicon.ico` 404 in the console logs is expected — no favicon shipped.

`?v=f1` query strings on every asset URL in `index.html` are cache-busters — bump them when you replace an asset so the browser refetches.

## Architecture

**Two files ship:**
- `index.html` — the config. Calls `mountScrollWorld(container, config)` with the `sections[]` (5 dive scenes), `connectors[]` (4 clips bridging them), brand, theme CSS vars, and scroll tuning. All Atlantis art direction and dark-theme CSS overrides live in its `<style>` and the config object.
- `scrub-engine.js` — the engine, **vendored verbatim from the scroll-world skill**. Framework-agnostic vanilla JS, zero deps, builds its own namespaced (`.sw-*`) DOM + CSS into the container. Do not hand-edit it to fix a page — it is shared code; change `index.html` config or CSS overrides instead. Its 87-line header comment is the API reference.

**The flight is an interleaved segment chain:** `dive0, conn0, dive1, conn1, … diveN-1`. Each dive is a scene's fly-in clip; each connector bridges two adjacent dives. The engine loads each clip as a **Blob** and scrubs `currentTime` against scroll position (does not rely on HTTP byte-range). `connectors.length === sections.length - 1`.

**The seam contract (the thing that makes it seamless):** a connector's start frame must equal the previous dive's *last* frame, and its end frame the next dive's *first* frame. Connectors are generated start-image→end-image from the neighbouring dives' actual extracted frames, then verified with an SSIM check (see `work/final_chain.sh`). Broken seams = visible cuts; this is the #1 failure mode (see prior session memory on frame-handoff failures).

**Mobile:** two independent axes. *Clip tier* (which file) keys off device class — phone short-side ≤600px gets `clipMobile`/`posterMobile`; iPad/desktop get the master. *Behaviour hardening* (seek coalescing, priming, poster-hold) keys off coarse-pointer / ≤860px viewport. **Stills mode** is an automatic fallback (reduced-motion, data-saver, iOS Low Power Mode) that cross-dissolves stills with no video decode.

**SEO:** the engine builds DOM client-side, so real crawlable copy lives in the `data-sw-seo` block inside `#world` — the engine hides it on mount. Keep it in sync with the section copy.

## Asset generation pipeline

All media is generated with the **Higgsfield CLI** (`higgsfield generate create seedance_2_0 …`) and encoded with **ffmpeg**. The generation order matters because of the seam contract:

1. **Stills** — one establishing render per scene (`work/prompts/sceneN_*.txt`, sharing `work/prompts/preamble.txt` for consistent art direction / palette).
2. **Dives** — animate each still into a fly-in clip (`work/prompts/diveN_*.txt`), start-image = the scene still.
3. **Frames** — extract each dive's first + last frame with ffmpeg.
4. **Connectors** — generate start→end between adjacent dives' extracted frames (`work/prompts/connN.txt`), SSIM-gated (accept ≥0.5 on the start seam).

`work/final_chain.sh` runs the full dive+connector chain (1080p, parallel, 3–4 retries each, caches finished `.mp4`s). Run it from `work/`. `work/previz/` holds earlier lower-res passes; `work/final/` is the current master render.

**Encoding to web assets** (from `references/pipeline.md` in the skill):
- Clips: `libx264 -crf 20 -g 8 -keyint_min 8 -sc_threshold 0 -pix_fmt yuv420p -movflags +faststart`, no audio, light `unsharp`. Small GOP is deliberate — scrub seek cost scales with frames-from-keyframe.
- Mobile clips: `scale=-2:720`, `-crf 23 -g 4`.
- Posters: extract the **encoded** clip's first frame (`ffmpeg -ss 0 -frames:v 1`) → `cwebp -q 84`. Extracting from the encoded file (not the still) makes the still→video swap pixel-identical.
- Stills: `cwebp -q 84 -resize 1800 0`.
- Note: `cwebp` was unavailable in a prior session — a WebP fallback path was used. Verify the binary before relying on it.

## Source of truth for how to do all of the above

The `scroll-world` skill — invoke it before regenerating scenes or editing the flight. Its reference files are the real manuals:
`~/.claude/plugins/cache/scroll-world/scroll-world/0.2.0/skills/scroll-world/` → `SKILL.md`, `references/pipeline.md` (ffmpeg/cwebp commands), `references/prompts.md` (prompt structure), `references/gotchas.md` (NSFW moderation retries, aspect-ratio traps, model quirks).

Project-specific learnings from past sessions are in the memory index at `~/.claude/projects/c--Claude-Scroll-World-Website/memory/MEMORY.md` and the claude-mem timeline — check them for known Higgsfield/SSIM gotchas before re-rolling assets.

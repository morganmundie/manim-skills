---
name: manim-export
description: Render an existing Manim (ManimCE) Scene to a final video/image file with the right quality, dimensions, and transparency for how it'll be used. Use this whenever the user asks to render, export, output, or "actually make the video" from a manim scene, or specifies quality ("hi def", "high quality", "4k", "low res", "quick preview"), format/orientation ("short", "vertical", "for TikTok/Reels/Shorts", "long", "landscape", "widescreen"), or background ("transparent", "with alpha", "no background", "so I can overlay it"). This is purely about the `manim` CLI render step on a scene that already exists — for writing/fixing the animation code itself, use the manim-create skill instead.
---

# Manim: Export / Render

Manim always renders through the `manim` command-line tool acting on a Python file containing one or more `Scene` subclasses. This project's `manim` is a `uv`-managed dependency (see the manim-create skill), so invoke it via `uv run` rather than calling `manim` directly:

```
uv run manim [flags] <script>.py <SceneName>
```

If the script contains only one `Scene` subclass you can omit `<SceneName>`. If you want every `Scene` in the file rendered, use `-a` instead of naming one.

Your job is to pick the right flag combination from what the user asked for — resolve **quality**, **dimensions/orientation**, and **background** independently, then combine them into one command. Don't guess at a scene's filename/class name — check the actual file if you don't already know it.

## 1. Quality (hi-def / low-def / preview)

| They said | Flag | Resolution | FPS |
|---|---|---|---|
| quick preview / draft / "just check it works" | `-ql` | 854x480 | 15 |
| standard / medium | `-qm` | 1280x720 | 30 |
| **hi-def / high quality (default assumption if they just say "hi def" or "good quality")** | `-qh` | 1920x1080 | 60 |
| 2k | `-qp` | 2560x1440 | 60 |
| 4k / ultra hi-def | `-qk` | 3840x2160 | 60 |

Default to `-ql` for quick iteration/preview while a scene is still being developed, and `-qh` once the user wants an actual deliverable, unless they say otherwise. Always ask yourself which situation you're in rather than defaulting blindly — a request like "render this" mid-editing-session usually wants a fast preview; "export the final video" wants `-qh` or better.

Override the frame rate independent of the quality preset with `--fps <n>` if asked for something like 24fps or 25fps.

## 2. Dimensions / orientation (short vs long)

The quality flags above assume **landscape** (widescreen, width > height) — that's "long-form" orientation, for YouTube-style or standard playback. For "short" formats (vertical, for TikTok/Reels/Shorts/Stories), swap width and height using `-r width,height`, which overrides the quality preset's resolution while keeping everything else (you can still combine it with a quality flag for FPS, or set `--fps` directly).

| Orientation | at low-def | at hi-def (1080) | at 4k |
|---|---|---|---|
| **Long / landscape (default)** | 854x480 | 1920x1080 | 3840x2160 |
| **Short / vertical/portrait** | `-r 480,854` | `-r 1080,1920` | `-r 2160,3840` |

Example — hi-def vertical short: `uv run manim -qh -r 1080,1920 scene.py MyScene`

Manim recalculates the in-scene coordinate frame width from the new aspect ratio automatically (the frame stays 8 manim-units tall by convention), so existing scene code doesn't need to change — but a scene built and eyeballed for landscape may end up with wide elements running off the narrower edges in portrait. If the user wants a vertical export of a scene that was designed for landscape, mention that some repositioning (or scaling down of wide mobjects, larger font sizes since there's less horizontal room) may be worth revisiting, rather than silently assuming the crop will look right.

If they name a specific platform, that implies vertical: TikTok/Reels/Shorts/Stories → vertical. YouTube/general/no platform named → landscape (default).

For square (Instagram feed-style): `-r 1080,1080` (or matching square at whatever quality).

## 3. Background (transparent vs regular)

**Default is a regular (opaque) background** — don't add transparency unless the user asks for it ("transparent", "with alpha", "no background", "so I can layer/composite it").

- Regular: no extra flag needed. Background is whatever `self.camera.background_color` is set to in the scene (black by default).
- Transparent: add `-t` (`--transparent`). Because standard mp4 doesn't support an alpha channel, manim will output to an alpha-capable container instead — default is `.mov`; if the user needs `.webm` specifically, add `--format webm` alongside `-t`.

Example — transparent hi-def: `uv run manim -qh -t scene.py MyScene`

If the user wants a solid custom color instead of manim's default black (and isn't asking for transparency), that's set via `-c "#RRGGBB"` (or `--background_color`) at render time, or `self.camera.background_color` in the scene itself.

## 4. Other useful flags to reach for as needed

| Need | Flag |
|---|---|
| Play the video automatically once rendered | `-p` |
| Open the containing folder instead of playing | `-f` |
| Custom output filename | `-o <name>` |
| GIF instead of video | `--format gif` |
| Only render/save the final frame (fast thumbnail) | `-s` (combine with a quality flag, e.g. `-s -qh`) |
| Render every Scene class in the file | `-a` |
| Skip caching / force re-render from scratch | `--disable_caching` |

## Putting it together

Resolve each of the three axes (quality, orientation, background) from the request, then combine flags in one call. A few worked examples:

- "give me a low-res preview" → `uv run manim -pql scene.py MyScene`
- "export this in high def for YouTube" → `uv run manim -qh scene.py MyScene`
- "I need a 4k vertical version for Reels" → `uv run manim -qk -r 2160,3840 scene.py MyScene`
- "hi-def with a transparent background so I can put it over other footage" → `uv run manim -qh -t scene.py MyScene`
- "quick vertical draft, transparent" → `uv run manim -ql -r 480,854 -t scene.py MyScene`

Output lands under `media/videos/<script_name>/<resolution>p<fps>/<SceneName>.mp4` (or `.mov`/`.gif`/etc. matching the format) relative to wherever `manim` was invoked, unless `-o` was used — check there (or use `-f` to pop the folder open) rather than assuming a path.

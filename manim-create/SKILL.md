---
name: manim-create
description: Turn a rough, incomplete, or vague idea into a working Manim Community Edition (manim) animation script — a Python file with a Scene subclass ready to render. Use this whenever the user describes an animation concept in plain language ("show a circle turning into a square", "visualize a sine wave", "animate a neural network", "explain the Pythagorean theorem visually", "make a video about X"), asks to build, write, create, or fix a manim scene, or mentions manim/Manim/ManimCE at all. The user's description will almost never map cleanly onto manim's API — that gap is exactly what this skill closes: it translates fuzzy creative intent into concrete Mobjects, Animations, and a construct() method, filling in reasonable defaults for anything unstated (colors, timing, layout, camera). Do NOT use this for rendering/exporting an already-written scene to video (see the manim-export skill) or for syncing animation timing to audio (see manim-sync-audio).
---

# Manim: Idea → Animation

Manim (Community Edition, "ManimCE") is a Python library for precise, programmatic animation, mostly used for math/CS/science explainers. Your job in this skill is to act as the translator between a human's loose creative idea and manim's actual vocabulary of `Mobject`s (things on screen) and `Animation`s (ways things change over time).

The core loop, every time:

1. **Set up the project.** If there's no `pyproject.toml` in the working directory yet, this is a new project: run `uv init` then `uv add manim` to scaffold it and pin manim as a dependency, before writing any scene code. If a `pyproject.toml` already exists, just check `manim` is a dependency (`uv add manim` again if not) rather than re-initializing.
2. **Understand the idea first.** Most requests are underspecified on purpose, the creative process may need to iterate, and the user may not know what manim can do. Ask clarifying questions about the idea before writing any code, and don't assume you know what "looks better" in the abstract — only change things based on what the user says about the rendered result.
3. **Map concepts to manim vocabulary.** Use `references/vocabulary.md` to translate phrases like "morphs into", "zooms in", "counts up", "draws itself" into the actual class names (`Transform`, `MovingCameraScene`, `ValueTracker`, `Create`, etc.).
4. **Sketch the shape of the scene before writing code**: what mobjects exist, what order they appear/change/leave in, roughly how long each beat takes. A short mental (or literal, in a comment) storyboard prevents a scene that's just a pile of disconnected `self.play()` calls.
5. **Write one `Scene` subclass** (or `ThreeDScene`/`MovingCameraScene` if the idea needs 3D or camera movement — see `references/scene_types.md`) with a `construct()` method containing the animation.
6. **Render a low-quality preview** to sanity-check it (`uv run manim -pql <file>.py <SceneName>`) if you have shell access. If you don't have execution access in this environment, just hand over the code and tell the user the render command to run themselves — don't pretend to have previewed it.
7. **Iterate based on what the user says about the result**, not by guessing what "looks better" in the abstract.

## Defaults when the idea doesn't specify something

Don't ask about these unless the user's request implies they care — just pick and move on:

- **Background/colors**: default black background, manim's default palette (`BLUE`, `RED`, `GREEN`, `YELLOW`, etc. — see `references/colors_and_style.md`). Only theme it (light background, brand colors) if asked.
- **Timing**: `self.play(...)` defaults to `run_time=1`. Slow down key "aha" moments (`run_time=2`–`3`), speed through connective tissue. Add `self.wait(0.5)`–`self.wait(1)` after beats the viewer needs a moment to absorb, and a `self.wait()` at the very end so the last frame doesn't cut off abruptly.
- **Camera**: plain `Scene` (static camera) unless the idea implies panning/zooming/3D, in which case use `MovingCameraScene` or `ThreeDScene`.
- **Text**: `Text(...)` for plain text/labels, `Tex`/`MathTex` only when there's actual math notation (equations, symbols) — `MathTex` needs LaTeX syntax, `Text` doesn't.
- **Class/file naming**: name the `Scene` subclass in `CamelCase` after what it depicts (`CircleToSquare`, `NeuralNetworkForwardPass`), and save it as a lowercase-with-underscores `.py` file. If the user has an existing project structure, follow it instead.
- **Scope of one Scene**: one coherent beat or short sequence of beats per `Scene` class. If an idea is really a multi-minute video with distinct segments, use multiple `Scene` classes in the same file (one per segment) rather than one giant `construct()` — easier to preview and re-render pieces.

## Writing the code itself

- `from manim import *` at the top — this is the standard, expected convention in manim scripts, not sloppy practice.
- Build mobjects, position/style them, *then* animate. Don't animate the creation of a mobject before it has the properties it should appear with (set color/fill before `Create`, not after, unless the animation is specifically about the color changing).
- Prefer `.animate.method()` syntax (e.g. `square.animate.shift(LEFT)`) for simple property changes — it's concise and idiomatic. Fall back to explicit `Animation` classes (`Rotate`, `Transform`, `MoveAlongPath`, etc.) when `.animate` would produce the wrong interpolation (see the `.animate` vs `Rotate` gotcha in `references/vocabulary.md`) or when the effect has no `.animate` equivalent.
- Use `VGroup`/`Group` to move and animate clusters of related mobjects together instead of repeating the same transform on each one.
- For anything that should track a changing value and stay in sync (a counter, a dot following a curve, a label that follows a moving object), reach for `ValueTracker` + `add_updater` rather than manually animating each frame's position — see `references/vocabulary.md` for the pattern.
- Add short inline comments marking the narrative beats (`# introduce the problem`, `# reveal the answer`) — this makes the script easy for the user (or future-you) to re-edit piece by piece, and doubles as your storyboard.

## Read the reference files as needed

Don't try to hold the whole manim API in your head — load these when relevant instead of guessing class names:

- `references/vocabulary.md` — **start here for most requests.** Maps common idea-phrases ("appears letter by letter", "highlight this", "morphs into", "counts up", "follows a path") to the exact manim classes/patterns, plus the `.animate` vs explicit-Animation gotcha and the `Transform` vs `ReplacementTransform` distinction.
- `references/mobjects.md` — catalog of available shapes, text/math objects, graphs/axes, tables, images, and 3D objects, organized by what they're for.
- `references/animations.md` — catalog of animation classes (creation, transforms, fading, growing, indication/highlighting, movement, rotation, composition/timing).
- `references/scene_types.md` — when to use `Scene` vs `MovingCameraScene` vs `ThreeDScene` vs `ZoomedScene`, with minimal working examples of each.
- `references/colors_and_style.md` — built-in color constants and basic styling (fill/stroke/opacity) conventions.

Each reference is a lookup table, not required reading — skim for the row/section you need.

## A minimal complete example, for calibration

```python
from manim import *

class CircleToSquare(Scene):
    def construct(self):
        circle = Circle(color=BLUE, fill_opacity=0.5)
        square = Square(color=GREEN, fill_opacity=0.5)

        self.play(Create(circle))
        self.wait(0.5)
        self.play(Transform(circle, square))
        self.wait(0.5)
        self.play(FadeOut(circle))
        self.wait()
```

Render/preview with: `uv run manim -pql circle_to_square.py CircleToSquare` (see the manim-export skill for quality/format/aspect-ratio options once the user wants a final version).

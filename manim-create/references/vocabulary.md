# Idea → Manim vocabulary

A lookup table for translating what someone *says* they want into what to actually write.

## "It appears / shows up"

| They said | Use |
|---|---|
| "just appears" / pop in | `FadeIn(mobj)` |
| "draws itself" / traces its outline | `Create(mobj)` (good for shapes, lines, graphs) |
| "fills in" with border first | `DrawBorderThenFill(mobj)` |
| "grows in" from nothing | `GrowFromCenter(mobj)`, `GrowFromPoint(mobj, point)`, `GrowFromEdge(mobj, edge)`, `GrowArrow(arrow)` |
| "spins in" | `SpinInFromNothing(mobj)` |
| text "types on screen" letter by letter | `AddTextLetterByLetter(text_mobj)` |
| text appears word by word | `AddTextWordByWord(text_mobj)` |
| handwriting-style reveal (works for `Tex`/`MathTex`/`Text`) | `Write(mobj)` |
| a partial/gradual reveal of an existing shape | `ShowPartial`, `ShowIncreasingSubsets`, `ShowSubmobjectsOneByOne` |

## "It leaves / disappears"

| They said | Use |
|---|---|
| "fades away" | `FadeOut(mobj)` |
| "erases itself" (reverse of Write/Create) | `Uncreate(mobj)`, `Unwrite(text_mobj)` |
| text "un-types" | `UntypeWithCursor` / `RemoveTextLetterByLetter` |

## "It morphs / becomes / turns into"

| They said | Use |
|---|---|
| "A turns into B" (interpolates points/color/etc. of A toward B, keeps identity of `a`) | `Transform(a, b)` |
| "A turns into B" and B should be the thing left on screen / referenced afterward | `ReplacementTransform(a, b)` — literally replaces `a` with `b` in the scene |
| chaining several morphs on the same mobject in sequence | `Transform` repeatedly on the same source mobject (avoids re-tracking which copy is "current") — see `TransformCycle` pattern below |
| morphing text where matching letters/parts should glide to their new spot | `TransformMatchingShapes(old_tex, new_tex)` or `TransformMatchingTex(...)` |
| "snaps back to how it was" (undo an animate.something) | call `.save_state()` on the mobject earlier, then `Restore(mobj)` |
| color change only | `mobj.animate.set_color(RED)` or `FadeToColor(mobj, RED)` |

```python
# TransformCycle pattern: morph one shape through several targets in sequence
a = Circle()
for target in [Square(), Triangle()]:
    self.play(Transform(a, target))
```

## "It moves"

| They said | Use |
|---|---|
| simple move/shift/scale/rotate of one mobject | `.animate` syntax: `mobj.animate.shift(RIGHT).scale(0.5)` |
| move along a specific path/curve | `MoveAlongPath(mobj, path_mobj)` |
| continuous rotation over time (not a fixed end-angle) | `Rotating(mobj, radians, about_point=...)` |
| one-shot rotation by a fixed angle — prefer this over `.animate.rotate()` for 180°/full rotations, since `.animate` interpolates start/end state directly and can produce a shrink-then-grow artifact on symmetric rotations | `Rotate(mobj, angle=PI)` |
| position relative to another mobject | `mobj.next_to(other, RIGHT, buff=0.5)`, `.to_edge(UP)`, `.to_corner(UL)`, `.move_to(other.get_center())` |
| something continuously tracks/follows another moving thing | `add_updater` (see Updaters section below) |

## "Highlight / draw attention to"

| They said | Use |
|---|---|
| "circle it" / box it | `Circumscribe(mobj)`, `SurroundingRectangle(mobj)` |
| "flash" / brief attention-getter | `Flash(point_or_mobj)`, `Indicate(mobj)` |
| "wiggle" | `Wiggle(mobj)` |
| "make a wave pass through it" | `ApplyWave(mobj)` |
| "shake the camera's focus onto it" | `FocusOn(point)` |
| a moving object should leave a fading streak | `ShowPassingFlash(mobj.copy().set_color(...))` |

## "Counts up / a number changes / a value drives motion"

Use `ValueTracker` — the standard manim pattern for anything continuously parameterized by a changing number (counters, sliders, angle trackers, physics-like motion).

```python
tracker = ValueTracker(0)
number = DecimalNumber(0)
number.add_updater(lambda d: d.set_value(tracker.get_value()))
self.add(number)
self.play(tracker.animate.set_value(100), run_time=2)
```

For a mobject whose position/appearance is a function of the tracker (a dot walking along a curve as `t` increases, an angle arc that grows), give it an `add_updater(lambda m: m.become(...))` or `add_updater(lambda m: m.move_to(...))` that reads `tracker.get_value()`, then animate the tracker itself with `tracker.animate.set_value(target)`.

Remove updaters with `mobj.remove_updater(fn)` once they're no longer needed (e.g. before the mobject is repurposed for something else).

## "Camera moves / zooms / pans / is 3D"

See `references/scene_types.md` for the full picture — short version:

| They said | Use |
|---|---|
| camera pans/zooms to follow action | `MovingCameraScene`, animate `self.camera.frame` |
| a small inset magnifying part of the scene | `ZoomedScene` |
| genuinely 3D content / rotate around an object | `ThreeDScene`, `self.set_camera_orientation(phi=..., theta=...)` |
| slow continuous orbit around a 3D scene | `self.begin_ambient_camera_rotation(rate=...)` / `self.stop_ambient_camera_rotation()` |

## "Multiple things happen together / in sequence / staggered"

| They said | Use |
|---|---|
| simultaneously | pass multiple animations to one `self.play(anim1, anim2, ...)` |
| one after another, but as a single beat | `Succession(anim1, anim2, ...)` |
| staggered start (cascading effect) | `LaggedStart(*anims, lag_ratio=0.1)`, or `LaggedStartMap(AnimClass, group_of_mobjects, lag_ratio=0.1)` |
| grouped so they can be moved/animated as one unit | wrap mobjects in `VGroup(...)` first |

## Plotting / graphs / data

- Coordinate axes: `Axes(x_range=[...], y_range=[...])`, `NumberPlane()` (gridlines), `ComplexPlane()`, `PolarPlane()`, `ThreeDAxes()`.
- Plot a function: `axes.plot(lambda x: ...)`.
- Label a graph: `axes.get_graph_label(graph, "f(x)")`.
- Bar chart: `BarChart(values=[...])`.
- A dot/marker that tracks a point on a graph as a `ValueTracker` changes — see the counting pattern above, using `axes.c2p(x, y)` (coords-to-point) inside the updater.

## Common structural gotchas

- **`.animate` vs explicit `Animation`**: `.animate.rotate(PI)` interpolates the mobject's *start state* directly toward its *end state* — for a 180° rotation the start and end look identical, so `.animate` may produce a shrink/degenerate interpolation instead of visibly rotating. Use `Rotate(mobj, angle=PI)` for full or half rotations, reserve `.animate.rotate(...)` for smaller angles.
- **`Transform` vs `ReplacementTransform`**: `Transform(a, b)` still leaves `a` as the object in the scene (now looking like `b`) — keep referring to `a` afterward. `ReplacementTransform(a, b)` actually swaps `b` into the scene — refer to `b` afterward. Either works for a single morph; `ReplacementTransform` reads more clearly when chaining distinct named objects.
- Set a mobject's final visual properties (color, fill_opacity, position) *before* animating its creation, unless the animation's whole point is that property changing.

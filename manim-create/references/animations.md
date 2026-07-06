# Animation catalog

Every animation is something passed to `self.play(...)`. Grouped by category.

## Creation / removal

`Create`, `Uncreate`, `DrawBorderThenFill`, `Write` (handwriting-style, for text/Tex), `Unwrite`, `AddTextLetterByLetter`, `AddTextWordByWord`, `RemoveTextLetterByLetter`, `TypeWithCursor`, `UntypeWithCursor`, `ShowPartial`, `ShowIncreasingSubsets`, `ShowSubmobjectsOneByOne`, `SpiralIn`.

## Fading & growing

`FadeIn(mobj, shift=UP)` (optional `shift` gives a fade+slide effect), `FadeOut`, `GrowFromCenter`, `GrowFromPoint`, `GrowFromEdge`, `GrowArrow`, `SpinInFromNothing`.

## Transform family

`Transform(a, b)`, `ReplacementTransform(a, b)`, `TransformFromCopy(a, b)` (leaves `a` in place, animates a copy into `b`), `MoveToTarget(mobj)` (paired with `mobj.generate_target()` then modifying `mobj.target`), `ClockwiseTransform`, `CounterclockwiseTransform`, `CyclicReplace(a, b, c, ...)` (rotates mobjects into each other's positions), `FadeTransform`, `FadeTransformPieces`, `FadeToColor(mobj, color)`, `ScaleInPlace`, `ShrinkToCenter`, `Restore(mobj)` (paired with an earlier `mobj.save_state()`), `Swap`, `TransformAnimations`.

Matching-parts variants (for text/Tex where corresponding letters/terms should glide to their new position rather than cross-fade): `TransformMatchingShapes`, `TransformMatchingTex`.

Function-based: `ApplyFunction`, `ApplyMethod`, `ApplyMatrix`, `ApplyComplexFunction`, `ApplyPointwiseFunction`, `ApplyPointwiseFunctionToCenter`.

## Movement

`MoveAlongPath(mobj, path)`, `Homotopy`, `ComplexHomotopy`, `PhaseFlow`, `SmoothedVectorizedHomotopy`.

## Rotation

`Rotate(mobj, angle=...)` (one-shot, angle-based — prefer over `.animate.rotate()` for large/symmetric angles), `Rotating(mobj, radians=..., about_point=...)` (continuous, good for "spins" over a duration).

## Indication / highlighting

`Circumscribe(mobj)`, `Indicate(mobj)`, `Flash(point_or_mobj)`, `Wiggle(mobj)`, `ApplyWave(mobj)`, `FocusOn(point)`, `ShowPassingFlash(mobj)`, `ShowPassingFlashWithThinningStrokeWidth`, `Blink(mobj)`.

## Numbers

`ChangingDecimal`, `ChangeDecimalToValue` — animate a `DecimalNumber`'s displayed value directly (alternative to the `ValueTracker` + updater pattern for simple counters).

## Composition & timing

- `AnimationGroup(*anims, lag_ratio=0)` — bundle animations, optionally staggered.
- `LaggedStart(*anims, lag_ratio=0.1)` — cascading start times.
- `LaggedStartMap(AnimClass, mobject_group, lag_ratio=0.1)` — apply the same animation type across a group with a stagger, without writing out each call.
- `Succession(*anims)` — plays as one `self.play()` call but animations run one after another rather than simultaneously.
- `ChangeSpeed(anim, speedinfo={...})` — remap an animation's speed over its duration.
- `Wait` — same as `self.wait()`, rarely constructed directly.

## Updaters (continuous, not a single `self.play` animation)

Not technically "Animation" objects — these attach a per-frame function to a mobject:

```python
mobj.add_updater(lambda m, dt: ...)   # dt = seconds since last frame, for physics-like continuous motion
mobj.add_updater(lambda m: m.move_to(other.get_center()))  # no dt: recompute from current state each frame
mobj.remove_updater(fn)
always_redraw(lambda: SomeMobject(...))  # returns a mobject that rebuilds itself every frame from the given function
```

Useful updater helpers: `MaintainPositionRelativeTo`, `UpdateFromFunc`, `UpdateFromAlphaFunc`.

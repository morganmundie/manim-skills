# Creative direction: inventing a visual worth watching

The most common failure mode of generated manim scenes: the first idea is a **cliché** — a sine wave on bare axes, a generic bar chart, a circle morphing into a square, bullet points fading in. Technically correct, visually dead, and it illustrates nothing specific about the concept.

**The cliché test**: if your storyboard would work equally well for a *different* topic, it's too generic. A good scene shows the *mechanism* of this particular concept — the thing that makes it tick — not a stock image that gestures at it.

## Ideation questions (answer these before storyboarding)

1. **What's the concrete instance?** Don't animate "sorting" — animate *these 12 numbers* getting sorted. Don't animate "attention" — animate *this sentence's* tokens attending to each other. Pick real values, and compute the real result in Python inside `construct()` (see "Real computation" below).
2. **What's the process to witness?** The viewer should watch something *happen* — a signal propagate, a quantity decompose, an approximation sharpen, an algorithm make its choices — not read a labeled diagram of the end state.
3. **What single quantity varies?** That's your `ValueTracker`. The most memorable manim scenes are "turn one knob, watch everything respond."
4. **What physical objects live in this world?** A laser, a chip, a switch, a neuron, a wire. Build small icons for them from primitives (see "Build the world") instead of writing the word in a box.
5. **What's the "aha" beat?** Everything before it is setup; slow down (`run_time=2`–`3`) and clear visual clutter when it lands.

## Signature moves

These are the recurring techniques behind high-craft explainers (3blue1brown's videos are the reference point). Each is a pattern to adapt, not a snippet to paste.

### One driver, many synced views

One `ValueTracker` drives several *simultaneous* representations of the same quantity: the physical thing, a measurement inset, and a live number/formula. The viewer sees they're the same thing because they move together.

```python
theta = ValueTracker(60 * DEGREES)

# view 1: the thing itself (e.g. a rotating polarization vector)
vect = always_redraw(lambda: Vector(
    np.array([np.cos(theta.get_value()), np.sin(theta.get_value()), 0]),
    color=BLUE,
))
# view 2: the measurement (angle arc + label that tracks it)
arc = always_redraw(lambda: Arc(0, theta.get_value(), radius=0.4, color=YELLOW))
# view 3: the live number
readout = DecimalNumber(unit=r"^\circ").add_updater(
    lambda d: d.set_value(theta.get_value() / DEGREES).to_corner(UR)
)

self.add(vect, arc, readout)
self.play(theta.animate.set_value(10 * DEGREES), run_time=3)
self.play(theta.animate.set_value(80 * DEGREES), run_time=3)
```

In a `ThreeDScene`, put the inset views in a screen-fixed HUD with `self.add_fixed_in_frame_mobjects(...)` while the 3D content moves behind them — a 3D wave with a 2D "sample plane" inset in the corner is far more informative than either alone.

### Real computation in the scene

Compute the actual math/data with Python/numpy inside `construct()` and animate *that*, instead of hand-placing a fake example. A scene that finds two subsets with equal sums by brute force, trains 30 steps of real gradient descent, or softmaxes a real matrix is automatically specific, correct, and re-randomizable. This is the single highest-leverage move against "simple known examples."

```python
values = random.sample(range(1, 100), 8)          # real data
match = find_equal_sum_subsets(values)            # real algorithm, run for real
# ...then build mobjects FROM the computed result
```

### Diagram → equation

Don't `Write` a formula next to a diagram — fly copies of the diagram's labeled parts *into* the formula's slots, so each symbol visibly *is* the quantity it came from. Build the `MathTex` from indexable parts:

```python
equation = MathTex(r"\text{Area}", "=", r"\tfrac{\pi}{2}R", r"\times", r"2\pi R")
equation.to_edge(UP)
self.play(
    Write(equation[0]), Write(equation[1]), Write(equation[3]),
    TransformFromCopy(arc_label, equation[2]),     # label on the diagram
    TransformFromCopy(circumference_label, equation[4]),
)
```

### Montage of cases

After showing one example carefully, rapid-fire many randomized ones to convey generality — no `self.play`, just add/wait/remove:

```python
for _ in range(20):
    example = make_random_example()   # rebuilds mobjects from fresh random data
    self.add(example)
    self.wait(0.4)
    self.remove(example)
```

### Dim the rest

To focus attention, don't just point at the important thing — fade everything else:

```python
self.play(
    everything_else.animate.set_opacity(0.25),
    focus.animate.set_color(TEAL),
    Create(SurroundingRectangle(focus, color=TEAL, buff=0.1)),
)
```

### Pull out and return

Examine one component of a composite up close, then put it back — a "digression" the viewer doesn't have to track:

```python
self.play(
    part.animate.scale(1.5).move_to(2 * DOWN),
    rate_func=there_and_back_with_pause,
    run_time=4,
)
```

### Passing flash

Trace energy/signal/emphasis along any curve or outline without permanently changing it:

```python
self.play(ShowPassingFlash(
    wire.copy().set_stroke(YELLOW, width=6),
    time_width=0.3, run_time=1.5,
))
```

### Glow

ManimCE has no glow primitive; layer translucent copies at increasing stroke widths (for outlines) or stacked fading circles (for point sources):

```python
def glow_layers(mobj, color=TEAL, n=8, max_width=18, opacity=0.06):
    return VGroup(*[
        mobj.copy().set_fill(opacity=0).set_stroke(color, width=w, opacity=opacity)
        for w in np.linspace(3, max_width, n)
    ])

def glow_dot(point, color=YELLOW, max_radius=0.3, n=15):
    return VGroup(*[
        Circle(radius=max_radius * k / n, stroke_width=0,
               fill_color=color, fill_opacity=0.07).move_to(point)
        for k in range(n, 0, -1)
    ])
```

### Ambient life

A `dt`-updater that keeps things gently moving (drifting, oscillating, orbiting) makes a scene feel alive during exposition instead of freezing between beats:

```python
def add_wobble(mobj, amplitude=0.05, speed=2.0):
    mobj.base = mobj.get_center()
    mobj.phase = np.random.uniform(0, TAU)
    def wobble(m, dt):
        m.phase += dt * speed
        m.move_to(m.base + amplitude * np.array(
            [np.cos(m.phase), np.sin(1.7 * m.phase), 0]))
    mobj.add_updater(wobble)
```

Used per-piece on a group (each with its own random phase/velocity), this is how "superposition"-style jitter clouds are made.

### Camera as narrator (3D)

In `ThreeDScene`, choreograph slow camera moves *during* the action, not just between beats — a 6–12 s `self.move_camera(phi=..., theta=..., run_time=8)` overlapping other animations reads as a guided tour. `self.begin_ambient_camera_rotation(rate=0.02)` during exposition keeps 3D legible.

### Progressive refinement via subclass config

For "now with more slices/steps/neurons" sequences, parameterize the scene with class attributes and subclass for variants — each renders separately:

```python
class CircleArea(Scene):
    n_slices = 12
    def construct(self):
        ...  # everything derived from self.n_slices

class CircleArea48(CircleArea):
    n_slices = 48
```

Same trick with `random_seed`-style attributes for re-rolled montages.

### Icons over labels ("build the world")

Write module-level helper functions that return small device icons composed from primitives — reusable across scenes in the file:

```python
def chip_icon(height=1.5, label="CPU"):
    body = RoundedRectangle(corner_radius=0.08, width=height, height=height)
    body.set_fill(GREY_D, 1).set_stroke(GREY_B, 2)
    pins = VGroup(*[
        Line(ORIGIN, 0.18 * direction).set_stroke(GREY_B, 3)
        for direction in [UP, DOWN, LEFT, RIGHT]
        for shift in [-0.4, 0, 0.4]
    ])  # position pins along each edge with .next_to/.shift as needed
    text = Text(label, font_size=24).move_to(body)
    return VGroup(body, pins, text)
```

Boolean ops help here: `Union(Square(), ArrowTip().move_to(...))` makes a "machine with input/output" silhouette as one fillable shape. See `references/domain_recipes.md` for per-domain icon recipes.

## Structural habits worth copying

- **Many small `Scene` classes per file**, one per narrative beat, rather than one giant `construct()` — each previews and re-renders independently.
- **Module-level helper functions** for anything used twice (icons, network builders, slice generators).
- **`# comment` headers inside `construct()`** marking each beat — they're the storyboard.

## Caution: 3b1b's repo is manimgl, not ManimCE

If you look at 3blue1brown's published scene code (github.com/3b1b/videos) for inspiration — which is a great idea — remember it uses his personal library **manimgl**, whose API differs from ManimCE. Steal the *choreography*, translate the *names*:

| manimgl | ManimCE |
|---|---|
| `ShowCreation` | `Create` |
| `VShowPassingFlash` | `ShowPassingFlash` |
| `Tex` (math mode!) / `TexText` | `MathTex` / `Tex` |
| `InteractiveScene` | `Scene` |
| `self.frame.reorient(...)` | `ThreeDScene.move_camera(phi=..., theta=...)` or `MovingCameraScene`'s `self.camera.frame` |
| `mobj.fix_in_frame()` | `self.add_fixed_in_frame_mobjects(mobj)` (ThreeDScene) |
| `GlowDot` | layered-circle helper (see Glow above) |
| `mobj.replicate(n)` | `VGroup(*[mobj.copy() for _ in range(n)])` |
| `VGroup(gen for x in ...)` | `VGroup(*[... for x in ...])` (unpack a list) |
| `Randolph`/`Mortimer`, `TeacherStudentsScene` | not available — omit the pi creatures |
| `TimeVaryingVectorField` | `ArrowVectorField`/`StreamLines` + updaters |

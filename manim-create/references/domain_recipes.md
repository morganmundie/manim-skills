# Domain recipes: electricity, computer systems, machine learning, math

Starting-point patterns for the domains this skill is most used for. Each recipe is a sketch to adapt — combine with the moves in `references/creative_direction.md` (one driver/many views, real computation, glow, passing flash, montage).

## Electricity & electromagnetism

### Traveling wave (the right way)

Never plot a static sine curve — make it propagate. A phase `ValueTracker` + `always_redraw` gives a live wave any other element can sync to:

```python
axes = Axes(x_range=[0, 10], y_range=[-1.5, 1.5])
phase = ValueTracker(0)
wave = always_redraw(lambda: axes.plot(
    lambda x: np.sin(2 * x - phase.get_value()), color=BLUE))
self.add(axes, wave)
self.play(phase.animate.set_value(4 * TAU), run_time=6, rate_func=linear)
```

Sample the field with vectors riding the wave (draw the *field*, not just its envelope):

```python
arrows = always_redraw(lambda: VGroup(*[
    Arrow(axes.c2p(x, 0), axes.c2p(x, np.sin(2 * x - phase.get_value())),
          buff=0, stroke_width=3, color=BLUE_B, max_tip_length_to_length_ratio=0.15)
    for x in np.arange(0.5, 10, 0.5)
]))
```

### 3D EM wave (E ⊥ B)

In `ThreeDScene`, two perpendicular `ParametricFunction` waves sharing one phase tracker — the classic light-wave shot. Add a slow `self.move_camera(theta=...)` orbit while it propagates:

```python
e_wave = always_redraw(lambda: ParametricFunction(
    lambda t: axes.c2p(t, np.sin(2 * t - phase.get_value()), 0),
    t_range=[0, 10], color=BLUE))
b_wave = always_redraw(lambda: ParametricFunction(
    lambda t: axes.c2p(t, 0, np.sin(2 * t - phase.get_value())),
    t_range=[0, 10], color=YELLOW))
```

Pair with a screen-fixed 2D inset (`self.add_fixed_in_frame_mobjects`) showing the field vector at one sample point — the "one driver, many views" move.

### Fields of charges

`ArrowVectorField` for a snapshot, `StreamLines` for flow. Compute the field for real from charge positions:

```python
charges = [(np.array([-2, 0, 0]), +1), (np.array([2, 0, 0]), -1)]
def field(p):
    total = np.zeros(3)
    for pos, q in charges:
        r = p - pos
        total += q * r / (np.linalg.norm(r) ** 3 + 0.1)
    return total

self.add(ArrowVectorField(field))
# or, for animated flow:
stream = StreamLines(field, stroke_width=2, max_anchors_per_line=30)
self.add(stream)
stream.start_animation(warm_up=False, flow_speed=1.5)
self.wait(4)
```

Mark charges with the `glow_dot` helper (creative_direction.md) — a glowing point reads as "charge/energy" far better than a labeled dot.

### Circuits from primitives

Build component icons once as helper functions, place them on wire paths:

```python
def resistor(width=1.2, height=0.35, n_zags=6):
    xs = np.linspace(0, width, 2 * n_zags)
    points = [ORIGIN] + [
        np.array([x, height / 2 * (1 if i % 2 else -1), 0])
        for i, x in enumerate(xs[1:-1], start=1)
    ] + [np.array([width, 0, 0])]
    return VMobject(stroke_color=WHITE).set_points_as_corners(points)

def battery(height=0.8):
    long_plate = Line(UP * height / 2, DOWN * height / 2, stroke_width=4)
    short_plate = Line(UP * height / 4, DOWN * height / 4, stroke_width=8)
    short_plate.next_to(long_plate, RIGHT, buff=0.15)
    return VGroup(long_plate, short_plate)
```

A capacitor is two parallel equal plates; a bulb is `Circle` + a small cross; wires are `Line`s or a single closed `VMobject` loop.

### Current flow

Two options, by mood:
- **A pulse**: `ShowPassingFlash(wire.copy().set_stroke(YELLOW, 6), time_width=0.3)` — one packet of energy.
- **Continuous flow**: electrons as dots cycling around the loop with a dt-updater:

```python
loop = wire_path  # a closed VMobject
electrons = VGroup(*[Dot(radius=0.05, color=YELLOW) for _ in range(12)])
for i, e in enumerate(electrons):
    e.alpha = i / len(electrons)
def circulate(group, dt, speed=0.1):
    for e in group:
        e.alpha = (e.alpha + dt * speed) % 1
        e.move_to(loop.point_from_proportion(e.alpha))
electrons.add_updater(circulate)
self.add(electrons)
self.wait(4)  # they just keep flowing under any other animation
```

Tie `speed` to a `ValueTracker` for "voltage up → current up" moments, and drive a `DecimalNumber` ammeter readout from the same tracker.

## Computer systems

### Logic gates and signals

Gate icons from boolean ops (or labeled `RoundedRectangle`s when speed matters):

```python
def and_gate(h=1.0):
    body = Union(
        Rectangle(width=h / 2, height=h).shift(LEFT * h / 4),
        Circle(radius=h / 2),
    )
    return body.set_fill(GREY_D, 1).set_stroke(WHITE, 2)
```

Choreograph *evaluation*, not a static truth table: input bits (`Text("1")`) sit on input wires, a pulse flashes along each wire into the gate, the gate `Indicate`s, and the output bit fades in moving outward:

```python
self.play(*[ShowPassingFlash(w.copy().set_stroke(YELLOW, 6), time_width=0.4)
            for w in in_wires])
self.play(Indicate(gate), FadeIn(out_bit, shift=0.3 * RIGHT))
```

Then the zoom-out move: shrink the single gate to a corner while a full circuit (many gates wired together) `Write`s in — "this simple thing composes into that".

### Memory, registers, data movement

A memory bank is squares in a grid with bit contents; data movement is `TransformFromCopy` from source cells to destination (the copy *travels*):

```python
cells = VGroup(*[Square(0.55, stroke_width=1) for _ in range(32)])
cells.arrange_in_grid(rows=4, buff=0)
bits = VGroup(*[Text(str(random.randint(0, 1)), font_size=28).move_to(c)
                for c in cells])
# read: highlight, then fly a copy into the register
self.play(Indicate(VGroup(*bits[8:16])))
self.play(TransformFromCopy(VGroup(*bits[8:16]), register_bits))
```

Address/label the rows, dim inactive regions (`set_opacity(0.25)`), and use `SurroundingRectangle` as the read/write head.

### Pipelines, packets, dataflow

Stages as labeled `RoundedRectangle`s connected by `Arrow`s; work items as small labeled chips flowing through with `LaggedStart` so several are in flight at once — that's what makes it read as a *pipeline* rather than a flowchart:

```python
self.play(LaggedStart(*[
    Succession(*[item.animate.move_to(stage) for stage in stages])
    for item in items
], lag_ratio=0.25, run_time=8))
```

Same skeleton covers network hops (nodes = circles, packet = glowing dot along edges), instruction cycles, and request/response diagrams. For a counter or clock, `Integer` + `ValueTracker`; render binary with a `Text` updater: `lambda t: t.become(Text(format(int(tracker.get_value()), "08b")))`.

### Code execution

`Code(...)` renders a syntax-highlighted program; animate execution by sliding a translucent highlight bar line to line while the state (variables, stack, memory grid) updates beside it. The split view — code on the left, live state on the right — is the mechanism-level way to show "what a program does".

## Machine learning

### Neural networks — built from real weights

Build the network with a helper, and derive edge appearance from an actual weight matrix (real numpy, not decoration): sign → color, magnitude → width/opacity.

```python
def network(layer_sizes, weights, h_buff=2.0, v_buff=0.5):
    layers = VGroup(*[
        VGroup(*[Circle(radius=0.18, stroke_color=WHITE,
                        fill_color=BLACK, fill_opacity=1)
                 for _ in range(n)]).arrange(DOWN, buff=v_buff)
        for n in layer_sizes
    ]).arrange(RIGHT, buff=h_buff)
    edges = VGroup()
    for l1, l2, W in zip(layers, layers[1:], weights):
        for j, n2 in enumerate(l2):
            for i, n1 in enumerate(l1):
                w = W[j, i]
                edges.add(Line(
                    n1.get_center(), n2.get_center(), buff=0.18,
                    stroke_color=BLUE if w > 0 else RED,
                    stroke_width=3 * abs(w), stroke_opacity=min(1, abs(w)),
                ))
    return VGroup(edges, layers)
```

**Forward pass**: per layer, lagged passing flashes along the edges, then neurons fill with their (actually computed) activations:

```python
acts = sigmoid(W @ prev_acts + b)  # compute for real
self.play(LaggedStart(*[
    ShowPassingFlash(e.copy().set_stroke(YELLOW, 3), time_width=0.5)
    for e in layer_edges
], lag_ratio=0.02))
self.play(*[n.animate.set_fill(BLUE, opacity=float(a))
            for n, a in zip(neurons, acts)])
```

**Backprop** is the same choreography in reverse with a different flash color.

### Gradient descent — run it for real

Iterate real gradient steps in numpy, then animate the recorded trajectory. Show it on level sets (`ImplicitFunction`) or a 3D `Surface`, with a synced loss curve as a second view:

```python
w = np.array([2.5, 2.0])
trail = [w.copy()]
for _ in range(40):
    w = w - 0.1 * grad_f(w)   # real gradient of your real loss
    trail.append(w.copy())

levels = VGroup(*[ImplicitFunction(lambda x, y, c=c: f(np.array([x, y])) - c,
                                   color=GREY_B) for c in [0.5, 1, 2, 4]])
dot = Dot(axes.c2p(*trail[0]), color=YELLOW)
path = VMobject().set_points_smoothly([axes.c2p(*p) for p in trail])
self.add(levels, TracedPath(dot.get_center, stroke_color=YELLOW), dot)
self.play(MoveAlongPath(dot, path), run_time=5, rate_func=linear)
```

Re-run from several random starts as a montage; contrast learning rates side by side.

### Attention / heatmaps

A matrix of values → grid of squares whose fill opacity *is* the (really computed, really softmaxed) value; token `Text`s label rows/columns; optionally draw token-to-token arcs with thickness proportional to weight:

```python
A = softmax(scores)  # real numbers from a real (tiny) example sentence
grid = VGroup(*[
    Square(0.5, stroke_width=1, stroke_color=GREY_D).set_fill(BLUE, opacity=float(v))
    for v in A.flatten()
]).arrange_in_grid(rows=len(tokens), buff=0)
```

Animate one query row at a time — dim the rest of the matrix, flash the arcs from that token to the ones it attends to.

### Convolution, embeddings, training curves

- **Convolution**: image as a grid of grey-filled squares (pixel values), a kernel-sized `SurroundingRectangle` sliding position to position while output cells fill in with the real dot products.
- **Embeddings/clustering**: `Dot`s at real (or PCA-projected) coordinates on a `NumberPlane`; morph a decision boundary (`ImplicitFunction`) between training steps; labels attach with updaters so they follow their points.
- **Training curve**: don't just `Create` a loss curve — draw it in sync with something happening (network edges re-coloring, boundary morphing, samples re-classifying) driven by one shared tracker.

## Math

- **Diagram → equation** (the core move): build `MathTex` from indexable parts and `TransformFromCopy` labeled quantities from the figure into their slots in the formula. See creative_direction.md for the snippet.
- **Rearrangement proofs**: cut a shape into pieces and physically re-assemble them (circle → `AnnularSector`s interleaved into a near-rectangle; square rearrangements for the Pythagorean theorem). Give each piece a `generate_target()` in its destination arrangement, then `LaggedStartMap(MoveToTarget, pieces)`. Alternate two fill shades (`BLUE_D`/`BLUE_E`) so pieces stay trackable mid-flight.
- **Progressive refinement**: parameterize slice/step count as a class attribute and subclass (`n_slices = 12` → `48` → `100`) to render the "finer and finer" sequence — see creative_direction.md.
- **Numbers as objects**: for combinatorics/number theory, a grid of `Integer`s (`arrange_in_grid`) *is* the world: sample from it, highlight subsets with `SurroundingRectangle`s + dim-the-rest, fly chosen numbers down into an equation built from the same mobjects, and compute the interesting structure (equal sums, primes, collisions) for real so the example is honest.
- **Sweep to measure**: to show a length/width/area varying across a shape, animate a measuring `Line` with an updater sweeping through it (driven by a `ValueTracker`), rather than annotating a static dimension.

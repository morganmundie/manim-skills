# Analog circuits in ManimCE: the manim-circuit plugin & hand-rolled primitives

Two paths, freely mixable: the **manim-circuit plugin** for proper schematic symbols with minimal code, and **hand-rolled primitives** for stylized/animated components (glowing bulbs, closing switches) and full geometric control. Everything below was verified against manim-circuit 0.0.3 on ManimCE 0.20.

## Path 1: the manim-circuit plugin

```
uv add manim-circuit
```

```python
from manim import *
from manim_circuit import *   # re-exports manim, plus the circuit classes
```

### Components (verified signatures)

| Class | Signature | Terminals via `.get_terminals(...)` |
|---|---|---|
| `Resistor` | `(label=None, direction=DOWN)` | `"left"`, `"right"` |
| `Inductor` | `(label=None, direction=DOWN)` | `"left"`, `"right"` |
| `Capacitor` | `(label=None, direction=DOWN, polarized=False)` | `"left"`, `"right"` |
| `VoltageSource` | `(value=1, label=True, direction=LEFT, dependent=True)` | `"positive"`, `"negative"` |
| `CurrentSource` | `(value=1, label=True, direction=LEFT, dependent=True)` | `"positive"`, `"negative"` |
| `Ground` | `(ground_type="ground", label=None)` | no argument: `ground.get_terminals()` |
| `Opamp` | `(bias_supply=None, label=False)` | `"positive_input"`, `"negative_input"`, `"output"`, `"positive_bias"`, `"negative_bias"` |

`direction` positions the label relative to the body. Components support `.rotate(angle)` and `.shift()`/`.move_to()` as usual; `get_terminals()` returns *current* coordinates, so position first, wire second.

### The `Circuit` class — wiring done for you

```python
class RCLoop(Scene):
    def construct(self):
        source = VoltageSource(value="5", direction=LEFT, dependent=False).shift(LEFT * 3)
        r = Resistor(label="10", direction=UP).shift(UP * 1.8)
        c = Capacitor(label="100").rotate(-PI / 2).shift(RIGHT * 3)
        ground = Ground().shift(DOWN * 2.4)

        circuit = Circuit()
        circuit.add_components(source, r, c, ground)
        circuit.add_wire(source.get_terminals("positive"), r.get_terminals("left"))
        circuit.add_wire(r.get_terminals("right"), c.get_terminals("left"))
        circuit.add_wire(c.get_terminals("right"), source.get_terminals("negative"))
        circuit.add_wire(source.get_terminals("negative"), ground.get_terminals())
        self.add(circuit)
```

`add_wire(end1, end2)` auto-inserts one right-angle elbow when the endpoints share neither x nor y (pass `invert=True` to flip which corner it turns at, `diagonal=True` to allow a straight diagonal), and automatically adds junction `Dot`s where wires meet. Animate construction with `Create(circuit)` or per-piece by adding components/wires in separate `self.play` calls.

### Verified gotchas

- **`dependent=True` is the default** for sources and draws the *diamond* (dependent-source) symbol. For an ordinary battery-like source you almost always want `dependent=False` (circle symbol).
- **Labels render with LaTeX** (`Tex`; `Resistor(label="10")` typesets "10 Ω") — so labels require a LaTeX install. No LaTeX? Pass no label and place your own `Text` beside the component.
- Components are fairly **small at default scale** relative to a 14-unit-wide frame — `.scale(1.5)` or keep the layout tight (~3 units between components).
- The plugin is young (v0.0.3): treat exotic layouts with suspicion and check the still frame. Its PyPI metadata targets Python 3.9–3.12 but it installs and runs fine on newer interpreters.
- Alternative plugin: `manim-eng` (v0.1.0) has a richer `Circuit.connect()` terminal API and many more component types (diodes, transistor-era switches, European symbols), but it's a single early release — reach for it only if manim-circuit lacks a symbol you need and you can't build it from primitives.

## Path 2: hand-rolled primitives

### The occluder pattern

Every component gets an opaque backplate so it can sit *on top of* a wire (and hide electrons passing "through" it) — this is what makes the loop-first layout below work:

```python
def occluder(w, h):
    return Rectangle(width=w, height=h, stroke_width=0,
                     fill_color=BLACK, fill_opacity=1)

def resistor(width=1.2, height=0.35, n_zags=6):
    xs = np.linspace(-width / 2, width / 2, 2 * n_zags)
    points = [np.array([-width / 2, 0, 0])] + [
        np.array([x, height / 2 * (1 if i % 2 else -1), 0])
        for i, x in enumerate(xs[1:-1], start=1)
    ] + [np.array([width / 2, 0, 0])]
    zigzag = VMobject(stroke_color=WHITE, stroke_width=3).set_points_as_corners(points)
    return VGroup(occluder(width, height), zigzag)

def battery(h=0.9, gap=0.3):
    plus_plate = Line(UP * h / 2, DOWN * h / 2, stroke_width=3).shift(RIGHT * gap / 2)
    minus_plate = Line(UP * h / 5, DOWN * h / 5, stroke_width=6).shift(LEFT * gap / 2)
    plus = Text("+", font_size=26).move_to([gap / 2 + 0.3, h / 2, 0])
    return VGroup(occluder(gap + 0.1, h), plus_plate, minus_plate, plus)

def bulb(r=0.3):
    globe = Circle(radius=r, stroke_color=WHITE, stroke_width=3,
                   fill_color=BLACK, fill_opacity=1)
    x = VGroup(Line(UL, DR, stroke_width=2), Line(UR, DL, stroke_width=2)
               ).scale(r * 0.55).move_to(globe)
    return VGroup(globe, x)
```

More from the same toolbox: **capacitor** = two parallel `Line` plates with a gap (occluder behind); **inductor** = `VGroup(*[Arc(radius=r, start_angle=PI, angle=-PI).shift(RIGHT * 2 * r * i) for i in range(4)])`; **switch** = pivot `Dot` + contact `Dot` + a blade `Line` rotated open by ~35° — closing it is `Rotate(blade, -35 * DEGREES, about_point=pivot.get_center())`, the perfect "and now current flows" trigger; **LED** = triangle + bar + two tiny emission arrows that `FadeIn(shift=UR * 0.2)` when lit.

If you need transform-safe connection points on these (for Manhattan wiring between arbitrarily placed components), attach `VectorizedPoint` ports exactly as the logic-gate helpers do — see `references/logic_gates.md`.

### Loop-first layout — design the path, then decorate it

For any closed loop (the standard battery-resistor-bulb teaching circuit), don't wire component-to-component. Instead: **one closed rectangle is simultaneously the wire and the track everything rides on**, and components are dropped onto its edges. This kills the classic bug of an electron path that doesn't quite match the drawn wires. Render-verified scene:

```python
class SimpleLoop(Scene):
    def construct(self):
        # 1. design the loop first
        L, R, T, B = -3, 3, 2, -2
        loop = VMobject(stroke_color=GREY_B, stroke_width=3).set_points_as_corners(
            [[L, T, 0], [R, T, 0], [R, B, 0], [L, B, 0], [L, T, 0]])

        # 2. electrons ride the same path (added after the loop, before the
        #    components, so component backplates occlude them)
        speed = ValueTracker(0.05)
        electrons = VGroup(*[Dot(radius=0.05, color=YELLOW) for _ in range(16)])
        for i, e in enumerate(electrons):
            e.alpha = i / len(electrons)

        def circulate(group, dt):
            for e in group:
                e.alpha = (e.alpha + dt * speed.get_value()) % 1
                e.move_to(loop.point_from_proportion(e.alpha))

        circulate(electrons, 0)          # snap into place before first frame
        electrons.add_updater(circulate)

        # 3. drop components onto the loop, rotated to match their edge
        batt = battery().rotate(PI / 2).move_to([L, 0, 0])
        res = resistor().move_to([0, T, 0])
        lamp = bulb().move_to([R, 0, 0])
        self.add(loop, electrons, batt, res, lamp)
        self.wait(2)

        # 4. one tracker drives everything: speed + glow + ammeter
        glow = VGroup(*[
            Circle(radius=0.35 + 0.12 * i, stroke_width=0,
                   fill_color=YELLOW, fill_opacity=0.13 * (1 - i / 4))
            for i in range(4)
        ]).move_to(lamp)
        amps = DecimalNumber(0.5, num_decimal_places=1, font_size=30)
        amps.add_updater(lambda m: m.set_value(speed.get_value() * 10))
        readout = VGroup(amps, Text("A", font_size=26).next_to(amps, RIGHT, buff=0.1))
        readout.next_to(res, UP, buff=0.4)
        self.add(readout)
        self.play(speed.animate.set_value(0.18), FadeIn(glow), run_time=2)
        self.wait(2)
```

The moves worth reusing anywhere:

- **Circulating electrons** = per-dot `alpha` proportion + dt-updater + `point_from_proportion`. They keep flowing underneath any other animation. The dot spacing is uniform because alphas are `i / n`.
- **One `ValueTracker`, many views**: `speed` drives electron velocity, the bulb glow, and the `DecimalNumber` ammeter at once — the "voltage up → everything responds" moment is one `self.play(speed.animate.set_value(...))`.
- **A current *pulse*** instead of continuous flow: `ShowPassingFlash(loop.copy().set_stroke(YELLOW, 6), time_width=0.3)`.
- Showing **electron flow vs conventional current**: electrons circulate one way, add an arrow labeled "I" pointing the other; say it on screen.

### Scene sketches for the classic requests

- **RC charging**: one tracker `t`; charge fraction `q = 1 - np.exp(-t / (R * C))` computed for real. Drive four synced views: `+`/`−` charge marks accumulating on capacitor plates (`FadeIn` marks as thresholds pass, or opacity ∝ `q`), electron `speed` proportional to `np.exp(-t / (R * C))` (current decays as it charges), a `DecimalNumber` voltmeter reading `q * V`, and a plotted `q(t)` curve revealed by `Create` at `rate_func=linear` alongside. The story is *current chokes off as voltage rises* — the electron slowdown is the aha.
- **Ohm's law / series-parallel**: same loop twice via scene subclassing (`R = 10` → `R = 20` class attribute, see manim-create's creative_direction on config variants); electrons visibly slower, meter halved. For parallel, two loops sharing the battery edge, electron streams splitting at a junction dot.
- **Switch scenes**: build the loop with a gap where the switch sits; electrons' updater gated on `switch_closed` flag or `speed` at 0 until `Rotate` closes the blade, then ramp speed up — cause, then effect, in that order.
- **AC**: make the electron speed *signed and oscillating* — keep a running clock inside the updater and the same `circulate` code sloshes the electrons back and forth:

  ```python
  clock = ValueTracker(0)
  clock.add_updater(lambda m, dt: m.increment_value(dt))
  self.add(clock)
  # inside circulate: e.alpha = (e.alpha + dt * 0.15 * np.sin(2 * clock.get_value())) % 1
  ```

  Pair it with a sine `always_redraw` plot sharing the same clock — the aha is "the electrons don't travel anywhere, the energy does".

For field-level views (charges, `StreamLines`, EM waves) see `references/domain_recipes.md` in the manim-create skill — this file stays at the schematic level.

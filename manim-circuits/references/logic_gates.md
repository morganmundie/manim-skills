# Logic gates & digital circuits in ManimCE

Everything here is plain ManimCE — no plugin, no LaTeX (`Text` only). The gate shapes are the IEEE "distinctive shape" symbols, built as closed `VMobject` paths, and every helper was render-verified as written.

## The gate helper module

Save this once per project (e.g. `gates.py`) and `from gates import *` in scenes. Design decisions baked in:

- **Ports are `VectorizedPoint`s inside the gate's `VGroup`**, so they follow every `shift`/`rotate`/`scale`, and `.copy()` preserves them. Read live positions with `gate.inputs[i].get_center()` and `gate.output.get_center()` — always at animation-build time, after all positioning.
- **Bodies have opaque fill** (`GREY_E`) so gates occlude wires and electrons passing behind them.
- **Input stubs start slightly *inside* the body** (`in_inset`) so the curved backs of OR/XOR stay visually connected to their input wires.
- Inverted variants (`NAND`/`NOR`/`XNOR`/`NOT`) get a small output bubble, and the output port shifts right past it automatically.

```python
from manim import *
import numpy as np

HIGH = YELLOW      # logic 1: wire color, bit color
LOW = GREY_D       # logic 0
WIRE = GREY_B      # un-driven wire
GATE_FILL = GREY_E # opaque fill so gate bodies occlude wires behind them
GATE_STROKE = WHITE


def _finish_gate(body, in_points, out_point, extras=None, stub=0.35, in_inset=0.25):
    """Attach input/output stub wires and port markers to a gate body."""
    stubs, gate, inputs = VGroup(), VGroup(), []
    for p in in_points:
        p = np.array(p, dtype=float)
        stubs.add(Line(p + RIGHT * in_inset, p + LEFT * stub,
                       stroke_color=WIRE, stroke_width=3))
        inputs.append(VectorizedPoint(p + LEFT * stub))
    out_point = np.array(out_point, dtype=float)
    stubs.add(Line(out_point, out_point + RIGHT * stub,
                   stroke_color=WIRE, stroke_width=3))
    out_port = VectorizedPoint(out_point + RIGHT * stub)
    gate.add(stubs, body)          # stubs first: the opaque body hides insets
    if extras:
        gate.add(*extras)
    gate.add(*inputs, out_port)
    gate.inputs, gate.output, gate.body = inputs, out_port, body
    return gate


def _bubble(tip, r=0.09):
    """Inversion bubble at a gate's output tip. Returns (circle, new_tip)."""
    c = Circle(radius=r, stroke_color=GATE_STROKE, stroke_width=2.5,
               fill_color=GATE_FILL, fill_opacity=1).move_to(tip + RIGHT * r)
    return c, tip + RIGHT * 2 * r


def and_gate(h=1.1, w=1.5, invert=False):
    r = h / 2
    body = VMobject(stroke_color=GATE_STROKE, stroke_width=2.5,
                    fill_color=GATE_FILL, fill_opacity=1)
    body.start_new_path(np.array([w / 2 - r, h / 2, 0]))
    body.add_line_to(np.array([-w / 2, h / 2, 0]))
    body.add_line_to(np.array([-w / 2, -h / 2, 0]))
    body.add_line_to(np.array([w / 2 - r, -h / 2, 0]))
    arc = ArcBetweenPoints(np.array([w / 2 - r, -h / 2, 0]),
                           np.array([w / 2 - r, h / 2, 0]), angle=PI)
    body.append_points(arc.points)   # arc starts exactly where the path ends
    ins = [[-w / 2, h / 4, 0], [-w / 2, -h / 4, 0]]
    out, extras = np.array([w / 2, 0, 0]), []
    if invert:
        bub, out = _bubble(out)
        extras.append(bub)
    return _finish_gate(body, ins, out, extras)


def or_gate(h=1.1, w=1.6, invert=False, xor=False):
    a = np.array([-w / 2, h / 2, 0])   # top-left
    b = np.array([w / 2, 0, 0])        # right tip
    c = np.array([-w / 2, -h / 2, 0])  # bottom-left
    top = ArcBetweenPoints(a, b, angle=-PI / 4)
    bot = ArcBetweenPoints(b, c, angle=-PI / 4)
    back = ArcBetweenPoints(c, a, angle=PI / 3)   # concave back edge
    body = VMobject(stroke_color=GATE_STROKE, stroke_width=2.5,
                    fill_color=GATE_FILL, fill_opacity=1)
    body.set_points(np.vstack([top.points, bot.points, back.points]))
    ins = [[-w / 2, h / 4, 0], [-w / 2, -h / 4, 0]]
    out, extras = b, []
    if xor:
        extra_back = ArcBetweenPoints(c, a, angle=PI / 3).shift(LEFT * 0.15)
        extras.append(extra_back.set_stroke(GATE_STROKE, 2.5))
    if invert:
        bub, out = _bubble(out)
        extras.append(bub)
    return _finish_gate(body, ins, out, extras,
                        stub=0.5 if xor else 0.35, in_inset=0.45)


def not_gate(h=0.9, w=1.0, invert=True):
    body = Polygon([-w / 2, h / 2, 0], [-w / 2, -h / 2, 0], [w / 2, 0, 0],
                   stroke_color=GATE_STROKE, stroke_width=2.5,
                   fill_color=GATE_FILL, fill_opacity=1)
    out, extras = np.array([w / 2, 0, 0]), []
    if invert:                    # invert=False gives a plain buffer
        bub, out = _bubble(out)
        extras.append(bub)
    return _finish_gate(body, [[-w / 2, 0, 0]], out, extras)


def nand_gate(**kw): return and_gate(invert=True, **kw)
def nor_gate(**kw):  return or_gate(invert=True, **kw)
def xor_gate(**kw):  return or_gate(xor=True, **kw)
def xnor_gate(**kw): return or_gate(xor=True, invert=True, **kw)
def buffer_gate(**kw): return not_gate(invert=False, **kw)


def wire(p, q, x_mid=None, color=WIRE):
    """Manhattan (right-angle) wire between two port coordinates.
    Splits the horizontal run at x_mid (default: halfway)."""
    p, q = np.array(p, dtype=float), np.array(q, dtype=float)
    if abs(p[1] - q[1]) < 1e-6 or abs(p[0] - q[0]) < 1e-6:
        pts = [p, q]
    else:
        if x_mid is None:
            x_mid = (p[0] + q[0]) / 2
        pts = [p, [x_mid, p[1], 0], [x_mid, q[1], 0], q]
    return VMobject(stroke_color=color, stroke_width=3).set_points_as_corners(pts)


def junction(at, color=WIRE):
    """Solder dot marking an electrically connected T-joint (fan-out)."""
    return Dot(at, radius=0.055, color=color)


def bit_label(value, at, direction=UP):
    return Text(str(int(bool(value))), font_size=30,
                color=HIGH if value else LOW).next_to(at, direction, buff=0.15)
```

Notes on adapting it:

- **Gate labels**: the distinctive shapes are self-identifying, but for teaching add `Text("AND", font_size=24).move_to(gate.body)` on top (the fill is opaque, so it reads fine).
- **More inputs**: pass more entries in `ins` (e.g. 3-input AND at `y = ±h/3, 0`). Keep inputs on the left edge.
- **Orientation**: gates point right by default. `gate.rotate(PI / 2)` makes it point up and the ports follow — but rotate *before* reading port positions for wiring.
- **Sanity-check ports** while developing: `self.add(*[Dot(p.get_center(), radius=0.04, color=RED) for p in gate.inputs])` on a `-s -ql` still.

## Signal choreography

Two distinct visual vocabularies — use both, for different things:

- **Level** (steady state): the wire's stroke color *is* its logic value. `w.animate.set_stroke(HIGH if v else LOW)`, with a `bit_label` near the wire end. This is what persists on screen.
- **Edge/event** (the moment of change): a pulse racing along the wire — `ShowPassingFlash(w.copy().set_stroke(HIGH, 6), time_width=0.4)`. Flash only wires carrying a 1 (a burst of nothing reads wrong), and flash *into* a gate before it reacts.

The gate "fires" with `Indicate(gate.body, scale_factor=1.06)` — note `Indicate` also flashes the fill bright yellow, which reads as the gate lighting up; if that's too loud use `Indicate(gate.body, color=WHITE)` or `Circumscribe(gate.body)`.

The evaluation beat, in causal order (this exact sequence was render-verified in the SKILL.md example):

1. Input bits + input wire levels change (one `self.play`, ~0.6s).
2. Pulses flash along the HIGH input wires into the gate (~0.7s).
3. Gate indicates; output wire level + output bit settle (one `self.play`, ~0.8s).

**Always compute the logic in Python** (`q = a & b`, `s = a ^ b`) and derive every color, bit, and table row from those variables. The animation can then never disagree with the truth.

## Multi-gate circuits: fan-out and propagation order

- **Fan-out**: run the source wire to a `junction()` dot, then separate `wire()` calls from the junction to each destination port. Choose distinct `x_mid` values per branch so parallel elbows don't overlap.
- **Propagation**: animate gates in topological order (inputs → first layer → second layer). One `self.play` per layer, so causality sweeps left to right. For a long combinational path, `LaggedStart` the per-gate beats with `lag_ratio≈0.6`.
- **Layout**: inputs on the left edge as labeled rails, outputs on the right, gate layers in vertical columns. Space input rails ≥ 0.8 units apart or their bit labels collide with each other and with wires (this bit *will* bite — check the still frame).

## Worked example: half adder cycling its truth table

XOR computes the sum, AND the carry; the scene cycles all four input combinations while a synced truth table highlights the active row. Render-verified end to end.

```python
from manim import *
import numpy as np
from gates import *

class HalfAdder(Scene):
    def construct(self):
        # schematic: XOR computes the sum bit, AND the carry bit
        g_xor = xor_gate().shift(RIGHT * 0.5 + UP * 1.2)
        g_and = and_gate().shift(RIGHT * 0.5 + DOWN * 1.2)
        xa, xb = g_xor.inputs[0].get_center(), g_xor.inputs[1].get_center()
        aa, ab = g_and.inputs[0].get_center(), g_and.inputs[1].get_center()

        a_src = np.array([-4.7, 2.2, 0])     # input rails, spread apart
        b_src = np.array([-4.7, 0.2, 0])
        ja = np.array([-3.4, a_src[1], 0])   # fan-out junctions
        jb = np.array([-2.9, b_src[1], 0])

        w_a = VGroup(wire(a_src, xa, x_mid=-2.0), wire(ja, aa, x_mid=-3.4))
        w_b = VGroup(wire(b_src, xb, x_mid=-1.6), wire(jb, ab, x_mid=-2.9))
        s_end = g_xor.output.get_center() + RIGHT
        c_end = g_and.output.get_center() + RIGHT
        w_s = wire(g_xor.output.get_center(), s_end)
        w_c = wire(g_and.output.get_center(), c_end)

        labels = VGroup(
            Text("A", font_size=30).next_to(a_src, LEFT, buff=0.2),
            Text("B", font_size=30).next_to(b_src, LEFT, buff=0.2),
            Text("S", font_size=30).next_to(s_end, RIGHT, buff=0.2),
            Text("C", font_size=30).next_to(c_end, RIGHT, buff=0.2),
        )
        self.add(w_a, w_b, w_s, w_c, junction(ja), junction(jb),
                 g_xor, g_and, labels)

        # truth table, computed for real from the same expressions
        combos = [(0, 0), (0, 1), (1, 0), (1, 1)]
        table = Table(
            [[str(a), str(b), str(a ^ b), str(a & b)] for a, b in combos],
            col_labels=[Text(s) for s in "ABSC"],
            element_to_mobject=Text,
            include_outer_lines=False,
        ).scale(0.35).to_edge(RIGHT, buff=0.4)
        self.play(FadeIn(table))

        # bits live just above their wire ends, clear of the name labels
        bit_a = bit_label(0, a_src + RIGHT * 0.4).set_opacity(0)
        bit_b = bit_label(0, b_src + RIGHT * 0.4).set_opacity(0)
        bit_s = bit_label(0, s_end + LEFT * 0.4).set_opacity(0)
        bit_c = bit_label(0, c_end + LEFT * 0.4).set_opacity(0)
        self.add(bit_a, bit_b, bit_s, bit_c)

        row_box = None
        for i, (a, b) in enumerate(combos):
            s, c = a ^ b, a & b  # compute the logic for real

            # inputs arrive: bits update, wires take their levels
            new_a, new_b = (bit_label(v, ORIGIN).move_to(m)
                            for v, m in [(a, bit_a), (b, bit_b)])
            box = SurroundingRectangle(table.get_rows()[i + 1], color=HIGH, buff=0.1)
            self.play(
                Transform(bit_a, new_a), Transform(bit_b, new_b),
                w_a.animate.set_stroke(HIGH if a else LOW),
                w_b.animate.set_stroke(HIGH if b else LOW),
                (Transform(row_box, box) if row_box else Create(box)),
                run_time=0.6,
            )
            row_box = row_box or box

            # signals race through the gates
            flashes = [
                ShowPassingFlash(w.copy().set_stroke(HIGH, 6), time_width=0.5)
                for grp, v in [(w_a, a), (w_b, b)] if v for w in grp
            ]
            if flashes:
                self.play(*flashes, run_time=0.7)

            # outputs settle
            new_s, new_c = (bit_label(v, ORIGIN).move_to(m)
                            for v, m in [(s, bit_s), (c, bit_c)])
            self.play(
                Indicate(g_xor.body, scale_factor=1.05),
                Indicate(g_and.body, scale_factor=1.05),
                w_s.animate.set_stroke(HIGH if s else LOW),
                w_c.animate.set_stroke(HIGH if c else LOW),
                Transform(bit_s, new_s), Transform(bit_c, new_c),
                run_time=0.8,
            )
            self.wait(0.6)
        self.wait()
```

Adaptation targets: full adder (two XOR, two AND, one OR — same beats, one more layer), 2-to-1 mux, decoder. The "shrink the single gate to a corner while the full circuit writes in" zoom-out move from manim-create's domain recipes pairs well as an opener.

## Clocks and sequential logic

- **Clock waveform**: build the square wave as one polyline and reveal it in sync with the circuit:

  ```python
  pts, level = [], 0
  for k in range(n_half_periods):
      x0, x1 = k * 0.8, (k + 1) * 0.8
      pts += [axes.c2p(x0, level), axes.c2p(x1, level)]
      level = 1 - level
      pts.append(axes.c2p(x1, level))
  clk = VMobject(stroke_color=BLUE).set_points_as_corners(pts[:-1])
  self.play(Create(clk), run_time=n_half_periods * 0.5, rate_func=linear)
  ```

  Run circuit beats *between* segments of this reveal (or drive both from one loop) so edges on the waveform line up with events in the circuit.
- **Flip-flop as a black box**: a `RoundedRectangle` with `Text` pins ("D", "Q", "CLK") and a small `Triangle` marker at the clock pin; ports via `VectorizedPoint`s exactly like the gates. On each *rising* edge: `Flash` the CLK pin, then transfer the D bit to Q with `TransformFromCopy(d_bit, q_bit)` — the copy visibly travels through the box. Keep Q frozen at every other moment; that stillness between edges *is* the concept.
- **SR latch from primitives**: two `nor_gate()`s, second one `.rotate(PI)` or mirrored layout, with feedback wires from each output to the other's input (choose `x_mid` outside the gate column so the feedback loops around). Settle it honestly: iterate `q, qn = 1 - (r | qn), 1 - (s | q)` in Python until stable, and animate one iteration per beat — for the metastable-looking cases that oscillation is the story.

## Buses and registers

A multi-bit value is a `VGroup` of bit `Text`s over a row of parallel wires (or one thick wire with a slash + `Text("8")` width marker, the standard schematic shorthand). Move data with `TransformFromCopy(src_bits, dst_bits)` so the copy travels; for memory grids and register-file choreography see `references/domain_recipes.md` in the manim-create skill ("Memory, registers, data movement").

---
name: manim-circuits
description: Build and animate electrical circuits and digital logic in Manim Community Edition (ManimCE) — analog schematics (resistors, capacitors, inductors, batteries, op-amps, current/electron flow, RC charging) and digital logic (AND/OR/NOT/NAND/NOR/XOR gates, truth tables, adders, latches, clock signals, binary buses). Use whenever a manim animation involves a circuit, schematic, logic gate, boolean logic, signal propagation, current, voltage, or digital hardware — "animate an RC circuit charging", "show how a NAND gate works", "visualize a half adder", "electrons flowing through a wire", "explain a flip-flop", "current through a resistor". This skill supplies the domain layer: render-tested component geometry (gate shapes from primitives, the manim-circuit plugin for analog parts), terminal-based wiring discipline, and signal choreography. Use it together with manim-create (overall scene workflow, creative direction) and manim-export (rendering the final video).
---

# Manim: Circuits & Logic Gates

Circuit scenes are different from general manim scenes in one structural way: everything is **connection topology plus flowing state**. A circuit diagram is a graph of components joined at exact terminal points, and the animation payoff is almost never the diagram itself — it's the *signal* moving through it (current circulating, a logic pulse racing through gates, a capacitor filling). Get the topology mechanically right and the choreography does the rest.

Follow the general manim-create workflow (uv project setup, clarify the idea, storyboard, preview with `uv run manim -pql`). This skill adds the circuit-specific layer.

## Four rules that prevent 90% of circuit-scene bugs

1. **Everything connects at terminals — never eyeball a junction.** Every component must expose its connection points programmatically (the helpers in the references attach `VectorizedPoint` ports that survive `shift`/`rotate`/`scale`/`copy`; the manim-circuit plugin has `.get_terminals()`). Wires are drawn *from terminal to terminal*. A wire that visibly misses a pin by 0.05 units destroys the "this is a real machine" illusion.
2. **Wires route Manhattan-style.** Real schematics use horizontal/vertical runs with right-angle elbows, never diagonals. Use the `wire()` helper (references) or the plugin's auto-elbowing `Circuit.add_wire()`. Mark electrically-connected T-joints with a solder `Dot` (a junction); crossings without a dot read as "not connected".
3. **Build the schematic as a correct still frame first, then animate.** Lay out every component and wire, render one frame with `-s -ql`, and check it *visually* before writing any `self.play` calls. Layout bugs are cheap to fix in a still and miserable to fix inside a finished animation. When something must flow along the circuit later (electrons, a pulse), derive its path from the same geometry the wires were built from — never a separately hardcoded path (they *will* drift apart).
4. **The signal is the show.** A static schematic is a textbook figure; the animation earns its existence when state moves: pulses flash along wires (`ShowPassingFlash`), wires change color with logic level, electrons circulate with a dt-updater, meters count, gates fire. Compute all of it for real in Python (`s, c = a ^ b, a & b`; `q = 1 - np.exp(-t / (R * C))`) — never hand-animate truth values or fake a waveform.

## Choosing your component source

| Situation | Use |
|---|---|
| **Digital logic** (gates, adders, latches, anything boolean) | Build gates from primitives with the helpers in `references/logic_gates.md`. There is **no maintained ManimCE logic-gate plugin** — circuitanim has gates but is ManimGL-only, and manim-eng is a single early release. The helpers are short, render-verified, and fully controllable. |
| **Analog schematic with standard parts** (R, L, C, sources, ground, op-amp) | `uv add manim-circuit` — a small ManimCE plugin with proper component symbols, `.get_terminals()`, and a `Circuit` class that auto-routes elbow wires with junction dots. See `references/analog_circuits.md` for its real API and gotchas (e.g. `dependent=False` for a normal round source symbol; labels require LaTeX). |
| **Creative/stylized circuits, or components the plugin lacks** (bulbs, switches, LEDs, custom looks) | Hand-rolled primitives in `references/analog_circuits.md` — zigzag resistor, battery plates, glowing bulb, plus the "design the loop first" pattern that makes electron flow trivial. |

Mixing is normal: plugin symbols for the schematic, hand-rolled glow and electron dots for the life.

## Defaults when the request doesn't specify

- **Logic level colors**: HIGH = `YELLOW`, LOW = `GREY_D`, undriven wire = `GREY_B`. Bits are `Text("1")`/`Text("0")` in the level color (plain `Text`, no LaTeX needed anywhere in the digital helpers).
- **Gate style**: white stroke ~2.5, opaque `GREY_E` fill (so bodies occlude wires passing behind), IEEE distinctive shapes (D-shape AND, shield OR, triangle-and-bubble NOT), ~1.1 units tall.
- **Current direction**: conventional current (+ → −) unless the user asks for electron flow; if the scene shows *electrons*, say so on screen, since they move opposite to conventional current.
- **Evaluation tempo**: one beat per event — inputs arrive (~0.6s), pulses propagate (~0.7s), gate fires and output settles (~0.8s), short `wait` to absorb. For multi-gate circuits, propagate in topological order so causality reads left to right.
- **Analog values**: pick clean round numbers (5 V, 10 Ω) and drive readouts (`DecimalNumber`) from the same `ValueTracker` that drives the visual flow.

## Reference files

- `references/logic_gates.md` — render-verified gate builders for all 8 gates (AND/OR/NOT/NAND/NOR/XOR/XNOR/buffer) with transform-safe ports, Manhattan `wire()` + `junction()` helpers, the gate-evaluation choreography, truth-table sync, and a complete worked half adder that cycles all four input combinations. Also: composing gates (fan-out, multi-gate propagation), clocks, and latch/flip-flop patterns.
- `references/analog_circuits.md` — the manim-circuit plugin's actual API (component signatures, terminal keys, `Circuit.add_wire`, verified gotchas), hand-rolled component primitives with the occluder pattern, the loop-first layout method, circulating-electron updater, glow, meters, and an RC-charging scene sketch.

## A minimal complete example, for calibration

A single AND gate evaluating `1 AND 1` (uses the helpers from `references/logic_gates.md`, assumed saved as `gates.py`):

```python
from manim import *
from gates import *  # and_gate, wire, bit_label, HIGH, LOW

class AndGateEval(Scene):
    def construct(self):
        gate = and_gate().scale(1.2)
        a_in, b_in = gate.inputs[0].get_center(), gate.inputs[1].get_center()
        out = gate.output.get_center()
        wa, wb = wire(a_in + LEFT * 2.5, a_in), wire(b_in + LEFT * 2.5, b_in)
        wq = wire(out, out + RIGHT * 2.5)
        self.add(wa, wb, wq, gate)

        a, b = 1, 1
        q = a & b  # compute the logic for real
        self.play(FadeIn(bit_label(a, wa.get_start()), bit_label(b, wb.get_start())))
        self.play(wa.animate.set_stroke(HIGH if a else LOW),
                  wb.animate.set_stroke(HIGH if b else LOW))
        self.play(*[ShowPassingFlash(w.copy().set_stroke(HIGH, 6), time_width=0.4)
                    for w, v in [(wa, a), (wb, b)] if v])
        self.play(Indicate(gate.body, scale_factor=1.06))
        self.play(wq.animate.set_stroke(HIGH if q else LOW),
                  FadeIn(bit_label(q, wq.get_end()), shift=0.3 * RIGHT))
        self.wait()
```

Preview: `uv run manim -pql <file>.py AndGateEval`. For final renders (quality, vertical formats, transparency) see the manim-export skill.

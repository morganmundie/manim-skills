# Colors & basic styling

## Built-in color constants

Imported automatically with `from manim import *`. Base names: `RED`, `GREEN`, `BLUE`, `YELLOW`, `PINK`, `PURPLE`, `ORANGE`, `TEAL`, `MAROON`, `GOLD`, `GRAY`/`GREY`, `WHITE`, `BLACK`.

Each has shade variants from lightest to darkest, e.g. for blue: `BLUE_A` (lightest), `BLUE_B`, `BLUE_C` (≈ base `BLUE`), `BLUE_D`, `BLUE_E` (darkest). Same pattern for the other base colors.

You can also use any hex string directly: `Circle(color="#87c2a5")`.

For a random color when it genuinely doesn't matter which: `random_color()` (rarely needed — usually pick something intentional instead).

## Fill vs stroke

Every `VMobject` (shapes, text, etc.) has independently controllable fill and stroke (outline):

```python
shape.set_fill(BLUE, opacity=0.5)   # fill color + opacity (0 = transparent, 1 = opaque)
shape.set_stroke(WHITE, width=4)    # outline color + thickness
shape.set_color(RED)                # sets both fill and stroke to the same color
```

Or pass at construction: `Circle(color=BLUE, fill_opacity=0.5, stroke_width=2)`.

## Background

```python
self.camera.background_color = "#ece6e2"   # set once in construct(), applies to the whole scene
```

(For a genuinely transparent background instead of a solid color, that's a render-time flag — see the manim-export skill, not something set in the scene code.)

## Positioning cheat-sheet

Directional constants for `shift`, `next_to`, `to_edge`, `to_corner`: `UP`, `DOWN`, `LEFT`, `RIGHT`, and combinations like `UL` (up-left), `UR`, `DL`, `DR`. Multiply for distance: `2 * UP`. `ORIGIN` = center of the frame.

```python
mobj.shift(RIGHT)                       # relative move
mobj.move_to(ORIGIN)                    # absolute move
mobj.next_to(other, DOWN, buff=0.5)     # relative to another mobject, with a gap
mobj.to_edge(UP)                        # snap to a frame edge
mobj.to_corner(UL)                      # snap to a frame corner
mobj.align_to(other, LEFT)              # align one edge without moving along the other axis
```

## Fonts & text sizing

`Text("...", font_size=48, font="Arial", weight=BOLD, slant=ITALIC)`. Default `font_size` is 48; scale relatively with `.scale(...)` if you need something to match another mobject's apparent size rather than guessing pixel sizes.

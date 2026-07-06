# Mobject catalog

Mobjects ("mathematical objects") are anything that can be shown/animated. Grouped by purpose — skim to the section you need rather than reading straight through.

## Basic shapes (`manim.mobject.geometry`)

- **Arcs & round things**: `Circle`, `Dot`, `AnnotationDot`, `Ellipse`, `Arc`, `ArcBetweenPoints`, `Annulus`, `AnnularSector`, `Sector`, `LabeledDot` (a `Dot` with a text label inside it).
- **Polygons**: `Square`, `Rectangle`, `RoundedRectangle`, `Triangle`, `Polygon`, `Polygram` (multiple disjoint polygons as one mobject), `RegularPolygon(n=...)`, `RegularPolygram`, `Star`.
- **Lines & vectors**: `Line`, `DashedLine`, `Arrow`, `DoubleArrow`, `Vector` (arrow rooted at origin), `Elbow`, `Angle`, `RightAngle`, `TangentLine`.
- **Boolean ops on shapes**: `Union`, `Intersection`, `Difference`, `Exclusion` — combine two shapes into a new one (e.g. `Intersection(circle1, circle2, color=GREEN)`).
- **Shape matchers / annotations**: `SurroundingRectangle(mobj)` (draws a box around any mobject), `BackgroundRectangle`, `Underline`, `Cross`.
- **Braces**: `Brace(mobj)` with `.get_text("label")` or `.get_tex(r"x_1")` for auto-positioned annotation labels; `BraceBetweenPoints`.

## Text & math

- `Text("plain text")` — regular text, any font, no LaTeX needed. Fastest to render.
- `MarkupText("<u>underlined</u> text")` — Pango markup for rich inline styling.
- `Paragraph("line one", "line two")` — multi-line text block.
- `Tex(r"\LaTeX\ text and \textbf{formatting}")` — LaTeX-rendered text (requires a LaTeX install); use for anything with math-adjacent typography even if it's not literally an equation.
- `MathTex(r"\sum_{n=1}^\infty \frac{1}{n^2} = \frac{\pi^2}{6}")` — LaTeX math mode. Split into multiple string arguments (`MathTex("a", "=", "b")`) to get indexable parts for `TransformMatchingTex` or per-part coloring/boxing (`text[1]`).
- `BulletedList(...)`, `Title(...)` — structured text helpers.
- `Code(...)` — syntax-highlighted source code block.
- `DecimalNumber`, `Integer`, `Variable` — numeric displays, typically driven by a `ValueTracker` (see vocabulary.md).

## Coordinate systems & plotting (`manim.mobject.graphing`)

- `Axes(x_range=[min, max, step], y_range=[...])` — the standard 2D axes; `.plot(function)`, `.get_axis_labels()`, `.get_graph_label()`, `.c2p(x, y)` / `.coords_to_point(x, y)` to convert data coords to screen points, `.get_area(...)`, `.get_riemann_rectangles(...)`, `.plot_line_graph(x_values=..., y_values=...)`.
- `NumberPlane()` — full gridlines, good background for "this is a coordinate plane" framing.
- `ComplexPlane()`, `PolarPlane()`, `ThreeDAxes()`.
- `NumberLine()` / `UnitInterval()` — a single axis.
- `BarChart(values=[...])`, `SampleSpace` — for statistics-flavored visuals.
- `FunctionGraph(function)`, `ParametricFunction(function)`, `ImplicitFunction(...)`.

## Groups & structure

- `VGroup(mobj1, mobj2, ...)` — group vector-based mobjects to move/animate/style together; supports `.arrange(RIGHT, buff=...)` to auto-layout children.
- `Group(...)` — like `VGroup` but for mixed mobject types (e.g. including `ImageMobject`).
- `VDict` — group with named/keyed access.

## Images & external assets

- `ImageMobject(path_or_array)` — raster images (also accepts a numpy array, e.g. for a synthetic gradient).
- `SVGMobject(path)` — vector graphics, editable as manim shapes.

## Tables & matrices

- `Table`, `MathTable`, `IntegerTable`, `DecimalTable`, `MobjectTable` — for tabular data.
- `Matrix`, `IntegerMatrix`, `DecimalMatrix`, `MobjectMatrix` — bracketed matrix notation.

## 3D (use with `ThreeDScene`)

- `Sphere`, `Cube`, `Cone`, `Cylinder`, `Torus`, `Prism`, `Dot3D`, `Line3D`, `Arrow3D`.
- `Surface(param_function, u_range=..., v_range=..., resolution=(nu, nv))` — parametric surfaces (e.g. `lambda u, v: np.array([...])`); supports `.set_fill_by_checkerboard(color1, color2)`.
- Polyhedra: `Tetrahedron`, `Cube`, `Octahedron`, `Dodecahedron`, `Icosahedron`, `ConvexHull3D`.

## Value drivers (not visible themselves)

- `ValueTracker(initial_value)` — the standard way to parameterize an animation by a number you can smoothly animate; drive other mobjects via `add_updater` reading `.get_value()`. See `references/vocabulary.md`.

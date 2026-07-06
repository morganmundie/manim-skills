# Choosing a Scene type

Default to plain `Scene`. Upgrade only when the idea specifically needs one of these:

## `Scene` (default)

Static camera, 2D. Use this unless something below clearly applies.

```python
class MyScene(Scene):
    def construct(self):
        ...
```

## `MovingCameraScene` — camera pans/zooms in 2D

Use when the idea involves the camera following action, zooming into detail, or panning across a larger layout.

```python
class MyScene(MovingCameraScene):
    def construct(self):
        self.camera.frame.save_state()  # so you can Restore(self.camera.frame) later
        circle = Circle()
        self.add(circle)
        self.play(self.camera.frame.animate.scale(0.5).move_to(circle))
        self.play(Restore(self.camera.frame))
```

To have the camera continuously follow a moving mobject, give `self.camera.frame` an `add_updater` that recenters on the moving mobject, same pattern as any other updater.

## `ZoomedScene` — a magnifying inset panel

Use when the idea is "zoom in on this detail while keeping the full picture visible" (picture-in-picture magnifier), not just "the camera moves closer."

```python
class MyScene(ZoomedScene):
    def __init__(self, **kwargs):
        super().__init__(zoom_factor=0.3, zoomed_display_height=1, zoomed_display_width=6, **kwargs)

    def construct(self):
        # position self.zoomed_camera.frame over the region to magnify, then:
        self.activate_zooming()
        self.play(self.get_zoomed_display_pop_out_animation())
```

## `ThreeDScene` — genuinely 3D content

Use for 3D shapes, surfaces, or camera orbiting around something in 3D space.

```python
class MyScene(ThreeDScene):
    def construct(self):
        axes = ThreeDAxes()
        self.set_camera_orientation(phi=75 * DEGREES, theta=30 * DEGREES)
        self.add(axes, Sphere())
        self.begin_ambient_camera_rotation(rate=0.1)  # slow continuous orbit
        self.wait(4)
        self.stop_ambient_camera_rotation()
```

- `phi` = angle down from the top (vertical tilt), `theta` = rotation around the vertical axis.
- `self.move_camera(phi=..., theta=..., run_time=...)` for a one-shot animated camera move (vs. the continuous `begin_ambient_camera_rotation`).
- 2D overlays (titles, labels that shouldn't rotate with the 3D scene) need `self.add_fixed_in_frame_mobjects(text_mobj)` before positioning them.
- `self.renderer.camera.light_source.move_to(...)` to reposition the light for shading on `Surface`/3D solids.

## `VectorScene` / `LinearTransformationScene`

Use only for linear-algebra-specific content (vectors in a plane, matrix transformations of the plane/grid) — these come with built-in helpers (`add_vector`, `apply_matrix` with automatic grid animation) that are overkill for anything else.

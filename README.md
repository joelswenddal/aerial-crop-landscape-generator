# Aerial Crop Field Generator

A perspective reference tool for generating abstract aerial agricultural landscape designs. The goal is to generate plausible, semi-abstract compositions of crop fields — including irrigated circles, rectangular plots, and irregular polygons — as they would appear from a light aircraft looking slightly downward, with correct perspective distortion throughout.

No libraries. No build step. One HTML file, open in any browser.

---

## Why this exists

When painting aerial landscapes from imagination rather than pure observation, you need a mental model of *how shapes distort* at different distances and angles from your eye. A circle directly below you looks round. The same circle near the edge of your view — further from the nadir point — flattens into a horizontal ellipse. Rows of fields compress together toward the horizon. The whole scene is governed by consistent geometry, and if you violate it, the viewer's eye notices even if they can't articulate why.

This tool lets you dial in a camera position, generate a random field composition, and use the output as a loose reference — not to copy exactly, but to internalize the spatial logic before working gesturally.

---

## The coordinate system

The ground plane is a flat 2D surface parameterised by `(gx, gy)` in **ground units**. The origin `(0, 0)` is the **nadir point** — the spot on the ground directly below the camera eye. Two ground units span the default canvas width, so one unit is roughly half the image width at normal settings.

- `gx` increases to the right
- `gy` increases downward (toward the near/bottom edge of the image)
- The camera is at `(0, 0, camH)` in 3D — directly above the nadir, at height `camH` ground units

This means the nadir point is the one location where the camera is looking *perfectly straight down*. It projects to a specific screen position that you can move with the Nadir X / Nadir Y sliders.

---

## The projection

### Straight-down (nadir) case

If the camera looks perfectly straight down with no tilt, a ground point `(gx, gy)` projects to screen coordinates via a simple perspective divide:

```
sx = nadirSx + (gx / camH) * F
sy = nadirSy + (gy / camH) * F
```

where `F = camH × scale` is the focal length in pixels (scaled so that nearby ground features map 1:1 with the canvas), and `(nadirSx, nadirSy)` is the nadir point in screen pixels.

In the pure nadir case, every circle on the ground projects as a **perfect circle** — there is no distortion, because the camera-to-ground angle is identical everywhere. This is the `tilt = 0` setting.

### Tilt

In practice, an aircraft camera is rarely perfectly nadir. It points slightly forward or backward, which introduces a preferred direction of foreshortening: fields toward the "far" direction flatten, while fields toward the camera compress.

We model this by making the effective **depth** of each ground point dependent on its `gy` position:

```
depth(gx, gy) = camH + gy × sinAlpha
```

where `sinAlpha` is a signed tilt factor. The full projection becomes:

```
sx = nadirSx + (gx / depth) × F
sy = nadirSy + (gy / depth) × F
```

**Positive tilt (▼ fwd):** The camera tilts slightly forward, so ground points with larger `gy` (farther from camera, toward the top of image) have larger depth and thus project *closer together* on screen. Fields at the top of the image compress and flatten. Fields at the bottom (near, small `gy`) are barely distorted and circles remain round. This matches the reference photo.

**Negative tilt (▲ back):** The opposite — compression happens at the bottom of the image.

**Why the depth formula works:** Think of it as the camera not pointing straight down, but angled slightly. A ground point directly ahead of the camera is further from the lens than one directly below it. The extra distance is proportional to how far "forward" the point is (`gy`) and the sine of the tilt angle. We approximate `sinAlpha ≈ tilt / (camH + 0.5)` so that higher altitude naturally reduces the tilt effect — consistent with reality.

### How circles become ellipses

A ground circle of radius `rg` centered at `(cx, cy)` is sampled at 64 evenly-spaced angles:

```
point_i = (cx + cos(a_i) × rg,  cy + sin(a_i) × rg)
```

Each point is then projected through `makeProj`. Because the depth formula adds `gy × sinAlpha`, points on the *far side* of the circle (larger `gy`) have more depth and project *closer to the center* in screen space. Points on the *near side* project further out. The result is an ellipse compressed along the vertical (depth) axis — wider than it is tall — which is exactly what you see in an aerial photograph.

The further a circle is from the nadir point, the greater the depth variation across its diameter, and the flatter it becomes.

---

## The inverse projection

To generate a field grid that always fills the canvas — even at extreme tilt where the ground "stretches" toward a distant horizon — the code needs to know what ground area is visible. We compute this by **inverting the projection**: given a screen point `(sx, sy)`, find `(gx, gy)`.

Starting from the forward projection:

```
sy = nadirSy + (gy / depth) × F       where depth = camH + gy × sinAlpha
```

Let `dy = (sy − nadirSy) / F`. Then:

```
dy = gy / (camH + gy × sinAlpha)
dy × camH + dy × gy × sinAlpha = gy
dy × camH = gy × (1 − dy × sinAlpha)
gy = (dy × camH) / (1 − dy × sinAlpha)
```

And for `gx`, once `gy` is known:

```
depth = camH + gy × sinAlpha
gx = ((sx − nadirSx) / F) × depth
```

The denominator `(1 − dy × sinAlpha)` approaches zero near the horizon — this is the mathematical equivalent of the vanishing point, where parallel ground lines converge. The code guards against this with a clamp.

The code probes 12 screen points (corners, edge midpoints, quarter-points along top and bottom edges), inverts each one to ground coordinates, takes the bounding box of all results, pads it by 0.5 ground units, and uses that as the grid extent. Row and column counts scale with the ground area so field density stays visually consistent.

---

## The terrain warp

Real agricultural land is not perfectly flat. Gentle undulation — a ridge, a drainage depression — subtly bends the grid lines and distorts circle shapes in ways that feel organic rather than mechanical. The warp is applied in **ground space** before projection, displacing each point by a sum of low-frequency sine waves:

```
gx_warped = gx + sin(gy × 1.3π + gx × 0.8 + phase) × warpAmt × 0.055
gy_warped = gy + cos(gx × 1.1π + gy × 0.6 + phase) × warpAmt × 0.038
```

The phase constants are arbitrary but fixed, so the warp pattern is spatially coherent — nearby points displace in similar directions, producing smooth-looking terrain rather than noise. Because the warp happens before the perspective divide, it interacts naturally with the projection: a warped circle still foreshortens correctly, it just follows the terrain contour rather than sitting on a perfect plane.

---

## The random number generator

All randomness uses a **seeded linear congruential generator (LCG)**:

```
s_next = (s × 16807) % 2147483647
value  = (s_next − 1) / 2147483646
```

This is the classic Park-Miller generator (also called Lehmer RNG). It produces a deterministic sequence of values in `[0, 1)` from any integer seed, with a period of `2^31 − 2`. Because it's seeded, the same number always produces the same composition — this is what makes the seed input useful for saving and recalling layouts.

The `Rand` class wraps it with `.range(a, b)`, `.int(a, b)`, and `.pick(array)` helpers.

---

## Field generation

Fields are generated entirely in ground space, then projected. The process:

1. **Row edges** are placed from `gBottom` upward, each row height randomly drawn from `[0.18, 0.42]` ground units (scaled proportionally if the visible ground area is large).

2. **Columns** within each row are given random relative widths, then normalized to span `gLeft → gRight`.

3. **Field type** is randomly assigned:
   - Plain rectangle (most common)
   - Harvest strips (alternating bands of two colors)
   - L-shaped sub-region
   - Trapezoid (slightly skewed quad)
   - Single pivot circle
   - Multi-circle block (2, 3, or 4 sub-circles sharing a block)

4. **Multi-circle layout** mirrors real quarter-section agriculture: a 2×2 grid of four equal pivots, two side-by-side, two stacked, or a 2+1 triangular arrangement. Sub-circle radii are calculated to fit within their sub-block with a small margin.

5. **Nadir circle**: whichever circle field is closest to the ground origin `(0, 0)` is marked as the nadir circle and drawn with a slightly different stroke — it's the one that should appear most round in the composition.

6. **Ponds**: a small number of irregular dark blobs are scattered across the ground using cubic Bézier curves, representing the water bodies visible in most aerial agricultural photographs.

---

## The controls

| Control | What it does |
|---|---|
| **Altitude** | Camera height in ground units. Higher = less distortion overall, fields more uniform. Lower = more dramatic foreshortening at the edges. |
| **Tilt** | Signed tilt of the camera off nadir. Positive = forward tilt, compression toward top of image (matches most aerial photos). Zero = pure nadir. Negative = rearward tilt, compression toward bottom. |
| **Nadir X / Y** | Screen position of the nadir point — the one spot where the camera looks perfectly straight down and circles appear round. Moving it off-center creates asymmetric compositions. |
| **Terrain warp** | Amplitude of the ground-plane displacement. Zero = perfectly flat. Higher values introduce organic bending of field edges and circle shapes. |
| **Circle density** | Probability that any given single-field cell contains a pivot circle rather than a rectangular crop pattern. |
| **Multi-circle freq** | Probability that a large enough field cell gets subdivided into 2–4 smaller pivot circles instead of being treated as a single field. |
| **Pivot arms** | Toggles the dashed line from each circle center, representing the physical pivot arm of the irrigation system. Also useful as a construction cue: the arm points along the ellipse's major axis. |
| **Circle fill / rings** | Toggles wedge shadows (partially harvested sectors) and concentric rings (topographic-style tonal variation within a circle). |
| **Terrain warp** (toggle) | Enables/disables the sine-wave ground displacement. |
| **Show nadir** | Draws a small crosshair at the nadir screen position. Useful for understanding where the "round circle" anchor sits in the composition. |

---

## Using it as a painting reference

The intended workflow is loose, not mechanical:

1. Hit **New composition** a few times until a layout feels interesting.
2. Note the seed number — type it back in later to recall the exact composition.
3. Dial in altitude, tilt, and nadir position to match the spatial feeling you want (close-up and dramatic vs. high and flat).
4. Observe which circles are roundest (near the nadir crosshair) and which are flattest (far from it, especially toward the "tilt" direction).
5. Use that as a mental framework, then work freely — you don't need to copy every edge exactly. The value is internalizing the logic so your gestural marks stay spatially coherent.
6. Hit **Export PNG** to save a reference image at full canvas resolution.

The terrain warp slider is particularly useful at the painting stage: a small amount of warp (0.15–0.30) breaks the mechanical regularity in a way that looks like actual land, and gives you permission to be equally loose in the painting without it reading as a mistake.

---

## File structure

Everything lives in a single `aerial_crop_generator.html` file:

- **CSS** (~120 lines): layout, slider styling, light/dark mode via `prefers-color-scheme`
- **HTML** (~60 lines): canvas element, sliders, buttons, seed input
- **JavaScript** (~320 lines):
  - `Rand` — seeded LCG random number generator
  - `makeProj` — forward projection function (ground → screen)
  - `unproj` — inverse projection (screen → ground), used for grid bounds
  - `drawCircle` — renders a single pivot circle with optional wedge and rings
  - `subCircleLayout` — computes sub-circle positions for multi-pivot blocks
  - `buildAndDraw` — main render function, called on every slider change
  - Event bindings and export

No external dependencies. Runs entirely in the browser.

---

## Further reading

If you want to go deeper on the math:

- **Perspective projection**: any computer graphics textbook covers the pinhole camera model. The key insight is that the perspective divide `x_screen = f × x_world / z_depth` is what makes parallel lines converge and distant objects appear smaller.
- **Ellipses from projected circles**: a circle on a plane, viewed obliquely, always projects as a conic section — specifically an ellipse (or circle when viewed face-on). The minor/major axis ratio equals `cos(θ)` where `θ` is the angle between the viewing ray and the circle's normal.
- **LCG random number generators**: the Park-Miller generator used here is described in *"Random Number Generators: Good Ones Are Hard to Find"* by Park & Miller (1988). It's not cryptographically secure, but it's perfectly fine for generative art — fast, seedable, and no external state.

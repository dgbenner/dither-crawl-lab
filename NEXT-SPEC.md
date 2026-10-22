# Dither Crawl Lab: next build

Spec for Claude Code. Three additions to the existing lab page, in this order: dot size, split dither, superformula shape scene.

## Ground rules

- Read the current lab file in full before changing anything. It has additions beyond the original build. Keep every existing control, preset, palette and behavior working.
- Add new controls to the existing control panel, in the matching section. Don't restructure the panel.
- Saved settings: add the new keys with defaults, and keep the existing localStorage key. Old saved settings must still load; any missing key falls back to its default.
- Stay in the current architecture: real HTML underneath at `opacity: 0`, and a low-resolution canvas on top that decides per cell whether a dot is on. No WebGL, no libraries.
- Update `LAB-NOTES.md` with the new controls (range, default, what it does).

---

## 1. Dot size (gaps between dots)

**Goal:** each grid cell draws a smaller square with a visible gap, like a dot-matrix screen. In the reference, even solid fills (the button) show a fine grid of gaps.

**Control:** "Dot size" in the Screen section. Range 50–100%, step 5, default 100% (the current look). Around 70% matches the reference.

**Implementation:**
- Keep rendering the low-resolution buffer exactly as now.
- Draw it to a display canvas sized to the stage in device pixels (CSS size × `devicePixelRatio`), scaled up with `imageSmoothingEnabled = false`.
- Then fill the whole display canvas once with a repeating pattern made from one cell-sized tile: transparent square in the middle (dot size × cell), screen color around it. The gaps take the screen color, so empty cells look unchanged and on-cells show as separated dots.
- Rebuild the tile when pixel size, dot size, palette or DPR changes, not every frame.
- Cells must line up with the pattern exactly. Size the display canvas to a whole number of cells.
- At 100%, skip the pattern pass.
- The existing "LCD pixel grid" overlay stays as a separate control.

**Check:** at 70% on Sage, the button's solid fill shows a light grid between dark dots, and the background looks the same as at 100%.

---

## 2. Split dither: surfaces vs halos

**Goal:** match the reference, which uses two dither styles at once:

- **Surfaces** (shaded shapes, ground shadows, any filled tone) use an ordered Bayer pattern that does not crawl. It reads as solid and stable, with checkerboards and plus shapes in the mid-tones.
- **Halos** (the glow around UI controls) keep the current crawling stipple.

**Controls:**

- "Halo pattern" in the Crawl section is the existing noise pattern control, renamed. It keeps Random / Ordered 8×8 jittered / Gradient, with Random as the default.
- "Surface pattern" goes in the Shadow section, with these options:
  - Ordered 4×4 (default)
  - Ordered 8×8
  - Random, static
- Surfaces never crawl, whatever the crawl speed.

**Implementation:**

- Write each tone into the low-res buffer tagged with its layer (`halo` or `surface`), and threshold each layer against its own pattern.
- Surface thresholds use a fixed screen-space Bayer matrix with no offset changes.
- Crawl speed, Churn and the crawl-activity readout apply only to halos.

---

## 3. Shape scene: superformula morph

**Goal:** a second scene on the same page.

- A rotating, lit 3D shape is shaded with the surface dither and sits over a dithered ground shadow.
- Below it are a name label and a "Morph" button with the crawling halo.
- Pressing Morph blends into the next shape, and the label scrambles from the old name to the new one.

The author of the reference described it this way: each shape comes from the superformula, JS turns it into thousands of triangles, rotates and lights them every frame, and each dot turns on or off by brightness. Morphing blends the numbers from one shape to the next.

### Scene switch

Add "Scene" at the top of the panel, with two options: "UI controls" (the existing scene, default) and "Shape morph". The current scene is saved with the other settings. Both scenes share the same renderer, palette, pixel size, dot size and dither settings.

### Superformula

The 2D superformula, with radius as a function of angle:

```
r(θ) = ( |cos(m·θ/4) / a|^n2 + |sin(m·θ/4) / b|^n3 )^(−1/n1)
```

For the 3D surface, combine two of them (A around, B top to bottom) over longitude θ ∈ [−π, π] and latitude φ ∈ [−π/2, π/2]:

```
x = rA(θ)·cos θ · rB(φ)·cos φ
y = rA(θ)·sin θ · rB(φ)·cos φ
z = rB(φ)·sin φ · zScale
```

- Each shape is 13 numbers: A {m, a, b, n1, n2, n3}, B {m, a, b, n1, n2, n3}, and zScale.
- Guard against division by zero and against NaN when n1 is near 0.
- Normalize every frame: scale the mesh so its largest extent fits a fixed target radius. Superformula radii vary widely between shapes.

### Shapes

Five shapes, in a fixed cycle: Shell → Gem → Seed pod → Starfish → Alien urchin → Shell. The values below are starting points. Tune them by eye until each one reads as its name. Matching the reference exactly isn't required.

| Name | A (m, a, b, n1, n2, n3) | B (m, a, b, n1, n2, n3) | zScale |
|---|---|---|---|
| Shell | 3, 1, 1, 5, 18, 18 | 0, 1, 1, 1, 1, 1 | 0.8 |
| Gem | 4, 1, 1, 1, 1, 1 | 4, 1, 1, 1, 1, 1 | 1.0 |
| Seed pod | 4, 1, 1, 12, 15, 15 | 3, 1, 1, 3, 4, 4 | 1.2 |
| Starfish | 5, 1, 1, 2, 7, 7 | 2, 1, 1, 2, 2, 2 | 0.3 |
| Alien urchin | 8, 1, 1, 0.6, 1, 1 | 8, 1, 1, 0.6, 1, 1 | 1.0 |

### Mesh, rotation and lighting

- Mesh resolution is 72 longitude × 36 latitude segments, two triangles per quad, rebuilt every frame from the current (possibly blended) values.
- Rotation: a slow yaw of about 20°/s, plus a fixed tilt of about 20° toward the viewer so the top is visible.
- Projection: orthographic, centered in the upper part of the stage.
- Lighting: Lambert with a light from the upper left front, `brightness = ambient 0.15 + 0.85 · max(0, N·L)`. Compute the normal per triangle, flat shading. Flip normals that face away from the camera.
- Rasterize into the low-res buffer with a per-cell z-buffer. Write darkness = 1 − brightness as a surface tone.
- Silhouette: cells on the outer edge of the shape render in ink, for a crisp outline.
- Reduced motion: no rotation.

### Ground shadow

- A flat ellipse below the shape.
- Its width follows the shape's current projected width (0.9×), and its height is 0.16 × its width.
- It's a surface tone, so it uses the static surface dither: darkness about 0.55 at the center, falling to 0 at the rim.
- It updates live, so a starfish gets a long, thin shadow.

### Morph

- Clicking Morph (a real hidden `<button>`) blends all 13 numbers from the current shape to the next over 900 ms with ease-in-out.
- A click during a blend starts from the current blended values toward the following shape.
- The button is drawn filled in ink with screen-colored text, and keeps the existing crawling halo (lift on hover/focus, press-down on click).

### Label scramble

- The name sits between the ground shadow and the button, centered, in the existing pixel font.
- On morph, over 600 ms, each character position switches from the old name to the new one at a staggered time, left to right with a little randomness. Before it switches, a position shows a random character from `A–Z a–z` or a blank. The label is padded to the longer of the two names.
- The real text is a hidden element with `aria-live="polite"`. It changes once, to the new name, when the morph starts. The scramble is canvas-only.
- Reduced motion: no scramble, just an instant swap.

### Performance

- Target 60 fps on a laptop at 3× pixel size.
- Only the active scene's DOM receives input. Hide the inactive scene's elements with `hidden`.

---

## Acceptance checks

1. Dot size at 70% shows gaps in the solid fills, and the background looks unchanged.
2. Halos still crawl. Surfaces and ground shadows don't change from frame to frame unless the shape moves.
3. Switching Surface pattern visibly changes the shape shading. Switching Halo pattern changes only the halo.
4. Shape morph scene:
   - All five shapes cycle with the Morph button and are recognizably different.
   - Morphs are smooth, with no popping, NaN holes or blown-out scale.
   - The label scrambles, then settles on the right name.
5. Keyboard: Tab reaches the Morph button and Enter or Space triggers it. A screen reader announces the new shape name once per morph.
6. Reduced motion: no rotation, crawl or scramble, and everything still works.
7. Saved settings from the previous version load without errors.
8. `LAB-NOTES.md` documents every new control.

## Out of scope

Per-element shadow settings, LCD ghosting, directional crawl, cursor light, font picker. These are listed in LAB-NOTES.md for later.

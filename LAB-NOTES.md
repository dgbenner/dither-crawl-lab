# Dither Crawl Lab: notes

Halftone shadows on live UI in a handheld-LCD style. The shadows crawl on purpose: the dense core holds still and the edge dots keep shifting.

Inspired by a post from Nacho Gavilanes on X, whose own description of the build was "custom renderer on top, real html hidden underneath."

---

## How it works

1. **Real HTML underneath.** The text field, Table/Chart tabs, Sound switch and Delete button are ordinary form elements set to `opacity: 0`. They still receive clicks, typing, focus and screen-reader access, so the browser does all the behavior.
2. **Custom renderer on top.** A full-size `<canvas>` sits over them with `pointer-events: none`, so clicks pass through. It renders at low resolution (one logical pixel = 2–4 screen pixels) and is scaled up with `image-rendering: pixelated`.
3. **Every frame the renderer reads the DOM.** For each element: position and size, plus state (input value, caret and selection, checked, selected tab, hover, focus, pressed). It then redraws outlines, pixel-font text, caret, switch knob and shadow itself.
4. **Shadows are a threshold.** Each shadow is one soft shape, offset from its element. At every pixel the shadow strength is compared to a noise value. If it's stronger, a dot is drawn.
5. **The crawl.** The noise is partly re-rolled several times a second. Pixels deep in the shadow (strength near 100%) always come out dark, and pixels far away (near 0%) always come out empty. Only the in-between pixels at the edge change. That is why the core stays still and the fringe moves.
6. **Two dither layers.** Halos (the glow around UI controls) use the crawling noise above. Surfaces (shaded shapes and ground shadows) use a fixed screen-space pattern that never crawls, so they read as solid and stable.
7. **Dot size.** The low-res buffer is scaled up to device pixels, then a repeating one-cell tile paints screen color around a smaller square in each cell, which leaves a gap between the dots.
8. **No transparency.** Every pixel is exactly one of three colors (screen, dots, ink). Overlapping shadows don't darken each other, which is why it doesn't behave like a normal CSS shadow.

The text field's real font size matches the canvas font (8 logical pixels per character), so clicking inside the text places the caret where you'd expect.

---

## Controls

On-screen wording was rewritten in plain language on 2026-10-24 (source: `_source/copy-edits-2026-10-24.txt`).

The panel is a stack of accordion sections, in this order: Scene (always shown), Color, the crawl readout and presets, Crawl, Shadow, Screen, Video, Inspect, Settings. Every section starts collapsed when the page loads. Video opens itself when you switch to the Video scene or load a clip.

### Scene

| Control | Options | Default | What it does |
|---|---|---|---|
| Scene | UI controls / Shape morph / Video | UI controls | Which scene is on screen. Both share the renderer, palette, pixel size, dot size and dither settings. The inactive scene's HTML is hidden, so it takes no input. |

### Crawl

| Control | Range | Default | What it does |
|---|---|---|---|
| Crawl speed | 0–60 updates/s | 2 | How often the noise changes. 0 stops the dots completely (shown as "Stopped"). |
| Churn | 2–100% | 35% | Share of the noise re-rolled on each update. Low = slow simmer, 100% = full reshuffle every update. Applies to Random noise only. |
| Halo pattern | Random / Ordered grid, shifting / Fine even grain | Random | The texture of the halo dots (see below). Formerly "Noise pattern". |

Crawl speed, Churn and the crawl-activity readout apply only to halos.

**Halo patterns**

- **Random:** scattered dots with no grid. Closest to the reference video.
- **Ordered grid, shifting:** classic Bayer cross-hatch, the textbook Game Boy dither look. Each update shifts the whole pattern to a new offset, so the crawl reads as a shimmer instead of a boil.
- **Fine even grain:** interleaved gradient noise, an even, fine-grained pattern between the other two. Each update jumps to a new offset.

### Shadow

| Control | Range | Default | What it does |
|---|---|---|---|
| Offset down | −10 to 16 px | 1 | Moves the shadow below (positive) or above (negative) the element. 0 gives an even halo. |
| Offset right | −10 to 10 px | −1 | Moves the shadow sideways. |
| Spread | 2–32 px | 4 | How far the dots reach past the shape. |
| Falloff | 0.5–3.0 | 1.7 | How fast the dots thin out. Higher pulls them in tight to the edge; lower gives a long, soft fade. |
| Density | 20–120% | 75% | Overall darkness of the shadow. Above 100% makes a solid core. |
| Shadow grows on hover and focus | on/off | on | Hover or keyboard focus raises the element: the offset grows 80% and the spread 45%. Pressing the button pushes it down (offset shrinks 75%). |
| Surface pattern | Ordered grid, small / Ordered grid, large / Random, still | Ordered grid, small | The dither on shaded shapes and ground shadows. A fixed pattern with no offset changes: surfaces never crawl, whatever the crawl speed. |

Pixel values are logical pixels, so they scale with Pixel size.

### Color

| Control | Range | Default | What it does |
|---|---|---|---|
| Palette | 7 presets + Custom | Light grey, dark grey | Three colors: screen, dots, ink. |
| Color pickers | any color | from palette | Screen, Dots and Ink. Changing any of them switches to Custom. |

### Screen

| Control | Range | Default | What it does |
|---|---|---|---|
| Pixel size | 2×, 3×, 4× | 3× (2× on narrow screens) | Screen pixels per logical pixel. Bigger = chunkier. The whole layout scales with it. |
| Dot size | 50–100%, step 5 | 100% | How much of each cell the dot fills. Below 100% the gaps show in screen color, like a dot-matrix screen: empty areas look the same, solid fills show a fine grid. About 70% matches the reference. Gap widths round to whole device pixels. |
| LCD pixel grid | on/off | on | Faint grid lines between pixels, like a real LCD. |

### Video

| Control | Range | Default | What it does |
|---|---|---|---|
| Choose video… | any video file | demo clip | Loads a clip from your computer and switches to the Video scene. Dropping a file on the screen does the same. Nothing is uploaded; the file stays in the browser. |
| Play / Pause | | plays | Starts and stops the clip. It doesn't autoplay with reduced motion on, and it pauses when you leave the Video scene. |
| Black point | 0–90% | 8% | Source brightness at or below this becomes screen color. |
| White point | 10–100% | 85% | Source brightness at or above this becomes solid ink. |
| Gamma | 0.4–2.5 | 1.00 | Above 1 darkens the midtones, below 1 lifts them. |
| Video dither | Still / Crawling | Still | Still uses the Surface pattern, so dots only change when the picture does. Crawling uses the Halo pattern, Crawl speed and Churn. |
| Invert | on/off | off | Flips brightness, for light palettes where ink is the dark color. |

The clip is drawn into the low-res buffer each frame, letterboxed to fit, and each cell's brightness is mapped in two steps: the lower half dithers between screen and dots, the upper half between dots and ink. The stage takes the clip's aspect ratio, up to 80% of the window height. High-contrast footage works best: a bright subject on a dark background. The video settings are saved; a chosen file isn't, so after a reload the scene opens on the demo again.

**Demo clip.** `media/demo-fish.mp4` (a goldfish on black, 92 s, 640×348, about 650 KB, no audio) loads the first time the Video scene opens. It's re-encoded from the original in `_source/` (kept local) with the 6 px white strip on the right edge cropped off. While the demo plays, a "Choose a video / or drop one here" plate sits over it. The plate is drawn over a real hidden button that opens the file picker; it inverts on hover and keyboard focus, and goes away once you load your own clip. With reduced motion it opens paused at 6 s, past the empty water at the start.

### Inspect

| Control | What it does |
|---|---|
| Highlight crawling pixels | Pink = a dot that just appeared, light pink = a dot that just disappeared. Shows that the crawl lives at the edges. |
| Show the hidden HTML | Brings the real form elements to the front with dashed outlines. |

### Readout

**Crawl activity** is the share of shadow pixels that changed on the last noise update. "Paused" when Crawl speed is 0.

### Settings

- Changes are saved in your browser automatically, including the current scene. Settings saved by an older version still load; new keys take their defaults.
- **Copy settings** copies the current values and colors as JSON, ready to paste into a chat.
- **Reset** returns to the Default preset, the Light grey, dark grey palette, 100% dot size and Ordered 4×4 surfaces, keeping your pixel size and scene.

---

## Shape morph scene

A rotating, lit 3D shape shaded with the surface dither, over a dithered ground shadow, with a name label and a Morph button below.

- **Superformula.** Each shape is two 2D superformulas, A around and B top to bottom, `r(θ) = (|cos(mθ/4)/a|^n2 + |sin(mθ/4)/b|^n3)^(−1/n1)`, combined into `x = rA(θ)cosθ·rB(φ)cosφ`, `y = rA(θ)sinθ·rB(φ)cosφ`, `z = rB(φ)sinφ·zScale`. That's 13 numbers per shape. Near-zero n1, a and b are clamped, and any non-finite radius becomes 0.
- **Mesh.** 72 × 36 segments, two triangles per quad, rebuilt every frame from the current (possibly blended) numbers, then scaled so its largest extent fits a fixed radius.
- **View.** Yaw at 20°/s, tilted 20° toward the viewer, orthographic. Flat Lambert shading per triangle from the upper left front: `brightness = 0.15 + 0.85·max(0, N·L)`; darkness = 1 − brightness goes in as a surface tone, rasterized with a per-cell z-buffer. Shape dots draw in ink, and cells on the outer edge draw in ink as a silhouette.
- **Ground shadow.** A flat ellipse, 0.9× the shape's projected width and 0.16× as tall, darkness 0.55 at the center falling to 0 at the rim, dithered like a surface (dot color). The ground line is fixed; the shape eases up or down so its lowest point rests just above it.
- **Morph.** The real hidden button blends all 13 numbers to the next shape over 900 ms, ease-in-out. A click mid-blend starts from the current blended values toward the following shape. The button is drawn in ink with screen-colored text, keeps the crawling halo, and shows an outside ring on keyboard focus.
- **Label.** Over 600 ms each character switches from the old name to the new one at a staggered time (left to right, with some randomness), showing random letters or blanks until it switches. A hidden `aria-live="polite"` element changes once to the new name when the morph starts.
- **Reduced motion.** No rotation and no scramble. The crawl starts paused as before.

| Shape | A (m, a, b, n1, n2, n3) | B (m, a, b, n1, n2, n3) | zScale |
|---|---|---|---|
| Shell | 12, 1, 1, 4, 8, 8 | 2, 1, 1, 2, 4, 4 | 0.5 |
| Gem | 4, 1, 1, 1, 1, 1 | 4, 1, 1, 1, 1, 1 | 1.0 |
| Seed pod | 4, 1, 1, 12, 15, 15 | 3, 1, 1, 3, 4, 4 | 1.2 |
| Starfish | 5, 1, 1, 2, 7, 7 | 2, 1, 1, 2, 2, 2 | 0.3 |
| Alien urchin | 8, 1, 1, 0.6, 1, 1 | 8, 1, 1, 0.6, 1, 1 | 1.0 |

The cycle is Shell → Gem → Seed pod → Starfish → Alien urchin → Shell. Shell was changed from the spec's starting values (a three-lobed blob) to a ribbed clam.

---

## Presets

| Preset | Speed | Churn | Noise | Spread | Falloff | Density | Offset down |
|---|---|---|---|---|---|---|---|
| Default | 2 | 35% | Random | 4 | 1.7 | 75% | 1 |
| Calm | 5 | 12% | Random | 12 | 1.7 | 75% | 4 |
| Boiling | 30 | 100% | Random | 16 | 1.1 | 100% | 3 |
| Ordered | 8 | n/a | Ordered 8×8 | 12 | 1.7 | 75% | 4 |
| Amber | 3 | n/a | Gradient | 3 | 1.7 | 75% | 0 |

Amber also switches the palette to Amber on black and sets Offset right to 0. The other presets leave the palette alone.

## Palettes

| Palette | Screen | Dots | Ink |
|---|---|---|---|
| Sage (like the reference) | `#a4ad8a` | `#4f5c3e` | `#36412b` |
| Original green | `#9bbc0f` | `#306230` | `#0f380f` |
| Pocket grey | `#c4c4b8` | `#6d6d63` | `#2b2b27` |
| Light grey, dark grey | `#d6d6d3` | `#5e5e5b` | `#2c2c2a` |
| Graphite, light grey | `#2e2f31` | `#8a8c8f` | `#e2e3e4` |
| C64 blue | `#352879` | `#6c5eb5` | `#b3abf2` |
| Amber on black | `#17140c` | `#a6761d` | `#f2c14e` |

## Font

Press Start 2P (Google Fonts), an 8-bit console-style pixel font, used at exactly 8 logical pixels. It can be swapped for another fixed-width pixel font; the text field's caret math assumes every character is the same width.

---

## Effects not built yet

These would fit the same renderer. None of them is in the lab yet.

**Pixel and screen**
- Pixel size beyond 2–4× (1× for fine grain, 5–8× for very chunky). The renderer supports any whole number; the layout would need to adapt at the extremes.
- LCD ghosting: dots fade out over a few frames instead of switching off instantly, like the smear on an original handheld screen.
- A fourth color, so the shadow core and fringe use two different dot tones.
- Scanlines or a slight screen curve for a CRT look instead of an LCD.

**Shadow**
- Settings per element (for example, a bigger, slower shadow on the button and a tight one on the switch).
- An inner shadow: dots inside the field along its top edge, so it looks recessed.
- Directional crawl: the noise drifts one way (down, or away from the cursor) instead of re-rolling in place.
- Cursor light: the shadow is cast away from the pointer.
- Blue-noise texture: the most even random-looking dots, from a precomputed tile.

**Interaction**
- A dither wipe on press or on tab change: the selected state dissolves in through the dither instead of sliding.
- Marching-dots focus ring instead of a thicker outline.
- Dithered fills: buttons and the switch track filled with a dot pattern instead of solid ink.
- A click ripple that briefly thickens the dots around where you clicked.

**Type**
- A font choice in the panel.
- A real C64-style font (a free fan-made remake would need its license checked).

---

## Publishing on GitHub Pages

The site is one file, `index.html`, with no build step.

1. Create an empty public repo `dgbenner/dither-crawl-lab` (no README).
2. Push this folder to `main`.
3. In the repo go to **Settings → Pages**, set Source to **Deploy from a branch**, pick **main** and **/ (root)**, then save.
4. The site goes live at `https://dgbenner.github.io/dither-crawl-lab/` within a couple of minutes.

## References

- [A forum post about the dithering in Return of the Obra Dinn](https://setsideb.com/a-forum-post-about-the-dithering-in-return-of-the-obra-dinn/): Lucas Pope on keeping dither from crawling. This lab does the opposite on purpose.
- [Game Boy-style dithering (UX Pilot)](https://uxpilot.ai/labs/dither-generator/game-boy): four greens, Bayer 4×4, chunky pixels.
- [The Art of Dithering and Retro Shading for the Web (Maxime Heckel)](https://blog.maximeheckel.com/posts/the-art-of-dithering-and-retro-shading-web): shader math for ordered and blue-noise dithering.
- [Surface-Stable Fractal Dithering (runevision)](https://runevision.com/tech/dither3d): pinning dither to surfaces.
- [Exploring the HTML-in-Canvas proposal (Codrops)](https://tympanus.net/codrops/?p=114302): the browser API this lab does not use.

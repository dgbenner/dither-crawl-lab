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
6. **No transparency.** Every pixel is exactly one of three colors (screen, dots, ink). Overlapping shadows don't darken each other, which is why it doesn't behave like a normal CSS shadow.

The text field's real font size matches the canvas font (8 logical pixels per character), so clicking inside the text places the caret where you'd expect.

---

## Controls

### Crawl

| Control | Range | Default | What it does |
|---|---|---|---|
| Crawl speed | 0–60 updates/s | 2 | How often the noise changes. 0 freezes the dots completely. |
| Churn | 2–100% | 35% | Share of the noise re-rolled on each update. Low = slow simmer, 100% = full reshuffle every update. Applies to Random noise only. |
| Noise pattern | Random / Ordered 8×8 / Gradient | Random | The texture of the dots (see below). |

**Noise patterns**

- **Random (stipple):** scattered dots with no grid. Closest to the reference video.
- **Ordered 8×8, jittered:** classic Bayer cross-hatch, the textbook Game Boy dither look. Each update shifts the whole pattern to a new offset, so the crawl reads as a shimmer instead of a boil.
- **Gradient noise:** interleaved gradient noise, an even, fine-grained pattern between the other two. Each update jumps to a new offset.

### Shadow

| Control | Range | Default | What it does |
|---|---|---|---|
| Offset down | −10 to 16 px | 1 | Moves the shadow below (positive) or above (negative) the element. 0 gives an even halo. |
| Offset right | −10 to 10 px | −1 | Moves the shadow sideways. |
| Spread | 2–32 px | 4 | How far the dots reach past the shape. |
| Falloff | 0.5–3.0 | 1.7 | How fast the dots thin out. Higher pulls them in tight to the edge; lower gives a long, soft fade. |
| Density | 20–120% | 75% | Overall darkness of the shadow. Above 100% makes a solid core. |
| Shadow grows on hover and focus | on/off | on | Hover or keyboard focus raises the element: the offset grows 80% and the spread 45%. Pressing the button pushes it down (offset shrinks 75%). |

Pixel values are logical pixels, so they scale with Pixel size.

### Screen

| Control | Range | Default | What it does |
|---|---|---|---|
| Pixel size | 2×, 3×, 4× | 3× (2× on narrow screens) | Screen pixels per logical pixel. Bigger = chunkier. The whole layout scales with it. |
| Palette | 7 presets + Custom | Light grey, dark grey | Three colors: screen, dots, ink. |
| Color pickers | any color | from palette | Screen, Dots and Ink. Changing any of them switches to Custom. |
| LCD pixel grid | on/off | on | Faint grid lines between pixels, like a real LCD. |

### Inspect

| Control | What it does |
|---|---|
| Highlight crawling pixels | Pink = a dot that just appeared, light pink = a dot that just disappeared. Shows that the crawl lives at the edges. |
| Show the hidden HTML | Brings the real form elements to the front with dashed outlines. |

### Readout

**Crawl activity** is the share of shadow pixels that changed on the last noise update. "Paused" when Crawl speed is 0.

### Settings

- Changes are saved in your browser automatically.
- **Copy settings** copies the current values and colors as JSON, ready to paste into a chat.
- **Reset** returns to the Reference preset and the Light grey, dark grey palette, keeping your pixel size.

---

## Presets

| Preset | Speed | Churn | Noise | Spread | Falloff | Density | Offset down |
|---|---|---|---|---|---|---|---|
| Reference | 2 | 35% | Random | 4 | 1.7 | 75% | 1 |
| Calm | 5 | 12% | Random | 12 | 1.7 | 75% | 4 |
| Boiling | 30 | 100% | Random | 16 | 1.1 | 100% | 3 |
| Ordered | 8 | n/a | Ordered 8×8 | 12 | 1.7 | 75% | 4 |

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

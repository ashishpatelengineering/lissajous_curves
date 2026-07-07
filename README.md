# Lissajous Curves

(Lissajous Curves)[https://ashishpatelengineering.github.io/lissajous_curves/]
An interactive, single-file visualization of Lissajous curves. Drag the top edge to set the horizontal frequency and the left edge to set the vertical frequency, and watch the resulting curve draw itself in and settle to rest.

The whole thing is one self-contained `index.html` — no build step, no dependencies, no network. Open it in any modern browser (light and dark themes are both handled automatically).

---

## The Math

### What a Lissajous curve is

A **Lissajous curve** (also called a Lissajous figure or Bowditch curve) is the path traced by a point whose horizontal and vertical positions each oscillate independently. Picture two pendulums swinging at right angles to each other: one moves the pen left–right, the other moves it up–down. The shape the pen draws is a Lissajous curve.

Mathematically, the point's position over time is given by two parametric equations:

```
x(t) = cos(f₁ · t)
y(t) = sin(f₂ · t)
```

- `t` is time (or, equivalently, the angle that sweeps from 0 to 2π).
- `f₁` is the **horizontal frequency** — how many times the point completes a left–right cycle.
- `f₂` is the **vertical frequency** — how many times it completes an up–down cycle.

As `t` runs from 0 to 2π, the point moves and the trace fills in. This app maps `f₁` to the top edge and `f₂` to the left edge.

### Why the ratio is what matters

The *shape* of the curve is governed by the **ratio** of the two frequencies, `f₁ : f₂`, not their absolute values. A 4:3 curve and an 8:6 curve are the same figure. When the ratio is a simple whole-number fraction, the curve closes into a stable, repeating loop; the simpler the fraction, the simpler the figure.

A handy way to read a Lissajous figure: count how many times the curve touches the horizontal boundary versus the vertical boundary. Those two counts are the frequency ratio. For a 4:3 curve you'll find 4 "lobes" along one direction and 3 along the other — which is exactly why oscilloscopes have historically been used to compare two signal frequencies by feeding one into the X input and one into the Y input.

### The role of phase

The two oscillations can be shifted relative to each other. That shift is the **phase difference**, written `δ`:

```
x(t) = cos(f₁ · t + δ)
y(t) = sin(f₂ · t)
```

Phase changes how the same frequency ratio is drawn. The clearest example is the 1:1 ratio:

- With a 90° (π/2) phase difference, `x = cos(t)` and `y = sin(t)` trace a **circle**.
- With no phase difference, `x = sin(t)` and `y = sin(t)` move together and collapse the circle into a **diagonal line**.
- Anything in between produces an **ellipse**.

This app fixes the phase at **δ = π/2** — using cosine for `x` and sine for `y`. That is the choice that makes the curve pass cleanly through the center with crossing strands (rather than sitting tangent to the edges), and it's what lets a 1:1 setting draw a perfect circle.

### Things to try

- **1 and 1** → a circle (the π/2 phase at work).
- **2 and 1** → a figure-eight / parabola-like arc.
- **3 and 2** → a classic three-by-two weave.
- **4 and 3** → the default; a dense, lattice-like figure.
- Equal frequencies always give an ellipse or circle; the further apart and less "simple" the ratio, the busier the figure.

---

## The Code

Everything lives in `index.html`: markup, styles, and a single `<script>`. It renders onto one `<canvas>` with the 2D context and drives animation with `requestAnimationFrame`. There is no framework and no external library.

### The core mapping

The heart of the program is the parametric point function. Given an angle `theta` (the `t` from the math) it returns a pixel coordinate:

```js
function pt(theta, w, h) {
  const d = dims(w, h);
  return {
    x: d.cx + d.ax * Math.sin(fx * theta + delta),  // delta = π/2
    y: d.cy + d.ay * Math.sin(fy * theta)
  };
}
```

`fx` and `fy` are the two frequencies (integers 1–8). `d.cx`/`d.cy` are the canvas center; `d.ax`/`d.ay` are the horizontal and vertical amplitudes (half the drawable width/height after padding). Because `sin(θ + π/2) = cos(θ)`, the `x` equation is effectively a cosine, giving the phase relationship described in the math section. Sweeping `theta` from 0 to 2π and connecting the points produces the full curve.

### Layout: `dims()`

A single helper computes every position the renderer needs from the current canvas size: the center, the amplitudes, the padding, and where the two interactive rails sit (`railTop` near the top edge, `railLeft` near the left edge). Because everything is derived from the live width and height, the visualization stays correct when the window resizes or on high-DPI screens (the canvas backing store is scaled by `devicePixelRatio`).

### The animation lifecycle

The single `draw()` loop moves the piece through four phases, giving it a sense of "arrive, perform, rest":

1. **Hold** — for the first `holdDur` (700 ms) the faint full curve is shown but still. A beat of stillness before anything moves.
2. **Reveal** — over the next `revealDur` (2400 ms) the faint curve draws itself in, from 0 up to the full 2π, on an eased (cubic ease-out) progress curve.
3. **Run** — once revealed, a bright tracer with a fading tail travels along the curve. It runs `lapsBeforeRest` (2) full laps.
4. **Settle** — after those laps the tracer eases toward a stop over `restSpan` of angle, simultaneously slowing its speed and fading out the tail, head, glow, and projection lines. What remains is the calm, softly lit full curve. A `settling`/`atRest` flag pair tracks this state.

Any change to a frequency calls `replay()`, which resets the clock and the lap/settle state so the new curve runs through the whole lifecycle again.

### What gets drawn each frame

- **The faint full curve** — the complete figure at low opacity, so the shape is always legible.
- **The tail** — a short trailing segment behind the moving head, built from ~96 small line segments whose opacity, width, and color fade from the trace color toward the accent as they approach the head. This is what gives the motion direction and life.
- **The head** — a single accent-colored dot with a soft radial-gradient glow behind it. It's the one point of color; everything else is graphite/monochrome.
- **The projections** — two faint dashed lines dropping from the head to the top and left rails, plus small dots on those rails. These make the "two independent oscillations combine into one point" idea visible.
- **The rails** — the top and left edges, each with evenly spaced grey dots marking the integer frequencies 1–8 and a knob showing the current value. On hover or drag the rail brightens and its knob turns the accent color.

### Interaction

Pointer events (so mouse, trackpad, and touch all work through one code path) drive the rails:

- `railHit()` checks whether a pointer is near the top or left rail.
- On `pointerdown` over a rail, dragging begins and the pointer is captured.
- `posToFreq()` converts the pointer's position along a rail into a snapped integer frequency (1–8) via `Math.round`.
- When the snapped value changes, the corresponding frequency (`fx` or `fy`) updates and `replay()` restarts the lifecycle.

`touch-action: none` on the canvas wrapper prevents the page from scrolling while you drag a rail on a touch device.

### Theming and color discipline

All colors are CSS custom properties defined once for light mode and again inside a `prefers-color-scheme: dark` block, so the piece follows the system theme with no JavaScript. The renderer reads the relevant variables each frame, which is why switching themes is seamless.

The page uses a layered background: a slightly tinted base (`--page`) with a soft, neutral radial glow (`--glow`) centered behind the visualization, so the surrounding space reads as considered atmosphere rather than flat emptiness. The card surface (`--bg`) is kept lighter than the page and lifted with a soft layered drop shadow, so the visualization floats as a distinct object rather than dissolving into the background.

The visual rule throughout is deliberate restraint: everything on the canvas is monochrome graphite except a single accent-colored moving head (and the brief accent bloom on an active rail). The title uses a softened near-black to suit its thin weight, and the instruction line a mid-grey tuned to hold contrast against the tinted background.

### Small helpers

- `toRGB()`, `mix()`, `hexA()` — parse a CSS color to RGB and let the renderer interpolate between two colors or apply an alpha, used for the fading tail and the glow.
- `freqToPos()` / `posToFreq()` — the two-way conversion between a frequency value and a pixel position along a rail.

---

## Files

- `index.html` — the entire application (open this).
- `README.md` — this document.

## Running it

Locally, just open `index.html` in a browser — there's nothing to install or build.

### Hosting on GitHub Pages

Because it's a single static file named `index.html`, it works on GitHub Pages with no configuration:

1. Push these files to the repository root (or into a `/docs` folder).
2. On GitHub, go to **Settings → Pages**.
3. Under **Build and deployment**, set **Source** to *Deploy from a branch*.
4. Choose your branch (e.g. `main`) and the folder — `/ (root)` if the files are at the top level, or `/docs` if you put them there.
5. Save. After a minute or two your site is live at `https://<your-username>.github.io/<repo-name>/`.

Since the file is named `index.html`, that root URL loads the visualization directly — no filename needed in the path.

## License

Use it however you like.

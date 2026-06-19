Here's a complete handoff document for Claude Code.

---

# The Structure Beneath — Claude Code Handoff

## Project overview

A single-page browser experience for sixth-form Computer Science students. Six artworks, each concealing a different algorithm. Students observe, tune a live simulation, lock in their interpretation, and receive a reveal. Hosted on GitHub Pages, accessed externally on student devices.

## Repository structure expected

```
/
├── index.html                  (rename from The_Structure_Beneath_dc.html)
├── support.js                  (existing dc-runtime — do not modify)
├── vendor/
│   ├── react.production.min.js
│   └── react-dom.production.min.js
└── assets/
    └── images/
        ├── starry-night.jpg
        ├── the-scream.jpg
        ├── plum-park.jpg
        ├── cone-snail.jpg
        ├── harmonograph.jpg
        └── grande-jatte.jpg
```

---

## Task 1 — Vendor React locally

**Why:** `support.js` currently loads React 18 from `unpkg.com` at runtime. If unpkg has an outage the entire experience breaks. Remove the external dependency.

**What to do:**

Download these two files and place in `/vendor/`:
- `https://unpkg.com/react@18.3.1/umd/react.production.min.js`
- `https://unpkg.com/react-dom@18.3.1/umd/react-dom.production.min.js`

In `support.js`, find these constants near the bottom of the file:

```js
var REACT_URL = "https://unpkg.com/react@18.3.1/umd/react.production.min.js";
var REACT_DOM_URL = "https://unpkg.com/react-dom@18.3.1/umd/react-dom.production.min.js";
```

Change to:

```js
var REACT_URL = "./vendor/react.production.min.js";
var REACT_DOM_URL = "./vendor/react-dom.production.min.js";
```

Remove the SRI integrity attributes from the `loadScript` call — they will fail against local files. Find `loadScript` and remove the `s.integrity` and `s.crossOrigin` lines, or pass empty strings.

**Verify:** Open `index.html` from a local file with no internet connection. The gallery should render.

---

## Task 2 — Source and embed artwork images

**Why:** The six painting panels currently show hatched placeholders. Students need to see the actual artworks.

**What to source:** All six works are public domain and available on Wikimedia Commons. Download the highest resolution version practical for screen use (1200px wide is sufficient). Save to `/assets/images/` with the filenames listed above.

| Card ID | Artwork | Wikimedia search term |
|---|---|---|
| `starry` | The Starry Night — Van Gogh, 1889 | `Van Gogh Starry Night` |
| `scream` | The Scream — Munch, 1893 | `Munch The Scream 1893` |
| `hiroshige` | Plum Park in Kameido — Hiroshige, 1857 | `Hiroshige Plum Park Kameido` |
| `conesnail` | Cone snail shell — *Conus textile* | `Conus textile shell` |
| `harmonograph` | Harmonograph figure — c.1900 | `Harmonograph figure` |
| `seurat` | A Sunday on La Grande Jatte — Seurat, 1886 | `Seurat Grande Jatte` |

**In `index.html`:** In the gallery card template, replace the hatched placeholder div with an `<img>` tag:

```html
<img src="./assets/images/{{ item.imgFile }}" 
     alt="{{ item.title }}" 
     style="width:100%; aspect-ratio:4/3; object-fit:cover; display:block; border-bottom:1px solid #dcdad2;" />
```

Add `imgFile` to each artwork object in the `ARTS` array in the script block:

```js
{ id:'starry', imgFile:'starry-night.jpg', ... }
```

Do the same for the painting panel in the detective screen (left half of zone 2) and the reveal screen — replace the hatched div and placeholder span with the actual image.

---

## Task 3 — Fix the compare button position

**Why:** The button is currently `position:absolute` centred on the dividing line between the two panels. It occludes both canvases at the moment students most want to see them.

**What to do:** Remove the absolute positioning. Place the compare button in a dedicated strip between zone 2 and zone 3 — a narrow bar, full width, with the button centred in it.

```html
<div style="display:flex; justify-content:center; align-items:center; 
            padding:8px 0; border-top:1px solid #dcdad2; 
            border-bottom:1px solid #dcdad2; background:#f4f3ef;">
  <button onMouseDown="{{ startCompare }}" onMouseUp="{{ endCompare }}" 
          onMouseLeave="{{ endCompare }}" onTouchStart="{{ startCompare }}" 
          onTouchEnd="{{ endCompare }}"
          style="border:1px solid #c4c1b6; background:#fbfaf7; color:#1b1b1a; 
                 font-family:'IBM Plex Mono',monospace; font-size:11px; 
                 letter-spacing:0.1em; text-transform:uppercase; 
                 padding:7px 16px; border-radius:2px; cursor:pointer; 
                 user-select:none; white-space:nowrap;"
          style-active="background:#2f4a6b; color:#fff; border-color:#2f4a6b;">
    Hold to compare
  </button>
</div>
```

---

## Task 4 — Fix the backdrop blur on the lock overlay

**Why:** `backdrop-filter: blur` causes rendering problems on older hardware and some Chromebook configurations.

**Find** the lock sheet div (search for `backdrop-filter`):

```html
<div style="...background:rgba(251,250,247,.74); backdrop-filter:blur(2px); -webkit-backdrop-filter:blur(2px);...">
```

**Replace** with a solid overlay, no blur:

```html
<div style="position:absolute; inset:0; background:rgba(244,243,239,0.92); 
            display:flex; align-items:center; justify-content:center;">
```

---

## Task 5 — Fix the canvas selector in masterTick

**Why:** `document.querySelector('canvas')` grabs the first canvas in the DOM. On the reveal screen a stale canvas from the detective screen may still exist, causing the simulation to bind to the wrong element.

**Find** in the script block:

```js
const cv = onFlow ? document.querySelector('canvas') : null;
```

**Replace** with a more specific selector that targets only the visible, active canvas. The detective canvas sits inside the simulation panel div and the reveal canvas inside the reveal container. Add a data attribute to each canvas element in the HTML:

Detective screen canvas:
```html
<canvas data-sim="flow" style="position:absolute; inset:0; width:100%; height:100%; display:block;"></canvas>
```

Reveal screen canvas:
```html
<canvas data-sim="flow" style="position:absolute; inset:0; width:100%; height:100%; display:block; opacity:{{ overlayOpacity }}; transition:{{ overlayTransition }};"></canvas>
```

Update the selector to only find a canvas within the currently visible screen:

```js
const screenClass = s.screen === 'detective' ? '[data-screen="detective"]' : '[data-screen="reveal"]';
const cv = onFlow ? document.querySelector(screenClass + ' canvas[data-sim="flow"]') : null;
```

Add `data-screen` attributes to the detective and reveal wrapper divs accordingly.

---

## Task 6 — Build the Rule 30 simulation

**Why:** The most dramatic reveal, simplest algorithm to implement. Priority build.

**Algorithm:** 1D cellular automaton. Each row of pixels is one generation. Each cell's next state is determined by itself and its two neighbours, according to Rule 30's lookup table: `111→0, 110→0, 101→0, 100→1, 011→1, 010→1, 001→1, 000→0`.

**Controls required** (replace the `stubText` placeholder in the `notFlow` branch — you will need to extend the rendering logic to handle `type:'rule30'`):

- Seed pattern selector: Single seed (one central cell), Twin seed (two cells), Scatter (random sparse row). Implemented as a segmented button control matching the existing palette selector style.
- Colour map selector: Natural (cream/brown echoing the cone snail), Monochrome (black on white), Invert (white on black), Heat (amber gradient).

**Rendering approach:**

Draw top-to-bottom, one row per frame, wrapping when it reaches the bottom. Canvas is black by default. Each live cell draws a 1×1 pixel rectangle in the selected colour.

```js
// Rule 30 lookup
const rule30 = (l, c, r) => (0b00011110 >> (l*4 + c*2 + r)) & 1;
```

Initialise a `Uint8Array` of width `canvas.width`. Each frame, compute the next generation from the current row and draw it at `currentY`, then increment `currentY`. When `currentY >= canvas.height`, reset to 0 and optionally clear (configurable — continuous wrap is more visually interesting).

**Add to `ARTS` array** — update the `conesnail` entry to add:

```js
type: 'rule30',
paramDefaults: { seed:'single', colorMap:'natural' },
```

**Add colour maps:**

```js
RULE30_MAPS = {
  natural:  { bg:'#f5efe0', fg:'#3d1f0a' },
  mono:     { bg:'#ffffff', fg:'#000000' },
  invert:   { bg:'#000000', fg:'#ffffff' },
  heat:     { bg:'#1a0500', fg:'#ff9a2e' }
};
```

**Extend `masterTick`** to handle rule30 screens similarly to flow field, binding to the canvas when `art.type === 'rule30'`.

---

## Task 7 — Build the L-system simulation

**Algorithm:** String rewriting. Start with an axiom string. Each generation, replace every character according to the production rules. Interpret the result with turtle graphics: `F` = draw forward, `+` = turn right by angle, `-` = turn left by angle, `[` = push state, `]` = pop state.

**Controls required:**

- Preset selector (four buttons): Sparse branch, Dense canopy, Geometric, Asymmetric. Each loads a preset axiom, rules, angle, and iterations into the editable fields below.
- Editable fields: Axiom (text input), Rules (textarea, one rule per line e.g. `F=FF+[+F-F-F]-[-F+F+F]`), Angle (number input, degrees), Iterations (range slider 1–6). A Reset to preset button.
- Colour: Single colour picker or a small palette of four preset tree colours.

**Presets:**

```js
LSYSTEM_PRESETS = {
  'Sparse branch':  { axiom:'F', rules:{'F':'F[+F]F[-F]F'}, angle:25.7, iter:5 },
  'Dense canopy':   { axiom:'F', rules:{'F':'FF+[+F-F-F]-[-F+F+F]'}, angle:22.5, iter:4 },
  'Geometric':      { axiom:'F+F+F+F', rules:{'F':'F+F-F-FF+F+F-F'}, angle:90, iter:4 },
  'Asymmetric':     { axiom:'X', rules:{'X':'F+[[X]-X]-F[-FX]+X','F':'FF'}, angle:25, iter:5 }
};
```

**Rendering:** Compute the full string first (warn and cap at iteration 6 to prevent browser hang), then render via turtle graphics on the canvas, scaled to fit. Re-render whenever any parameter changes. This is not animated — it redraws on change.

---

## Task 8 — Build the Rössler attractor simulation

**Algorithm:** Integrate the Rössler system of ODEs using RK4:

```
dx/dt = -y - z
dy/dt = x + ay
dz/dt = b + z(x - c)
```

Project the 3D trajectory onto 2D by discarding `z` (or using a shallow isometric projection for depth). Draw as a continuous line with trail fade.

**Controls required** (four sliders):

- Curve tightness (`a`) — range 0.1 to 0.3, default 0.2. Small label underneath: `a`
- Loop depth (`b`) — range 0.1 to 0.4, default 0.2. Label: `b`
- Spread (`c`) — range 1.0 to 14.0, default 5.7. Label: `c`
- Trail fade — range 0 to 1, default 0.85

**Presets:**

```js
{ name:'Tight coil',  a:0.1,  b:0.1,  c:4.0 },
{ name:'Open orbit',  a:0.2,  b:0.2,  c:5.7 },
{ name:'Drift',       a:0.25, b:0.2,  c:3.5 },
{ name:'Chaos',       a:0.2,  b:0.2,  c:14.0 }
```

**Rendering:** Run the integrator each frame, accumulate ~5,000 points, then draw as a polyline. Use a dark background. Trail fade implemented as a low-opacity fill rect each frame (same technique as the flow field).

---

## Task 9 — Build the Voronoi stippling simulation

**Algorithm:** Lloyd's relaxation on a set of seed points, with colour sampled from the painting image.

**Approach:**

1. Place N seed points randomly across the canvas.
2. Compute the Voronoi diagram using a scan-line approach or Fortune's algorithm. For performance at N ≤ 2000, a brute-force nearest-neighbour scan per pixel is acceptable on a 600×450 canvas — profile first.
3. For each Voronoi region, sample the colour of the corresponding pixel in the painting image (loaded into an offscreen canvas).
4. Fill each region with that colour.
5. Each iteration, move each seed to the centroid of its region (Lloyd's step). Repeat.

**Controls required:**

- Seed count (range slider) — 200 to 2000, default 800
- Relaxation iterations (range slider) — 0 to 20, default 5. Run automatically on change.
- Colour mode (segmented buttons): Exact (sample from painting), Impressionist (add ±20 variance per channel), Monochrome

**Note:** This simulation requires the painting image to be loaded and accessible as pixel data. Use an offscreen `<canvas>` to read pixel values via `getImageData`. This will only work if images are served from the same origin — which they will be on GitHub Pages. CORS is not an issue.

**Presets:**

```js
{ name:'Sparse points', count:300,  iter:8  },
{ name:'Dense field',   count:1500, iter:5  },
{ name:'Pure colour',   count:800,  iter:0  }
```

---

## Task 10 — Add a GitHub Pages config

Add a `_config.yml` at the root (if using Jekyll) or simply ensure the repo has GitHub Pages enabled pointing at the root of `main`. No build step is required — this is a static site.

If the repo owner wants a custom 404, add a `404.html` that redirects to `index.html`.

---

## Known issues not to fix

- The `dc-runtime` template syntax (`{{ }}`, `sc-if`, `sc-for`) is part of the existing framework. Do not attempt to rewrite in vanilla JS — that is a separate decision for the project owner.
- The difficulty labels (Observable / Arguable / Open) are intentionally asymmetric — do not normalise them.
- There is no save, login, or data collection. This is by design.

---

## Definition of done

- [ ] React loads from `/vendor/` with no external requests at boot
- [ ] All six artwork images display in gallery, detective, and reveal screens
- [ ] Compare button sits below the split canvas, not on the divider
- [ ] Lock overlay uses solid background, no blur
- [ ] Canvas selector is screen-specific, not document-wide
- [ ] Rule 30 simulation runs live with seed and colour map controls
- [ ] L-system simulation renders with preset selector and editable rule string
- [ ] Rössler attractor runs live with four parameter sliders
- [ ] Voronoi stippling runs with seed count, iteration, and colour mode controls
- [ ] Experience loads and runs fully offline from a cloned repo

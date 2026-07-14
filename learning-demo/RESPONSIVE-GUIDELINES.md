# Responsive Lomda — Implementation Guidelines

> Drop this file into any fixed-canvas learning module (lomda / interactive
> lesson / kiosk-style HTML app) that has to run on **desktop and tablet**
> (both orientations) without redesigning per breakpoint.
>
> These rules exist because the naive "just center a 1280×720 canvas" approach
> letterboxes on tablet aspect ratios (top/bottom bars floating with empty white
> space above and below), and native HTML5 drag-and-drop silently does nothing
> on touch devices. Both bit us in production — the rules below are the fix.

---

## TL;DR — the four rules

1. **Design at a fixed grid (e.g. 1280×720). Never re-flow per breakpoint.**
2. **Scale-to-fit without distortion, but stretch the canvas to fill the viewport.**
   Content stays on the design grid; chrome anchored to the canvas edges
   (top bars, bottom bars, corner buttons) reaches the *actual* screen edges.
3. **Never use HTML5 drag-and-drop.** Use pointer events (`pointerdown` /
   `pointermove` / `pointerup`) so tablets work. Same rule for draggable popups.
4. **Ship a `viewer.html`** with Desktop / Tablet-Landscape / Tablet-Portrait
   presets and a screen-jump nav so QA can verify every screen at every ratio
   in one click.

If you follow only rule 2 you'll still ship broken drag-drop on tablets.
If you follow only rule 3 you'll still ship letterboxing. Do all four.

---

## Rule 1 — Fix the design grid

Pick a single design size (we use **1280×720**, matching the 720 reference
engine) and lay every screen out in absolute coordinates against it.

```css
html, body {
  width: 100%; height: 100%;
  overflow: hidden;
  background: #fff;   /* MUST match the canvas background — see rule 2 */
}

#app {
  position: absolute;
  /* Width/height are set dynamically by scaleApp() at runtime.
     The 1280×720 fallback keeps the page sensible if JS is still booting. */
  width: 1280px; height: 720px;
  transform-origin: top left;
  background: #fff;
  overflow: hidden;
}

.screen {
  position: absolute; inset: 0;
  /* NO fixed 1280×720 here — .screen fills #app (which fills the viewport).
     Per-screen content is positioned via absolute layout against the
     1280-wide design grid. */
  display: none; overflow: hidden;
}
.screen.active { display: block; }
```

**Why no fixed size on `.screen`?** Because rule 2 stretches `#app` beyond
1280×720 on non-16:9 viewports. Anything anchored to `.screen`'s
`bottom: 0` / `top: 0` (persistent action bars, flag/menu buttons, decorative
bottom strips) must reach the *stretched* edge — otherwise it floats inside
the design grid with empty space behind it.

---

## Rule 2 — Scale-to-fit **and extend the canvas**

The trap: `scale = min(w/1280, h/720)` alone will letterbox. You must also
grow the canvas (in design coordinates) so it fills the viewport.

```js
/* ─── Scale App ──────────────────────────────────────────────
   Scale-to-fit the 1280×720 design while EXTENDING the canvas
   to fill the viewport so chrome anchored to canvas edges
   (flag button at top, bottom bars at bottom) reaches the
   actual screen edges instead of letterboxing.
   - scale  = min(viewportW/1280, viewportH/720)  → no distortion
   - canvas = viewport / scale  → in design coords, fills viewport
   ─────────────────────────────────────────────────────────── */
function scaleApp() {
  const app = document.getElementById('app');
  const scale   = Math.min(window.innerWidth / 1280, window.innerHeight / 720);
  const canvasW = window.innerWidth  / scale;
  const canvasH = window.innerHeight / scale;
  app.style.width     = canvasW + 'px';
  app.style.height    = canvasH + 'px';
  app.style.transform = `scale(${scale})`;
  app.style.left      = '0px';
  app.style.top       = '0px';
}
window.addEventListener('resize', scaleApp);
scaleApp();
```

**How to think about it:** `scale` guarantees no distortion (uniform x/y).
`canvasW`/`canvasH` are the viewport size expressed in *design coordinates*
— always ≥ 1280 in one axis and ≥ 720 in the other. Content that lives on
the 1280-wide grid renders identically; only the extra room around it gets
absorbed into `#app`, which is why edge-anchored chrome reaches the physical
screen edge.

### Placement rules that follow from this

| Content type                                | How to position                                           |
| ------------------------------------------- | --------------------------------------------------------- |
| Illustrations, hero art, per-screen content | Absolute against the 1280×720 design grid — **do not** shift on resize. |
| Persistent top bars, flag/menu buttons      | `top: 0` (or `top: 16` etc.) on `.screen` — reaches viewport top. |
| Persistent bottom bars, action strips       | `bottom: 0` on `.screen` — reaches viewport bottom.       |
| Centered overlays / modals                  | `left: 50%; transform: translateX(-50%)` against the stretched canvas — auto-centers on any ratio. |
| Full-bleed backgrounds                      | `inset: 0` on `.screen` — fills viewport.                 |

**Body background must match the canvas background.** Even with the
stretch, subpixel rounding can expose a 1-px strip during resize. Matching
the body background to the canvas background hides it.

---

## Rule 3 — Pointer events for drag-and-drop (touch support)

HTML5 drag-and-drop (`draggable="true"`, `ondragstart`, `ondrop`, etc.) does
**not** fire on touch devices without a polyfill. On a tablet, the drag-drop
question just… doesn't work. The user taps and nothing happens.

**Never use** `draggable`, `ondragstart`, `ondragover`, `ondrop` attributes
in a lomda. **Always use** a pointer-event helper. The one below is generic
and reusable — copy it verbatim into any project:

```js
/* ═══════════════════════════════════════════════════════════
   Pointer-based drag-and-drop helper.
   Replaces native HTML5 drag so screens work on touch devices
   (tablet, both orientations) without losing mouse support.

   Usage:
     const dnd = createPointerDnd({
       canDrag: (dragId, elem) => boolean,
       onPick:  (dragId, elem) => void,    // pointerdown accepted
       onDrop:  (dragId, targetId) => void,
       onCancel:(dragId) => void,          // dropped outside a target
     });
     dnd.attachSource(elem, dragId);   // safe to call repeatedly
     dnd.attachTarget(elem, targetId);

   Ghost element (a clone of the source) follows the pointer
   in viewport-fixed positioning and is scaled to match the
   #app transform so it visually matches the source.
   ═══════════════════════════════════════════════════════════ */
function createPointerDnd(opts) {
  const targets = new Map(); // elem → targetId
  let active = null;

  function getAppScale() {
    const app = document.getElementById('app');
    const m = app && app.style.transform.match(/scale\(([^)]+)\)/);
    return m ? parseFloat(m[1]) : 1;
  }

  function attachSource(elem, dragId) {
    if (!elem || elem.dataset.pdragAttached === '1') {
      if (elem) elem.dataset.pdragId = dragId;
      return;
    }
    elem.dataset.pdragId = dragId;
    elem.dataset.pdragAttached = '1';
    elem.style.touchAction = 'none';    // critical — stop the browser scrolling
    elem.addEventListener('pointerdown', onSourceDown);
  }

  function attachTarget(elem, targetId) {
    if (!elem) return;
    targets.set(elem, targetId);
  }

  // ... onSourceDown / onMove / onUp — see reference implementation in
  //     methodica-science-mass-measure-01-01/js/main.js (createPointerDnd)
}
```

### Non-negotiable details

- `elem.style.touchAction = 'none'` on every drag source. Without it, the
  browser interprets the touch as a scroll gesture and the drag never starts.
- The **ghost element** (visual proxy that follows the pointer) must be
  positioned `position: fixed` in viewport pixels **and** scaled by the same
  factor as `#app` — otherwise it's the wrong size relative to the source.
  Read the scale from the current `#app` transform (don't cache it — it
  changes on resize).
- Strip `id` attributes from the ghost clone (`ghost.removeAttribute('id')`
  and same for descendants) — duplicate IDs in the DOM break any code that
  does `getElementById`.
- Attach `pointermove`/`pointerup`/`pointercancel` to `document`, not to the
  source — the pointer will leave the source's bounds immediately.
- Hit-test drop targets with `document.elementFromPoint(clientX, clientY)`.
  Do not track hover via `pointerenter` on targets — it doesn't fire while
  a capture is active.

### Same rule for draggable popups / floating windows

If your lomda has draggable feedback popups or modals, they must use
`pointermove` / `pointerup` too — **never** `mousemove` / `mouseup`.
Search-and-replace at project boundaries:

```
mousedown  →  pointerdown
mousemove  →  pointermove
mouseup    →  pointerup
```

---

## Rule 4 — Ship `viewer.html` for QA

A device-preview wrapper that loads your app in an iframe at fixed device
sizes. This is the only way QA (and you) can verify every screen looks
right at every ratio without physically holding a tablet.

Required features:

- **Preset toggles:** Desktop (1920×1080), Tablet Landscape (1024×768),
  Tablet Portrait (768×1024). Add more if the target device list requires.
- **Screen-jump nav** that calls the iframe's `goTo(n)` function directly,
  bypassing per-screen gates (quiz answers, video watch-through, etc.).
  Expose a `window.goTo` in your main app for this purpose.
- **Size readout** in the toolbar so QA can screenshot with dimensions
  visible.
- **Responsive viewer chrome** — the toolbar itself needs `900px` and
  `600px` breakpoints so it doesn't overflow when the QA browser is narrow.

Reference implementation: `viewer.html` in this repo. Copy it, swap the
iframe `src` and the screen list, and you're done.

---

## Anti-patterns — do not do these

| ❌ Don't                                              | ✅ Do instead                                              |
| ----------------------------------------------------- | ---------------------------------------------------------- |
| CSS media queries to re-flow layout at tablet widths  | Fixed design grid + scale-to-fit                           |
| `draggable="true"` + `ondragstart` / `ondrop` attrs   | `createPointerDnd()` (pointer events)                      |
| `addEventListener('mousemove', ...)` for drag         | `addEventListener('pointermove', ...)`                     |
| Fixed 1280×720 on `.screen`                           | `inset: 0` on `.screen`, let `#app` control size           |
| Scaling with `scale = min(...)` and centering with margins | Scaling with `scale = min(...)` and stretching `#app` |
| Body background different from canvas background      | Match them — hides subpixel letterbox flashes              |
| Testing "it works on my laptop"                       | Test via `viewer.html` on all three device presets         |
| Hard-coding pointer offsets in design pixels          | Read the current `#app` scale at drag time — it changes    |

---

## Testing checklist (do all before calling a screen "done")

Open `viewer.html` and cycle through every screen at every preset:

- [ ] **Desktop 1920×1080** — content centered on the 1280-wide grid, chrome flush against viewport edges.
- [ ] **Tablet Landscape 1024×768** — canvas taller than 720 in design coords; bottom bar and top bar reach viewport edges, no empty white strips.
- [ ] **Tablet Portrait 768×1024** — canvas much taller; same edge-flush check. Content still centered horizontally.
- [ ] **Every drag interaction** works with mouse **and** with touch simulation (Chrome DevTools → toggle device toolbar → set to touch).
- [ ] **Every draggable popup** works with mouse and touch.
- [ ] **Resize the browser window** during a drag — the ghost element resizes with the canvas, doesn't drift.
- [ ] **No horizontal or vertical scrollbars** appear at any preset.

If you skip the touch test you will ship a broken tablet build. It has
happened. Do the touch test.

---

## Cache-busting when you deploy responsive changes

The scale/CSS changes are invisible in the diff to a browser holding a
cached copy — QA will report "the fix didn't work" and it's actually just
their cached CSS. Bump a query param on every `<link>` and `<script>` when
you change scaling, canvas size, or drag logic:

```html
<link rel="stylesheet" href="css/style.css?v=2">
<script src="js/main.js?v=2"></script>
```

Increment the number each time. Cheap, avoids a class of ghost bug reports.

---

## Reference implementation

Everything above is live in this repo:

- Scaling: [methodica-science-mass-measure-01-01/js/main.js](methodica-science-mass-measure-01-01/js/main.js) → `scaleApp()`
- Canvas / screen CSS: [methodica-science-mass-measure-01-01/css/style.css](methodica-science-mass-measure-01-01/css/style.css) → `#app` and `.screen`
- Pointer drag-drop: [methodica-science-mass-measure-01-01/js/main.js](methodica-science-mass-measure-01-01/js/main.js) → `createPointerDnd()`
- Device-preview viewer: [methodica-science-mass-measure-01-01/viewer.html](methodica-science-mass-measure-01-01/viewer.html)

Copy the four pieces, adjust the design grid dimensions if your project
uses something other than 1280×720, and you're on the paved road.

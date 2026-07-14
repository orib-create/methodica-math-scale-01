# Project instructions — Metodika lomda

This is a fixed-canvas HTML learning module (lomda) that must run on
**desktop and tablet, both orientations**. The rules below are hard
requirements — they encode fixes for bugs that shipped to production.
Do not violate them without explicit permission from the user.

Full rationale + code samples: [RESPONSIVE-GUIDELINES.md](RESPONSIVE-GUIDELINES.md).
Read that file the first time you touch scaling, drag-drop, or layout in
this repo. The four rules below are the enforceable summary.

**Mechanical enforcement:** a pre-commit hook at
[.githooks/pre-commit](.githooks/pre-commit) blocks commits that violate
these rules. Activate it once per clone with:
`git config core.hooksPath learning-demo/.githooks` (path is relative to the
repo root, not this `learning-demo` folder). If a commit is blocked, fix the
issue — do not bypass with `--no-verify` unless explicitly authorized.

---

## Hard rules

### R1 — Fixed design grid, no per-breakpoint reflow

Every screen is laid out in absolute coordinates against a **1280×720**
design grid. Do not add CSS media queries that re-flow content at tablet
widths. Do not resize or reposition per-screen content on window resize.
The canvas scales; the layout does not.

### R2 — Scale-to-fit AND extend the canvas

`scaleApp()` must (a) compute `scale = min(vpW/1280, vpH/720)` so content
is never distorted, AND (b) set `#app` width/height to `viewport / scale`
so the canvas fills the viewport in design coordinates. `.screen` uses
`inset: 0` — **never** a fixed `1280×720`. Chrome anchored to
`top: 0` / `bottom: 0` on `.screen` must reach the actual viewport edge
on tablet aspect ratios. If you see letterboxing, R2 is broken.

Body background must match the canvas background (currently `#fff`).

### R3 — Pointer events only, never HTML5 drag or mouse events for drag

**Forbidden** in this codebase:
- `draggable="true"` attribute
- `ondragstart`, `ondragover`, `ondrop`, `ondragend` attributes or listeners
- `addEventListener('mousemove' | 'mouseup' | 'mousedown', ...)` inside any
  drag or draggable-popup handler

**Required** instead: `createPointerDnd()` (see `js/main.js`) for
drag-and-drop, and `pointermove` / `pointerup` / `pointerdown` for any
draggable UI (popups, floating panels, sliders). Every drag source must
set `touch-action: none`. Ghost elements must read the current `#app`
scale at drag time — do not cache it.

If you add a new interaction that involves dragging *anything*, use
pointer events. No exceptions.

### R4 — Verify in `viewer.html` before declaring done

`viewer.html` loads the app in an iframe at three presets: Desktop
1920×1080, Tablet Landscape 1024×768, Tablet Portrait 768×1024, and
exposes a screen-jump nav via `window.goTo(n)`. Any change to a screen,
scaling, or drag interaction must be verified at **all three presets**
before you tell the user it's done. If you can't run a browser, say so
explicitly — do not claim visual success from code inspection alone.

---

## When you change scaling, canvas, or drag code

Bump the `?v=N` query param on the `<link>` and `<script>` tags in
`index.html`. Dev browsers cache aggressively; without a bump, the user
will report "the fix didn't work" and you'll waste a round trip
debugging a cache.

## When adding a new screen

- Position content absolutely against the 1280×720 grid.
- Persistent chrome (top bar, bottom bar, corner buttons) anchors to
  `.screen` with `top: 0` / `bottom: 0`, not to fixed pixel offsets from
  a `720px` bottom.
- Expose the screen in `window.goTo` / the screen array so `viewer.html`
  can jump to it.
- Test all drag interactions with Chrome DevTools device toolbar in
  touch mode. Mouse-only testing is not sufficient.

## Red flags to catch in review

If you see any of these in a diff, stop and flag it:

- Any `draggable`, `ondragstart`, `ondrop` attribute → replace with
  `createPointerDnd()`.
- Any `mousemove` / `mouseup` inside a drag or popup handler → replace
  with pointer events.
- Fixed `width: 1280px; height: 720px` on `.screen` → replace with
  `inset: 0`.
- `scaleApp()` computing `left`/`top` offsets to center the canvas →
  R2 is being undone; the canvas must stretch, not center.
- CSS media queries in `style.css` that change content layout → R1
  violation.
- New CSS/JS changes without a `?v=` bump on the includes → will ship
  as a cache miss.

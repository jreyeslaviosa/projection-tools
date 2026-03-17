# Projection Tools — Agent Guide

You are working on a suite of browser-based HTML tools for projection mapping calculation. Each tool is a **single self-contained HTML file** with embedded CSS and JavaScript. No build step, no dependencies (except html2canvas via CDN).

---

## File Inventory

| File | Purpose |
|---|---|
| `wall-to-pixels.html` | Given a wall size, calculates content resolution locked to **1200px height** (WUXGA 1920×1200). Includes multi-projector edge blending calculator. |
| `lens-calculator.html` | Given projector resolution + lens throw ratio + wall dimensions, calculates throw distance, projected image size, pixel accounting, and room feasibility. |

---

## Design System

**Always match these values exactly. Do not introduce new colours, fonts, or layout patterns.**

### Colors
```
Background:       #000
Card bg:          #0a0a0a
Card border:      #1a1a1a
Input bg:         #111
Input border:     #2a2a2a   (focused: #555)
Label text:       #555
Body text:        #d4d4d4
Secondary text:   #ccc
Dim text:         #666 / #444 / #333
Bright value:     #fff
OK / positive:    #6dbb6d
Warning / error:  #cc6644
Bleed colour:     rgba(160,60,20,…)
```

### Typography
- Body font: `-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif`
- Labels: `0.7rem`, `font-weight: 600`, `text-transform: uppercase`, `letter-spacing: 0.05em`, color `#555`
- Primary values: `1.8rem`, `font-weight: 700`, `color: #fff`
- Secondary values: `0.9rem`, `font-weight: 600`, `color: #ccc`
- Section title: `0.65rem`, uppercase, `letter-spacing: 0.06em`, `color: #444`

### Structural Classes (reuse, don't reinvent)
```css
.card           /* main container, max-width 720px */
.row            /* flex row of .field items, gap 1rem, wraps */
.field          /* flex column, label + input/select */
.result-box     /* centered result display, dark bg */
.result-pair    /* two .result-box side by side */
.info-grid      /* 2-col grid of .info-card */
.info-card      /* small stat tile with .lbl and .val */
.section-box    /* grouped data rows */
.sb-row         /* label/value row inside .section-box */
.tabs           /* tab navigation bar */
.tab-btn        /* individual tab button */
.tab-content    /* tab panel, hidden unless .active */
.lock-pill      /* toggle pill button, active = white bg */
.diagrams       /* flex container for side-by-side canvases */
.footer         /* bottom bar with brand + button row */
.btn            /* dark button */
.btn-primary    /* white/inverted button */
```

### Canvas conventions
- Side-by-side pair: `width="320" height="180"`
- Full-width single: `width="640" height="380"` (or `width="650" height="220"` for blend maps)
- Background: always `#111`
- Padding inside canvas: `pad = 28–44` depending on label space needed
- Canvas labels (e.g. "FRONT VIEW"): `9px`, color `#333`, top-left corner at `(6, 12)`

---

## Calculation Reference

### Core Formula
```
throw_distance = image_width_meters × throw_ratio
```

### Solver — three fit modes
Given `wallWm, wallHm` (wall in meters), `projW, projH` (projector pixels), `throwRatio`:

**Lock Width** — image width = wall width, height may bleed above/below
```
imageWm = wallWm
imageHm = wallWm / (projW / projH)
```

**Lock Height** — image height = wall height, width may bleed left/right
```
imageHm = wallHm
imageWm = wallHm × (projW / projH)
```

**Fit Both** — no bleed, image fits inside wall (may leave margins)
```
if (wallWm/wallHm) <= (projW/projH):  use Lock Width  → imageHm ≤ wallHm ✓
else:                                  use Lock Height → imageWm ≤ wallWm ✓
```

### Pixel Accounting
```
mpx        = imageWm / projW          // meters per pixel
wallInPixW = wallWm / mpx             // wall width in projector pixel space
wallInPixH = wallHm / mpx

activePixW = min(projW, wallInPixW)   // pixels landing ON wall (width)
activePixH = min(projH, wallInPixH)   // pixels landing ON wall (height)
bleedPixW  = max(0, projW - wallInPixW)  // pixels outside wall (total, both sides)
bleedPixH  = max(0, projH - wallInPixH)

pixelUtil  = (activePixW × activePixH) / (projW × projH)
wallCoverage = min(1, imageWm × imageHm / wallWm / wallHm)
```

Physical bleed extent (per side = half these values):
```
bleedWm = bleedPixW × mpx
bleedHm = bleedPixH × mpx
```

### Unit Conversion
```javascript
function toMeters(val, unit) {
  return val * { m:1, cm:0.01, mm:0.001, ft:0.3048, in:0.0254 }[unit];
}
```

### Pixel Density
```
px/m = projW / imageWm
PPI  = (projW / imageWm) / 39.3701
mm/px = (imageWm × 1000) / projW
```

---

## How to Add a New Tool

1. **Copy `lens-calculator.html`** as a starting point — it has the full CSS design system embedded.
2. **Keep all CSS variables and class names identical** — this ensures visual consistency across tools.
3. **Wire all inputs** to a single `calculate()` function via `addEventListener('input', calculate)`.
4. **Internally store all distances in meters**, convert at input and display boundaries only.
5. **Always include:**
   - A project name input (`.project-input`)
   - A date stamp (`#dateStamp`)
   - Canvas diagrams (front view + at least one other)
   - "Copy Spec" button that writes a plain-text summary to clipboard
   - "Save as Image" button using `html2canvas`
   - Footer with tool name brand label

---

## Conventions

- **No frameworks** — pure vanilla JS, no React/Vue/etc.
- **Single file** — all CSS and JS inline in the HTML.
- **External CDN only**: html2canvas from `https://cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js`
- **`const $ = id => document.getElementById(id)`** — standard short alias, always define at top.
- **Tabs**: active tab gets `.active` on both `.tab-btn` and its corresponding `.tab-content`.
- **Lock/mode toggles**: pill-style buttons, active state = white background. Store state in a `let` variable, not DOM queries.
- **OK/Warn coloring**: use CSS classes `.ok` (green) and `.warn` (orange-red) on `.info-card` or inline `<span class="ok/warn">`.
- **Aspect ratios**: simplify using `gcd(Math.round(w), Math.round(h))`.
- **All physical values**: display in both metric (`fmtDist()`) and imperial (`toImperial()`) where space allows.
- **Screenshot**: always hide the button row before html2canvas, restore after.

---

## Planned / Future Tools

- `room-planner.html` — 2D top-view room layout with multiple projectors, surfaces, and throw cone visualization
- `offset-calculator.html` — Lens shift / keystone correction calculator
- `blend-depth.html` — Multi-projector blend zone calculator with per-projector depth requirements

---

## Owner Context

- Projection mapping workflow uses **MadMapper** for content mapping
- Primary projector: **WUXGA 1920×1200**
- Content resolution is typically locked to **1200px height**, width driven by wall aspect ratio
- Tools are used on-site for rapid setup calculations — minimal UI, keyboard-friendly, dark theme for low-light environments

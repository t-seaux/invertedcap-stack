---
name: design-language
description: >
  Inverted Capital design language reference. Apply whenever Tom asks to build,
  render, or export any visual artifact — charts, diagrams, skill maps, data
  visualizations, LP letter exhibits, or any HTML/canvas output. Also trigger
  when Tom says "use my design language", "match the visual style", "make it
  look like the skill map", or any variant requesting that an artifact conform
  to Inverted's established aesthetic. Always read this skill before writing
  any HTML, canvas, or SVG code for Tom. This skill is the single source of
  truth for colors, typography, spacing, card structure, export conventions,
  and interactive effects.
---

# Inverted Capital Design Language

Single source of truth for all visual artifacts. Read before writing any HTML,
canvas, or SVG for Tom.

---

## Typography

**Font family:** `'Courier New', Courier, monospace` for skill maps and stack
page. `'JetBrains Mono', monospace` for charts, exhibits, and data visuals.
Never use Inter, Roboto, Arial, or system-ui.

### Embedding JetBrains Mono (charts/exhibits)

Google Fonts does NOT load reliably in headless/Playwright environments.
Always embed from system TTF:

```bash
apt-get install -y fonts-jetbrains-mono
```

```python
import base64

weights = {
    '400': '/usr/share/fonts/truetype/jetbrains-mono/JetBrainsMono-Regular.ttf',
    '500': '/usr/share/fonts/truetype/jetbrains-mono/JetBrainsMono-Medium.ttf',
    '600': '/usr/share/fonts/truetype/jetbrains-mono/JetBrainsMono-SemiBold.ttf',
}
faces = []
for w, path in weights.items():
    with open(path, 'rb') as f:
        b64 = base64.b64encode(f.read()).decode()
    faces.append(
        f"@font-face {{ font-family: 'JetBrains Mono'; font-weight: {w}; "
        f"src: url(data:font/truetype;base64,{b64}) format('truetype'); }}"
    )
font_css = '\n'.join(faces)
```

---

## Color Palette

### Backgrounds
| Role | Hex |
|---|---|
| Page / body bg | `#0d1117` |
| Card / panel bg | `#161b22` |
| Ad hoc pill / elevated bg | `#0d1117` |
| Hover elevated | `#21262d` |

### Borders & Dividers
| Role | Hex |
|---|---|
| Card border (default) | `#30363d` |
| Card border (hover) | `#3d444d` |
| Subtle internal divider | `#21262d` |
| Scheduled pill border | `#c3c2bd` |

### Text
| Role | Hex |
|---|---|
| Primary (titles, values, names) | `#fffffb` (skill map) / `#e6edf3` (charts) |
| Secondary (descriptions, axis labels) | `#b1bac4` (skill map) / `#8b949e` (charts) |
| Tertiary (eyebrows, source lines) | `#8b949e` (skill map) / `#484f58` (charts) |

### Skill Pills
| Type | Background | Text | Border |
|---|---|---|---|
| Scheduled | `#f0efea` | `#2c2c2a` | `#c3c2bd` |
| Ad Hoc | `#0d1117` | `#8b949e` | `#30363d` |

### Database Line Colors
| Database | Hex |
|---|---|
| Opportunities | `#7F77DD` |
| People | `#1D9E75` |
| Company Updates | `#BA7517` |
| Notes | `#888780` |
| -1 Scanner | `#D85A30` |

### Chart / Data Series Colors
| Series | Hex |
|---|---|
| p95 / primary accent | `#f97316` |
| p90 / secondary accent | `#fb923c` |
| p75 / neutral light | `#e6edf3` |
| p25 / neutral gray | `#6b7280` |

---

## Card Structure

### Skill Map / Stack Page Cards

```css
.sm-fn-card, .sm-db-card {
  border: 0.5px solid #30363d;
  border-radius: 10px;
  background-color: #161b22;
  position: relative;
  overflow: hidden;
  isolation: isolate;
}
.sm-fn-card { padding: 10px 12px; }
.sm-db-card { padding: 12px 14px 10px; border-left-width: 3px; }
```

DB cards have a 3px colored left border matching their database line color.

### Chart / Exhibit Cards

```css
.card {
  background: #161b22;
  border: 1px solid #30363d;
  border-radius: 6px;
  padding: 28px 32px 26px;
  width: 624px;  /* Google Docs content width */
}
```

Internal layout order:
1. Eyebrow — `#484f58`, uppercase, 8–9px, `letter-spacing: 0.14em`
2. Title — `#e6edf3`, 15–18px, weight 600
3. Subtitle — `#8b949e`, 9–10px
4. Canvas / visual
5. Legend — `border-top: 1px solid #21262d`, `padding-top: 14px`, flex row
6. Source line — `#484f58`, uppercase, 7.5px, `margin-top: 12px`

---

## Shine Hover Effect (Skill Map)

All cards and pills use a diagonal light-sweep that animates IN on hover
and snaps back INSTANTLY on leave (no exit transition). This is intentional —
it prevents adjacent cards from showing simultaneous shine during cursor movement.

**Card implementation:**
```css
/* Default state — no background-position transition */
.sm-fn-card {
  background-image: linear-gradient(120deg,
    transparent 0%, transparent 40%,
    rgba(255,255,255,0.06) 47%, rgba(255,255,255,0.13) 50%,
    rgba(255,255,255,0.06) 53%, transparent 60%, transparent 100%);
  background-size: 300% 100%;
  background-position: 100% 0;
  transition: border-color 0.3s ease;  /* only border transitions by default */
}

/* Hover state — shine animates IN */
.sm-fn-card:hover {
  background-position: 0% 0;
  border-color: #3d444d;
  transition: background-position 0.8s ease, border-color 0.3s ease;
}
```

**Scheduled pill shine** (dark background, use white glow):
Same gradient pattern. Hover triggers `background-position: 0% 0` + `transition: background-position 0.8s ease`.

**Ad hoc pill shine** (light `#f0efea` background, use dark glow):
```css
background-image: linear-gradient(120deg,
  transparent 0%, transparent 35%,
  rgba(0,0,0,0.06) 44%, rgba(0,0,0,0.12) 50%,
  rgba(0,0,0,0.06) 56%, transparent 65%, transparent 100%);
```

**Key rule:** `background-position` transition is ONLY set on `:hover`. Default
state has NO `background-position` transition → instant snap-back on leave.

---

## SVG Connection Lines (Skill Map)

Three-layer approach per connection: base (dim, always visible) + glow halo
(blurred, breathing) + bright core (sharp, breathing). Each line gets a
staggered duration and delay for organic feel.

```javascript
var dur = (2.8 + (i % 4) * 0.4).toFixed(1);  // 2.8s – 4.0s
var delay = ((i * 0.4) % 2.2).toFixed(2);

// 1) Base: stroke-width 1.5, opacity 0.2, no animation
// 2) Glow: stroke-width 2.5, filter url(#glow), opacity animates 0.05→0.35→0.05
// 3) Core: stroke-width 2, opacity animates 0.12→0.55→0.12
// Both glow and core use: calcMode="spline", keySplines="0.45 0 0.55 1; 0.45 0 0.55 1"
```

Glow filter definition:
```xml
<filter id="glow">
  <feGaussianBlur stdDeviation="0.6" result="blur"/>
  <feMerge>
    <feMergeNode in="blur"/>
    <feMergeNode in="SourceGraphic"/>
  </feMerge>
</filter>
```

Connection mapping (fn → db → color):
- Pipeline Management → Opportunities `#7F77DD`, People `#1D9E75`, -1 Scanner `#D85A30`
- Intro Management → Opportunities `#7F77DD`, People `#1D9E75`
- Portfolio Management → Opportunities `#7F77DD`, Company Updates `#BA7517`
- Diligence Management → Opportunities `#7F77DD`, People `#1D9E75`, Notes `#888780`
- Research Management → Notes `#888780`

---

## Chart / Canvas Conventions

### Axes
- Y-axis labels: right-aligned, 7.5px JetBrains Mono, `#484f58`, `$XM` format, integers only (`Math.round()`)
- X-axis: year quarters as two lines — `Q1` in `#8b949e`, year in `#484f58`, 7–7.5px
- Grid lines: `#21262d`, 0.5px, horizontal only; bottom baseline `#30363d`
- No vertical grid lines, no axis border

### Lines
- `lineWidth: 1.4–1.5`, `lineJoin: 'round'`, `lineCap: 'round'`
- No dots/markers on data points (default)
- All series solid — dashed lines only when explicitly requested
- Draw order: bottom series first (p25 → p75 → p90 → p95)

### Labels
- Start (left): `600 8px`, left of first point, `dy = -4` above line
- End (right): value `600 8.5px` + percentile `7.5px #8b949e` below, `x + 6–8px` right of last point
- All values integers, no decimals

### Legend
```css
.legend-swatch { width: 14–16px; height: 2px; border-radius: 1px; }
```

---

## Responsive Scaling (Stack Page)

Design width is fixed at 1200px. Scale via JS transform:

```javascript
var DESIGN_WIDTH = 1200;
function scaleAndDraw() {
  var wrapper = document.getElementById('responsive-wrapper');
  var inner = document.getElementById('responsive-inner');
  var scale = wrapper.offsetWidth / DESIGN_WIDTH;
  inner.style.transform = 'scale(' + scale + ')';
  wrapper.style.height = inner.scrollHeight * scale + 'px';
}
window.addEventListener('load', scaleAndDraw);
window.addEventListener('resize', scaleAndDraw);
```

---

## Export Conventions (PNG / Google Docs)

| Parameter | Value |
|---|---|
| Google Docs content width | 624px logical (6.5" at 96dpi, 1" margins) |
| Canvas logical width | 560px (624px card minus 32px padding each side) |
| Export scale | 4x device pixel ratio → 2,496px output width |
| Min render wait | 2,500ms (font + canvas settle time) |

### Playwright Recipe

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page(
        viewport={"width": 800, "height": 900},
        device_scale_factor=4
    )
    page.goto("file:///path/to/file.html")
    page.wait_for_timeout(2500)
    card = page.query_selector('.card')
    box = card.bounding_box()
    page.screenshot(
        path="/Users/tomseo/Downloads/output-4x.png",
        clip={"x": box["x"], "y": box["y"],
              "width": box["width"], "height": box["height"]}
    )
    browser.close()
```

Always clip to the `.card` element. Never screenshot full page.
Save HTML to outputs for reference. Present PNG via `present_files`.

---

## Page / Body Wrapper

For PNG export (no padding — clip captures card flush):
```css
body { background: #0d1117; font-family: 'JetBrains Mono', monospace; padding: 0; margin: 0; }
```

For browser preview: `padding: 40px 48px`.

For stack page (iframe-embedded): `background: transparent`.

---

## Typography Scale Reference

| Role | Size | Weight | Color | Font |
|---|---|---|---|---|
| Stack page card title | 12–13px | 500 | `#fffffb` | Courier New |
| Stack page eyebrow | 9px | 400, uppercase, tracking 0.13em | `#8b949e` | Courier New |
| Stack page skill pill | 11px | 400 | `#2c2c2a` / `#8b949e` | Courier New |
| Chart card title | 15–18px | 600 | `#e6edf3` | JetBrains Mono |
| Chart eyebrow | 8–9px | 500, uppercase, tracking 0.14em | `#484f58` | JetBrains Mono |
| Chart axis label | 7–7.5px | 400 | `#484f58` | JetBrains Mono |
| Chart value label | 8–8.5px | 600 | series color | JetBrains Mono |
| Chart legend | 8px | 400, tracking 0.04em | `#8b949e` | JetBrains Mono |
| Chart source line | 7.5px | 400, uppercase, tracking 0.06em | `#484f58` | JetBrains Mono |

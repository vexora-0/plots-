# Rachakonda Layout Map — Integration Guide

## Overview

This project generates an **interactive SVG plot map** for Rachakonda Village (342 plots). The map is built from raw plot data (`all_plots.json`) through a Python build pipeline (`build_map.py`) into a final `map.svg` that can be embedded in any web or mobile application.

---

## Project Structure

```
srujan/
├── all_plots.json              # Master plot data (342 plots)
├── build_map.py                # SVG generator (Python)
├── layout.svg                  # Base SVG with roads, grid, background
├── map-project/
│   ├── index.html              # Reference web app
│   ├── map.js                  # Interactivity (click, modal, stats)
│   ├── map.css                 # Styling (colors, hover, modal)
│   └── map.svg                 # Generated output (embed this)
```

---

## File Reference

### `all_plots.json` — Plot Data

Array of plot objects. Each plot has:

```json
{
  "lbl": "103/A",          // Plot label (unique ID)
  "st": "available",       // Status: available | sold | booked | registered
  "sqyd": "400",           // Area in square yards
  "left": 35.964,          // CSS left% (used by build pipeline, not for integration)
  "top": 55.239,           // CSS top% (used by build pipeline, not for integration)
  "width": 3.439,          // CSS width% (used by build pipeline)
  "height": 3.299          // CSS height% (used by build pipeline)
}
```

**To update a plot's status**, change the `"st"` field and rebuild:

```bash
python build_map.py
```

### `map.svg` — Generated Map

The SVG has a viewBox of `0 0 595 842` (A4 portrait). It contains:

1. **Background layer** — roads, green areas, decorative elements (from `layout.svg`)
2. **`<g id="plots">`** — All 342 plot shapes (polygons/rects)
3. **`<g id="plot-labels">`** — Text labels for each plot

Each plot element looks like:

```xml
<polygon
  id="plot-103/A"
  class="plot available"
  points="218.0,408.7 228.3,408.7 228.3,419.2 218.0,419.2"
  data-label="103/A"
  data-status="available"
  data-sqyd="400"
/>
```

**Key attributes:**
| Attribute | Description |
|-----------|-------------|
| `id` | `plot-{label}` — unique element ID |
| `class` | `plot {status}` — CSS class for styling |
| `data-label` | Plot number/label |
| `data-status` | `available`, `sold`, `booked`, `registered` |
| `data-sqyd` | Plot area in sq. yards |

### `map.js` — Interactivity Reference

Minimal JS (~150 lines) that:
1. Fetches and injects `map.svg` into the DOM
2. Counts plots by status and updates stat counters
3. Handles plot click → opens modal with plot details
4. Provides an "Enquire" CTA for available plots

### `map.css` — Styling Reference

Plot colors are defined via CSS classes on the SVG elements:

```css
.plot.available  { fill: rgba(61,140,64,0.35);   stroke: rgba(61,140,64,0.8);   }
.plot.sold       { fill: rgba(155,45,155,0.35);   stroke: rgba(155,45,155,0.8);  }
.plot.booked     { fill: rgba(192,96,192,0.35);   stroke: rgba(192,96,192,0.8);  }
.plot.registered { fill: rgba(123,52,184,0.35);   stroke: rgba(123,52,184,0.8);  }

.plot:hover      { fill: rgba(60,140,255,0.45);   stroke: #4a9eff;              }
.plot.selected   { fill: rgba(200,168,75,0.35);   stroke: #c8a84b;              }
```

---

## Integration Options

### Option A: Embed as Web Component (Next.js, React, Vue, etc.)

The simplest approach — load the SVG inline and style/interact with it via JS/CSS.

#### 1. Copy these files into your project

```
map.svg         → public/map.svg (or assets/)
all_plots.json  → src/data/plots.json (for data lookups)
```

#### 2. Load SVG inline (React/Next.js example)

```jsx
import { useEffect, useRef, useState } from 'react';
import plotsData from '@/data/plots.json';

export default function PlotMap() {
  const containerRef = useRef(null);
  const [selected, setSelected] = useState(null);

  useEffect(() => {
    fetch('/map.svg')
      .then(r => r.text())
      .then(svg => {
        containerRef.current.innerHTML = svg;
        // Bind click handlers to all plots
        containerRef.current.querySelectorAll('.plot').forEach(el => {
          el.addEventListener('click', () => {
            const label = el.getAttribute('data-label');
            const status = el.getAttribute('data-status');
            const sqyd = el.getAttribute('data-sqyd');
            setSelected({ label, status, sqyd });
          });
        });
      });
  }, []);

  return (
    <div>
      <div ref={containerRef} style={{ width: '100%' }} />
      {selected && (
        <div className="modal">
          <h2>Plot {selected.label}</h2>
          <p>Status: {selected.status}</p>
          <p>Area: {selected.sqyd} sq.yd</p>
        </div>
      )}
    </div>
  );
}
```

#### 3. Style with your own CSS

Override the plot colors by targeting the CSS classes:

```css
/* Your custom colors */
.plot.available  { fill: rgba(0, 200, 100, 0.4); stroke: #00c864; }
.plot.sold       { fill: rgba(255, 50, 50, 0.4);  stroke: #ff3232; }
.plot.booked     { fill: rgba(255, 165, 0, 0.4);  stroke: #ffa500; }
.plot.registered { fill: rgba(0, 100, 255, 0.4);  stroke: #0064ff; }
```

### Option B: Flutter Integration

#### 1. Use `flutter_svg` for rendering

```yaml
# pubspec.yaml
dependencies:
  flutter_svg: ^2.0.0
```

#### 2. Load and display the SVG

```dart
import 'package:flutter_svg/flutter_svg.dart';

class PlotMap extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return InteractiveViewer(
      minScale: 0.5,
      maxScale: 4.0,
      child: SvgPicture.asset(
        'assets/map.svg',
        width: double.infinity,
      ),
    );
  }
}
```

#### 3. For interactive plots (tap detection)

`flutter_svg` doesn't support tap on individual elements. Use one of these approaches:

**Approach A: WebView (recommended for full interactivity)**

```dart
import 'package:webview_flutter/webview_flutter.dart';

class PlotMapWebView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return WebViewWidget(
      controller: WebViewController()
        ..loadFlutterAsset('assets/map-project/index.html')
        ..setJavaScriptMode(JavaScriptMode.unrestricted)
        ..addJavaScriptChannel('PlotChannel', onMessageReceived: (msg) {
          // msg.message contains the plot label
          final plotLabel = msg.message;
          // Show your Flutter dialog
        }),
    );
  }
}
```

Add this to `map.js` to bridge to Flutter:

```js
// Inside selectPlot(), add:
if (window.PlotChannel) {
  PlotChannel.postMessage(JSON.stringify({ label, status, sqyd }));
}
```

**Approach B: Custom painter with hit testing**

Parse `all_plots.json` and draw plots manually using `CustomPainter`, then use `GestureDetector` for tap detection. This gives full native control but requires reimplementing the coordinate system.

```dart
import 'dart:convert';

class PlotPainter extends CustomPainter {
  final List<Map<String, dynamic>> plots;
  PlotPainter(this.plots);

  @override
  void paint(Canvas canvas, Size size) {
    // Scale from SVG viewBox (595x842) to widget size
    final scaleX = size.width / 595;
    final scaleY = size.height / 842;

    for (final plot in plots) {
      final status = plot['st'];
      final paint = Paint()
        ..color = _colorForStatus(status)
        ..style = PaintingStyle.fill;

      // Draw using the polygon points from map.svg
      // (parse from SVG or pre-compute from build_map.py)
    }
  }

  Color _colorForStatus(String status) {
    switch (status) {
      case 'available': return Color(0x593d8c40);
      case 'sold': return Color(0x599b2d9b);
      case 'booked': return Color(0x59c060c0);
      case 'registered': return Color(0x597b34b8);
      default: return Color(0x593d8c40);
    }
  }
}
```

### Option C: Static Image Export

For simpler use cases (PDF, print, email), export the SVG to PNG:

```bash
# Using Inkscape CLI
inkscape map.svg --export-type=png --export-width=2000 -o map.png

# Using Chrome headless
chrome --headless --screenshot=map.png --window-size=1200,1700 index.html
```

---

## Common Customizations

### Change plot colors

Edit `map.css` or override in your app's CSS:

```css
.plot.available { fill: #your-color; stroke: #your-border; }
```

### Remove plot number labels

In `build_map.py`, comment out or delete the text generation block (search for `<text x=`), then rebuild:

```bash
python build_map.py
```

Or hide labels with CSS:

```css
#plot-labels { display: none; }
```

Or hide selectively:

```css
#plot-labels text { font-size: 0; }  /* hide all */
```

### Update a plot's status

1. Edit `all_plots.json` — change the `"st"` field:
   ```json
   { "lbl": "42", "st": "sold", ... }
   ```
2. Rebuild: `python build_map.py`

Or update at runtime via JavaScript:

```js
const plot = document.getElementById('plot-42');
plot.classList.remove('available');
plot.classList.add('sold');
plot.setAttribute('data-status', 'sold');
```

### Add custom data attributes

In `build_map.py`, find the element generation line and add attributes:

```python
# Search for: data-sqyd="{sqyd}"
# Add after it:  data-price="{price}" data-facing="{facing}"
```

You'll need to add the corresponding fields to `all_plots.json`.

### Change the font

The SVG labels use `Rajdhani` (Google Font). To change:

1. In `build_map.py`, search for `font-family="Rajdhani` and replace
2. Rebuild with `python build_map.py`

### Adjust plot opacity

```css
.plot.available { fill-opacity: 0.5; }  /* more opaque */
.plot.available { fill-opacity: 0.1; }  /* more transparent */
```

### Make labels larger/smaller

```css
#plot-labels text { font-size: 8px !important; }
```

### Highlight a specific plot programmatically

```js
document.getElementById('plot-42').classList.add('selected');
```

---

## Rebuilding the Map

After editing `all_plots.json` or `build_map.py`:

```bash
python build_map.py
```

This reads `all_plots.json` + `layout.svg` → outputs `map-project/map.svg`.

**Requirements:** Python 3.6+ (no external dependencies).

---

## Data Flow Diagram

```
┌──────────────────┐
│  all_plots.json  │  342 plots with status, area, coordinates
└────────┬─────────┘
         │
         ▼
┌──────────────────┐     ┌──────────────┐
│  build_map.py    │◄────│  layout.svg  │  Background roads, grid, decorations
│                  │     └──────────────┘
│  • Reads plots   │
│  • Transforms    │
│    coordinates   │
│  • Generates     │
│    SVG elements  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│    map.svg       │  Final map with all plots as clickable SVG shapes
└────────┬─────────┘
         │
    ┌────┴──────────────────────┐
    ▼                           ▼
┌──────────┐          ┌─────────────────┐
│ index.html│          │ Your App        │
│ map.js   │          │ (Next.js,       │
│ map.css  │          │  Flutter, etc.) │
└──────────┘          └─────────────────┘
  Reference             Your integration
  web app
```

---

## SVG Coordinate System

- **ViewBox:** `0 0 595 842` (A4 portrait proportions)
- **Origin:** Top-left corner (0,0)
- **Units:** Unitless SVG coordinates (not pixels)
- Plots are `<polygon>` elements with 4-point coordinates (TL, TR, BR, BL)
- Some plots are trapezoids (irregular quads) due to diagonal roads

---

## API-Driven Status Updates

For a live system, you can skip rebuilding and update statuses client-side:

```js
// Fetch latest status from your API
const response = await fetch('/api/plots');
const plots = await response.json();

plots.forEach(({ label, status }) => {
  const el = document.getElementById(`plot-${label}`);
  if (el) {
    el.className = `plot ${status}`;
    el.setAttribute('data-status', status);
  }
});
```

This lets `map.svg` remain static while statuses update dynamically.

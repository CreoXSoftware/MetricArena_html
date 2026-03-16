# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Metric Arena** is a single-file, zero-dependency athlete performance analytics web app. All logic — styles, HTML, and JavaScript — lives in `index.html`. There is no build system, no package manager, and no backend. Development means editing `index.html` directly and opening it in a browser (use a local HTTP server, not `file://`, to avoid File API restrictions).

## Architecture

### Single-file structure (in document order)
1. `<style>` — all CSS with CSS custom properties (design tokens in `:root`)
2. `<body>` — two screens: upload screen (initial) and dashboard (hidden until file loaded)
3. `<script>` — all application logic, inline at the bottom

### Key global state variables
- `sessionData` — raw parsed records from CSV/binary file
- `processedData` — gravity-corrected output (linear acceleration, speed, distance, time per record)
- `splits` — array of `{ id, name, tStart, tEnd, metrics, isCombined?, sourceIds?, sourceSplits? }`
- `selectedSplitId` — currently highlighted split
- `chartView` — `{ tStart, tEnd }` visible time window (zoom state)
- `thresholds` — 12 configurable threshold values driving metric computation
- `resizingBoundary` — active drag handle state: `{ splitId, side, startT }`
- `accelChartInfo` / `speedChartInfo` — chart layout metadata (canvas dims, axis ranges, padding)

### Data flow
1. File dropped → `handleFile()` → `parseBinary()` or `parseCSV()` → `sessionData`
2. `processSession()` applies complementary filter gravity correction (gyro + accel fusion, Rodrigues rotation) → `processedData`
3. `renderDashboard()` computes `computeMetrics(processedData)` for the full session, then renders all panels
4. Each split stores its own pre-computed `metrics` object created with `computeMetrics(splitSlice)`

### Binary file format
- v1 (47 bytes/record): `<I H5BH ff H fff fff>` — speed as `uint16 × 0.1 km/h`
- v0 (61 bytes/record): `<I H5BH fff fff fff fff>` — speed as `float32 km/h`
- Version detected from filename suffix (`_v0.bin`, `_v1.bin`)

### Charts
Both charts are drawn on `<canvas>` elements with no charting library.
- **Acceleration chart** — dual Y-axes: left = net linear acceleration (m/s², green), right = speed (km/h, blue). Subsampled to ~800 points for performance (`step = max(1, n/800)`). Supports mouse-wheel zoom (`chartView`) and drag-to-create-split selection.
- **Speed chart** — single-axis, zone background fills, sprint region highlights.

### Splits — regular vs combined
- **Regular**: `isCombined: false`. Created by dragging on the acceleration chart. Boundaries are draggable via handles rendered in `drawSplitOverlays()`.
- **Combined**: `isCombined: true`, has `sourceIds` and `sourceSplits` arrays. Metrics computed via `aggregateMetrics()` (weighted averages for averages, Math.max for peaks, sums for totals). When a child split is edited, all parent combined splits that reference it are recalculated automatically.
- Selecting a combined split highlights **all** child ranges in accent2 blue. Selecting a regular split highlights only its own range.

### Design tokens (CSS variables)
```
--accent:  #00e5a0  (teal/green — regular splits, primary accents)
--accent2: #00b8ff  (blue — combined splits, speed axis)
--accent3: #ff6b6b  (red — impact threshold, warnings)
--warn:    #ffaa00  (orange)
--bg:      #0a0a0f  (page background)
--surface: #12121a  (card background)
```

### Key functions to know
| Function | What it does |
|---|---|
| `computeMetrics(data)` | Computes all 27 metrics from a processedData slice |
| `aggregateMetrics(arr)` | Merges metrics from multiple splits (combined splits) |
| `renderAccelChart()` | Redraws acceleration canvas; call after zoom or data changes |
| `drawSplitOverlays(ctx, ai, override)` | Draws selected split shading + drag handles on accel chart |
| `setupChartSelection()` | Attaches all mouse events for split creation and boundary dragging |
| `selectSplit(id)` | Toggles `selectedSplitId` and re-renders overlay |
| `deleteSplit(id)` | Removes split and cascades removal from any combined parents |
| `renderSplits()` | Rebuilds the entire splits panel from the `splits` array |
| `applyThresholds()` | Re-runs metric computation + re-renders after slider change |

## Development

Open `index.html` via a local HTTP server (e.g. `npx serve .` or VS Code Live Server). There are no tests, no linter, and no build step.

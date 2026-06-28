# SquareCheck — FigUI Plugin Plan

## Overview

A FigUI plugin for probing and testing **squareness** and **straightness** of reference squares and straight edges on a CNC running FluidNC. Uses a touch probe in the spindle to contact workpieces and compute geometric errors.

---

## How It Works (Conceptual)

| Test | What It Measures | How |
|---|---|---|
| **Squareness** | Angle between two adjacent edges of a square reference | Probe each edge at multiple points, fit best-fit lines, compute angle between them, report deviation from 90° |
| **Straightness** | Flatness/straightness of a single edge | Probe the edge at multiple points along its length, fit a reference line, compute max peak-to-valley deviation |

---

## Probing Concept — Side Probing

The probe contacts the **vertical face** of the workpiece (not the top). Z stays fixed at the user-set height throughout the test. The probe moves only in X or Y to touch the side edge.

```
  Top view:
                          ◄── G38.2 probe direction
     ┌──────────────────┐
     │                  │
     │   Square /       │
     │   Straight Edge  │
     │                  │
     └──────────────────┘
          ▲
          probed points along edge
          (probe approaches from outside, moves inward horizontally)
```

The user jogs Z to the desired height first (typically with the probe tip mid-height on the workpiece edge). No Z motion during probing — only X/Y.

## Communication with FluidNC

- **Send G-code**: `call('sendCommand', { command: 'G38.2 X... F...' })` via postMessage → FigUI WebSocket bridge
- **Probe result**: Listen for `fluid-event` `line` events, parse `[PRB:x,y,z:1]` from response
- **Machine status**: Subscribe to `fluid-event` `status` for `wpos` (work position) and `wco` (work offset)
- **Abort**: `call('sendRealtime', { byte: 0x18 })` — sends CAN byte

---

## File Structure

```
plugins/square-check/
  plugin.json    — manifest (name, description, version, layout: "workspace", files list)
  index.html     — HTML + inline JavaScript (no separate JS file, same pattern as centre-probe)
  style.css      — dark theme using FigUI CSS variables (--bg, --accent, --ok, etc.)
  icon.png       — 48x48 icon
```

---

## UI Layout

Split layout (same as centre-probe):
- **Left panel** (46%): parameters, controls, progress steps, results
- **Right panel** (54%): HTML Canvas for visualization of probed points and fitted lines

Two tabs at top (tab-group pattern from centre-probe):
1. **Squareness**
2. **Straightness**

---

## Tab 1 — Squareness

### Mode Selection

A dropdown/segmented control at the top selects the probing mode:

| Mode | Use Case | Probe Position | Probe Direction |
|---|---|---|---|
| **Outside** | Solid square on the bed | Probe is outside the square, facing inward | Moves **toward** the square to contact the outer face |
| **Inside** | Pocket, bore, or hollow square | Probe is inside the square/pocket, facing outward | Moves **away** from center to contact the inner wall |

Both modes probe vertical faces at a fixed Z — the geometry calculation (line fitting, angle) is identical. Only the direction of G38.2 and the retract direction flip.

### Workflow (Outside)

1. User clamps a square reference on the bed
2. User jogs Z to the desired probing height (probe tip mid-height on the square's vertical face)
3. Positions probe near the start of **Edge A** (probe offset horizontally outward from the square's face) → clicks **"Record Edge A Start"**
4. Moves probe along the edge to the end (same Z, same outward offset) → clicks **"Record Edge A End"**
5. Same for **Edge B** (adjacent edge) — record Start and End
6. Sets parameters: points per edge (3–10), probe feed rate, tolerance
7. Clicks **"Start Squareness Test"**

### Workflow (Inside)

1. User has a pocket or hollow square on the bed
2. Jogs Z to the desired probing height
3. Positions probe near the start of **Edge A** (probe offset slightly inward from the pocket wall) → **"Record Edge A Start"**
4. Moves along Edge A to the end (same offset) → **"Record Edge A End"**
5. Same for **Edge B** (adjacent wall) — record Start and End
6. Sets parameters
7. Clicks **"Start Squareness Test"**

### Auto-Probe Sequence (per edge)

Z stays fixed at the user's current Z. For each of N equally-spaced points along the edge:

**Outside mode:**
1. `G0 X{pt.x} Y{pt.y}` (position outward from the edge face)
2. `G38.2 {axis}{target} F{feed}` (probe **inward** toward the square)
3. Wait for `[PRB:...:1]` → record contact
4. `G0 X{retract.x} Y{retract.y}` (retract back outward)

**Inside mode:**
1. `G0 X{pt.x} Y{pt.y}` (position inward from the pocket wall)
2. `G38.2 {axis}{target} F{feed}` (probe **outward** toward the wall)
3. Wait for `[PRB:...:1]` → record contact
4. `G0 X{retract.x} Y{retract.y}` (retract back toward center)

### Calculation

- For each edge's N contact points `(x_i, y_i)`:
  - Fit least-squares line: `y = m·x + b` (or `x = m·y + b` if near-vertical)
- Angle between lines: `θ = arctan(|m₁ − m₂| / |1 + m₁·m₂|)`
- Squareness error: `δ = |θ − 90°|`
- **Pass** if `δ < tolerance`

### Results

- Angle between edges (degrees)
- Deviation from 90° (degrees, with µm/m equivalent)
- Per-edge fitted line info
- Pass / Fail with color indicator

---

## Tab 2 — Straightness

### Workflow

1. User places a straight edge on the bed
2. User jogs Z to the desired probing height (probe tip mid-height on the edge's vertical face)
3. Positions probe at one end of the edge (offset horizontally outward from the face) → clicks **"Record Start"**
4. Moves probe along the edge to the other end (same Z, same outward offset) → clicks **"Record End"**
5. Sets parameters: number of points (5–20), probe feed rate, tolerance
6. Clicks **"Start Straightness Test"**

### Auto-Probe Sequence

Same as squareness — Z is fixed at user's current Z. For each of N equally-spaced points:

1. `G0 X{pt.x} Y{pt.y}` (position outward from the edge)
2. `G38.2 {axis}{target} F{feed}` (probe horizontally toward the edge face)
3. Wait for `[PRB:...:1]` → record contact
4. Retract outward

### Calculation

- For N contact points `(x_i, y_i)`:
  - Fit least-squares reference line
  - For each point, compute perpendicular distance to the line
  - Straightness error = `max(distance) − min(distance)` (peak-to-valley)
- **Pass** if error < tolerance

### Results

- Max deviation (mm)
- RMS deviation (mm)
- Pass / Fail with color indicator

---

## Parameters

| Parameter | Squareness | Straightness |
|---|---|---|
| Points per edge | 3–10 (default 5) | — |
| Total points | — | 5–20 (default 10) |
| Probe feed rate | 50–500 mm/min (default 100) | same |
| Tolerance | 0.001°–1.0° (default 0.05°) | 0.001–1.0 mm (default 0.05 mm) |
| Probe retract distance | 3–20 mm (default 5) | same |

No Z parameters — the user sets Z manually and it stays constant throughout.

---

## Canvas Visualization

- Axis crosshairs through first point
- Probed contact points shown as filled circles with coordinate labels
- Best-fit line(s) drawn in accent color
- For straightness: deviation lines from each point to the reference line (exaggerated scale toggle)
- Dimension annotations (angle value, max deviation value)

---

## Code Pattern

Follow `plugins/centre-probe/` exactly:

- **`doProbe(axis, target, feedRate)`** — sends `G38.2` and waits for `[PRB]` response, returns contact coordinates
- **Progress steps** — `pending → active → done / error` with dot indicators
- **Async/await** — sequential probing steps with `await` between moves
- **Abort** — `state.abort` flag checked after every move; real-time byte sent on abort button
- **Toast** — success / warning / error notifications
- **CSS** — `var(--bg)`, `var(--accent)`, `var(--ok)`, `var(--border)`, etc. for theme integration

## Probing Direction Detection

For each edge, the axis and direction are determined from recorded Start/End positions, edge orientation, and selected mode (inside/outside).

- **Edge direction** = vector from Start to End
- **Probe axis**: perpendicular to the edge face
  - Mostly horizontal edge (`|dy| < |dx|`): probe moves in **Y**
  - Mostly vertical edge (`|dx| < |dy|`): probe moves in **X**
- **Sign (Outside mode)**: probe moves from recorded positions (offset outward) **toward** the square face
- **Sign (Inside mode)**: probe moves from recorded positions (offset inward) **away from center toward** the pocket wall

The plugin shows an arrow diagram on the canvas confirming the probe direction before the user clicks Start.

---

## Repository Structure — Standalone Repo (no fork)

The plugin lives in its **own GitHub repo** (e.g. `FigUI-square-check`), not a fork of FigUI.

**Why:**
- No need to maintain the entire FigUI codebase just for 3 plugin files
- Zero build steps — `plugin.json`, `index.html`, `style.css` are self-contained
- Install directly via FigUI's **Add** button (upload folder) or copy to device `/plugins/`
- Future store listing only requires a PR against FigUI's `plugins/registry.json` — no fork needed
- Anyone can clone and install independently

**Installation methods:**
1. **SD card**: Copy `square-check/` folder to the FluidNC SD card's `/plugins/` directory → FigUI auto-discovers it
2. **Upload**: In FigUI, open the Plugins tab, click **Add**, and upload the folder
3. **Dev testing**: Place `square-check/` in FigUI's local `plugins/` directory and run `npm run dev`

---

## Implementation Order

1. Create `plugin.json` with manifest
2. Create `style.css` with all needed styles (copy/modify from centre-probe)
3. Create `index.html` with:
   - HTML structure (header with tabs, offset-bar, split-layout, parameter panels, canvas)
   - API bridge (`call()`, message listener)
   - State management
   - Tab switching
   - Edge recording (store Start/End positions from current machine position)
   - `doProbe()` primitive
   - Squareness test flow (auto-probe + calculation)
   - Straightness test flow (auto-probe + calculation)
   - Canvas drawing (points, lines, annotations)
   - Abort, Reset functionality
4. Test with `npm run dev` in FigUI

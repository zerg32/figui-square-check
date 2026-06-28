# Square Check

A FigUI plugin for probing squareness, straightness, and probe repeatability on a CNC running FluidNC. Uses a touch probe to contact workpiece faces and compute geometric errors and statistics.

## File Structure

```
plugins/square-check/
  plugin.json        — manifest (name, version, entry, icon, layout)
  index.html         — HTML + inline JavaScript (UI, probing, geometry, diagram)
  style.css          — dark theme using FigUI CSS variables
  icon.png           — 48×48 plugin icon
test-harness.html    — standalone browser test harness with mock FluidNC API
square-check-plugin-plan.md — original design document
```

## Plugin Functions (`plugins/square-check/index.html`)

### API Bridge

| Function | Description |
|---|---|
| `call(method, params)` | Sends a `fluid-request` postMessage to the parent FigUI frame and returns a Promise resolved by the matching `fluid-response`. |
| `window.onmessage` | Handles `fluid-response` (resolve/reject pending calls) and `fluid-event` `status` (updates `state.wco`, `state.toolPos`, detects alarms). |

### State

| Property | Description |
|---|---|
| `state` | Global state object: `wco`, `toolPos`, `probing`, `abort`, `probeReject`, `activeTab`, `probeSide`, `edges` (A/B/S with start/end/dir/orient), `result`, `diagram`. |

### UI

| Function | Description |
|---|---|
| `toast(msg, type)` | Shows a notification at the bottom of the screen. `type` is `'ok'`, `'warn'`, or `'err'`. Auto-hides after 2.8 s. |
| `switchTab(tab)` | Toggles between `'squareness'`, `'straightness'`, and `'repeatability'` panels. Shows/hides the mode bar (squareness only). |
| `onProbeSideChange()` | Updates `state.probeSide` from the dropdown (`'outside'` / `'inside'`). |

### Edge Recording

| Function | Description |
|---|---|
| `recordEdge(edge, which)` | Captures current tool position as start or end for a given edge (`'A'`, `'B'`, or `'S'`). Triggers orientation detection when both points are recorded. |
| `updateEdgeUI(edge)` | Refreshes coordinate display and button states for an edge. |
| `detectEdgeOrientation(edge)` | Determines if edge is `'horizontal'` or `'vertical'` from start/end delta, sets probe direction buttons (+Y/-Y or +X/-X). |
| `setEdgeDir(edge, dir)` | Sets the probe direction for an edge and highlights the active button. |

### Probe Primitive

| Function | Description |
|---|---|
| `doProbe(axis, target, feedRate)` | Sends `G38.2` on the given axis toward `target` at `feedRate`. Listens for `[PRB:...:1]` response. Rejects on 30 s timeout, alarm, or user abort. Returns `{ mx, my, mz, ok }`. |
| `f3(v)` | Shorthand for `v.toFixed(3)`. |

### Steps UI

| Function | Description |
|---|---|
| `initSteps(prefix, steps)` | Creates step progress rows with pending/active/done/error dot indicators. |
| `setStep(prefix, id, status, val)` | Updates a step's dot status and optional value text. |
| `markActiveError(prefix, steps)` | Marks any currently-active step as error (used on abort/failure). |

### Geometry Helpers

| Function | Description |
|---|---|
| `interpolatePoints(p1, p2, n)` | Generates `n` evenly-spaced points along the line from `p1` to `p2`. |
| `fitLine(pts)` | Least-squares line fit. Returns `{ m, b, useX, mx, my }`. Uses `y = mx + b` when the point cloud spans wider in X, `x = my + b` otherwise. |
| `lineY(l, x)` | Evaluates a fitted line at a given x (for `useX: true` lines). |
| `lineX(l, y)` | Evaluates a fitted line at a given y (for `useX: false` lines). |
| `angleBetweenLines(l1, l2)` | Computes the acute angle (in degrees) between two fitted lines. |
| `perpDistance(px, py, l)` | Perpendicular distance from a point to a fitted line. |

### Probe Direction Helpers

| Function | Description |
|---|---|
| `getProbeAxis(orient)` | Returns `'Y'` for horizontal edges, `'X'` for vertical edges. |
| `getProbeTarget(startPos, dist, dir)` | Computes the G38.2 target coordinate by adding `dist` in the probe direction from the starting position. |
| `getRetractPos(startPos, dir, dist)` | Computes a retract position 1.5× `dist` opposite to the probe direction. |
| `safeMoveTo(x, y, safeZ, origZ, lift)` | If `lift` is true and `safeZ` is set, lifts Z to `safeZ`, moves XY, then returns Z to `origZ`. When `lift` is false, only moves XY. |

### Squareness Test

| Function | Description |
|---|---|
| `startSquareness()` | Main async test flow: validates inputs, subscribes to line events, probes Edge A at `nPoints` interpolated positions, probes Edge B, fits lines to both contact sets, computes angle and deviation from 90°, computes per-edge flatness (peak-to-valley deviation from fitted line), displays pass/fail. |
| `resetSquareness()` | Clears edges A/B, hides results and steps, resets UI. |

### Straightness Test

| Function | Description |
|---|---|
| `startStraightness()` | Main async test flow: probes a single edge at `nPoints`, fits a reference line, computes peak-to-valley deviation from perpendicular distances. |
| `resetStraightness()` | Clears edge S, hides results and steps, resets UI. |

### Repeatability Test

| Function | Description |
|---|---|
| `startRepeatability()` | Probes the target position N times (5-50). Computes statistics: mean, standard deviation (σx, σy), range, max spread from mean, and ±3σ bands. |
| `resetRepeatability()` | Clears target position, resets direction, hides results and steps. |
| `recordRPTarget()` | Fills X/Y inputs from current machine position. |
| `setRPDir(dir)` | Sets the active probe direction button (`'+Y'`, `'-Y'`, `'+X'`, `'-X'`). |
| `calcStats(vals)` | Returns `{ n, mean, std, min, max, range }` for an array of numbers. |

### Common Actions

| Function | Description |
|---|---|
| `abortProbe()` | Sets `state.abort`, rejects any in-flight probe, sends CAN byte (`0x18`) via realtime command. |
| `resultLine(label, value, cls)` | Returns HTML for a single result row. |

### Diagram

| Function | Description |
|---|---|
| `drawDiagram()` | Renders the canvas: coordinate axes, recorded edge paths, probed contact points with labels, fitted lines, angle arc with annotation (squareness), deviation lines (straightness), pass/fail indicator. Repeatability mode shows numbered scatter, mean crosshair, and ±3σ ellipse. Auto-scales to fit all points with 40% margin. |
| `cssVar(name, fallback)` | Reads a CSS custom property value, falling back to `fallback`. |

### Init

| Function | Description |
|---|---|
| `init()` | Subscribes to `status` events and draws the initial diagram. |

### ResizeObserver

A `ResizeObserver` on the canvas wrapper triggers `drawDiagram()` whenever the diagram panel changes size (e.g. split pane resizing, results section appearing).

## Safe Z

An optional **Safe Z** parameter in all three test panels. When set to a Z height above the workpiece, the probe lifts to that height before every XY positioning move and returns to the original Z afterward.

This prevents crashes when moving between edges. For example, in squareness mode the probe retracts from Edge A and the first XY move to Edge B would otherwise drag the probe through the corner of the workpiece. Safe Z clears the workpiece before any XY move.

Leave the field empty to skip Z lifting (original behavior).

A checkbox **"Only lift Z between edges"** is enabled by default. When checked, Z only lifts on the first point of each edge; subsequent points on the same edge skip the lift since the XY moves run parallel to the workpiece face. Uncheck it to lift before every XY move (original safe Z behavior).

## Probe Log

Every probe operation is recorded in an internal log for debugging. Log entries include timestamps and cover: test start/end, positioning moves (G0), safe Z lifts, probe commands (G38.2), probe results (`[PRB]`), retract moves, alarms, aborts, and errors.

### Functions

| Function | Description |
|---|---|
| `clearProbeLog()` | Empties the probe log. Called automatically at test start. |
| `getProbeLog()` | Returns a copy of the log array. Accessible on `window` for console inspection. |

### Log Entry Types

| Type | Fields | Description |
|---|---|---|
| `test_start` | `test`, `nPoints`, `feed`, `probeDist`, `tolerance`, `safeZ`, `origZ` | Test begins |
| `test_end` | `pass`, result fields | Test completes |
| `move` | `x`, `y` | G0 XY positioning |
| `safe_z` | `from`, `to` | Z lift up/down |
| `probe` | `cmd`, `axis`, `target`, `feedRate` | G38.2 probe command sent |
| `probe_result` | `mx`, `my`, `mz`, `ok` | `[PRB]` response received |
| `retract` | `axis`, `x`, `y` | Post-probe retract move |
| `alarm` | `state` | Machine alarm detected |
| `abort` | — | User requested abort |
| `error` | `message`, `aborted` | Error caught (with abort flag) |

### Inspection

Open the browser console and call:

```js
getProbeLog()
clearProbeLog()
```

The log can be used to replay or verify the full probe sequence in a unit test.

## Test Harness (`test-harness.html`)

Opens the plugin in an iframe and simulates the FluidNC API for testing in a browser without a real machine.

### Toolbar

| Control | Description |
|---|---|
| X / Y / Z inputs | Mock machine position coordinates. |
| **Send Status** | Sends a `fluid-event` `status` message with current X/Y/Z values and zero WCO. |
| **Send [PRB]** | Sends a `[PRB:...:1]` probe result message using current coordinates. |

### Mock API

| Function | Description |
|---|---|
| `handleMethod(id, method, params)` | Routes `fluid-request` calls: `subscribe`/`unsubscribe` (status and line events), `sendCommand` (logs G-code, auto-responds to G38.2 with probe result after 200 ms, updates mock coordinates for G0/G1), `sendRealtime` (logs byte). |
| `respond(id, result)` | Sends `fluid-response` back to the plugin iframe. |
| `sendToPlugin(type, data)` | Posts a `fluid-event` message to the plugin iframe. |
| `sendMockStatus()` | Reads mock coordinates and dispatches a status event. |
| `sendMockProbe()` | Dispatches a probe result event from current mock coordinates. |
| `sendMockProbeResult()` | Internal — sends `[PRB:...:1]` line event. |
| `init()` | Fetches the plugin HTML and CSS, inlines them into the iframe via `srcdoc`. |

## Testing

Open `test-harness.html` in a browser:

1. The plugin loads in the iframe and receives an initial status after 500 ms.
2. Set mock X/Y coordinates and click **Send Status** to update the machine position.
3. Click **Record Start** / **Record End** for edges (squareness/straightness) or set **Target Position** X/Y (repeatability) — the plugin captures the current mock position.
4. Once start and end are recorded (squareness/straightness) or direction selected (repeatability), probe direction buttons appear.
5. Adjust parameters (points, feed, distance, tolerance). Optionally set **Safe Z** — a Z height above the workpiece to lift to between XY moves (prevents crashing when moving between edges). Leave empty to skip.
6. Click **Start Squareness Test**, **Start Straightness Test**, or **Start Repeatability Test** — the harness auto-responds to each `G38.2` with a probe result after 200 ms, simulating a full probing cycle.
7. Results display in the left panel and on the diagram canvas.

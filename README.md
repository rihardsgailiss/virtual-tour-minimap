# Handoff: Virtual Tour Minimap

## Overview
An interactive minimap for a real-estate virtual tour (360° panorama viewer). A circular minimap overlays the bottom-left of a full-bleed panorama. The user navigates the property by clicking/tapping rooms on the minimap; a location pin travels to the destination while the panorama crossfades to that room's photo. The minimap can be panned, zoomed, and rotates with the viewing direction. Includes a mobile phone-frame variant with a first-run onboarding overlay.

The prototype demonstrates several navigation/hover treatments (modes **A**, **B**, and **Mobile**) via an on-screen switch — see "Modes" below. In production you will implement **one** chosen behavior, not the switcher.

## About the Design Files
The files in this bundle are **design references created in HTML/JS** — a working prototype showing the intended look and behavior. They are **not production code to lift directly**. The task is to **recreate this design in the target codebase's environment** (React, Vue, native, whatever the tour viewer uses) following its established patterns — panorama engine, state management, asset pipeline. If no environment exists yet, pick the framework that best fits the panorama viewer and implement there.

The geometry math (floorplan coordinate transforms, hit-testing, snap-point layout) is the genuinely reusable part — it is framework-agnostic and worth porting closely. The rendering/markup is a reference for the visual result only.

## Fidelity
**High-fidelity (hifi).** Final colors, typography, spacing, iconography, timings, and interactions. Recreate the UI to match, using the codebase's existing component library and panorama viewer. Exact values are in "Design Tokens" below.

## Modes (prototype-only switcher)
A small A / B / M switch (top-left) toggles between three treatments so stakeholders can compare. **Ship one; drop the switch.**
- **A — hover darkens the room.** Hovering a room on the minimap tints its fill (8% `#1A222E`, multiply). All snap points render as static dots.
- **B — hover reveals snap points.** Hovering a room fades in that room's snap-point dots; the nearest point to the cursor is emphasized. On click, the pin drops, then the room's dots fade in briefly.
- **M — Mobile.** Renders the whole tour inside a 393×852 phone frame (46px radius, dark bezel), scaled to fit height. First run shows a circular onboarding overlay over the minimap: an animated tap GIF + "Tap anywhere on the plan to move around the property". Tap dismisses it (fades out, 320ms).

## Screens / Views

### 1. Tour viewer (desktop, full-bleed)
- **Purpose:** Explore the property by looking around the panorama and navigating rooms via the minimap.
- **Layout:** Single full-viewport stage (`100vw × 100vh`, background `#0b0e13`). Absolutely-positioned overlays on top of the panorama layer.
- **Components:**
  - **Panorama layer** — fills the stage. Two stacked `background-image` divs (`background-size: cover`) that ping-pong crossfade on room change (1.1s ease). Drag to look around: horizontal drag adjusts yaw with a damped `tanh` response, soft-capped at ±55° from the drag origin (`SOFT = 260px` drag for near-full swing). Yaw also shifts `background-position-x` by `50 − yaw×0.14`%. A subtle radial vignette sits on top (`radial-gradient(120% 120% at 50% 40%, transparent 55%, rgba(11,14,19,.28) 100%)`).
  - **In-scene hotspots** — circular pins placed at pixel coords over the panorama. 40×40 hit area; 30×30 accent-colored disc with `inset 0 0 0 2px #fff` ring + `0 4px 10px rgba(26,34,46,.4)` shadow; 18×18 white icon mask centered. Click navigates to a room.
  - **Mode switch (top-left, 24/24)** — pill, `rgba(26,34,46,.28)` + `backdrop-filter: blur(6px)`, 8px radius, 3px padding, 3px gap. Three 34×34 6px-radius buttons "A"/"B"/"M", 700 weight 15px. Selected: `#fff` bg / `#1a222e` text. Unselected: transparent / `#fff`.
  - **Top-right controls (24/24, 16px gap)** — four 40×40 8px-radius glass buttons (`rgba(26,34,46,.28)` + blur 6px), each a 30×30 icon: Info (with a 7px `#F2C94C` notification dot top-right), Hotspots-off, Measure tool, Fullscreen.
  - **Bottom toolbar (16/16, 8px gap)** — Map button (40×40 glass, 24×26 icon); Building button (40×40 glass, 30×30 icon); a floor selector group (glass pill): down-arrow 30×30, current floor number "1" (`#fff` 700 18px) + floors icon 22×22, up-arrow 30×30.
  - **Minimap** — see next section.

### 2. Minimap (the core component)
Positioned `left:16px; bottom:80px`, `360×360`.
- **Disc** — circle, `rgba(20,26,36,.34)` + `backdrop-filter: blur(3px)`, shadow `0 10px 30px rgba(11,14,19,.35)`.
- **Floorplan art** — clipped to the circle. The plan lives in a coordinate space of `264.784 × 191.675`, placed via `matrix(0.506,-0.862,0.862,0.506,30.273,245.545)` (an isometric-ish rotation) inside a group that itself gets `translate(panX,panY) scale(zoom) rotate(−yaw)` around origin `180px 180px`. So the map counter-rotates with the view.
  - **Room fills** — 5 rooms, each an SVG `<path>` with a base fill (parchment tones, see tokens). Mode B flattens all fills to `#E5E6E8`. A second overlay path per room provides the hover tint (`rgba(26,34,46,.10)`, `mix-blend-mode: multiply`, opacity 0→1, 0.18s ease).
  - **FloorplanArt** — static detail art (walls, doors, staircase, furniture) drawn as ink paths (`rgb(26,34,46)`) on top of the fills. See `FloorplanArt.jsx`.
- **Pin layer** (clipped to circle, same transform):
  - **Snap-point dots** — `dot.svg`, 16×16, drawn at fixed per-room points. Entrance animation `mm-dotIn` (0.28s `cubic-bezier(.34,1.56,.64,1)`), staggered delay. Counter-scaled by `1/zoom` and de-rotated so dots stay upright and constant-size.
  - **Destination preview pin** (during travel) — `Pin-location.svg` 24×30 with a pulsing ring behind (`mm-ring` 1.1s ease-out infinite, `rgba(26,34,46,.28)`). Pin entrance `mm-pinIn` 0.4s. Rotates to `2×yaw`.
  - **Current-location pin** — `Pin-location.svg` 24×30, transitions `left`/`top` over 0.9s `cubic-bezier(.4,0,.2,1)` when moving. A white label chip beside it (de-rotated to stay level): `#fff` bg, `#1a222e` text, 6px radius, 4px 10px padding, 700 14px, showing `{room name} {area}` e.g. "Living room 12.05 m²".
- **Interaction surface** — a transparent circle on top (`z-index:5`, `touch-action:none`, `cursor:pointer`):
  - **Click/tap (no drag):** move to the nearest snap point of the clicked room. Snap logic and mode-A preview always agree (shared `closestSnap`).
  - **One-finger / mouse drag:** pan the map (clamped so it can't leave the frame: `±(zoom−1)×180`). A drag > 3px is treated as pan, not a tap.
  - **Two-finger pinch:** zoom `1 → 3.5`, panning by the pinch midpoint.
  - **Wheel:** zoom by ×1.15 / ÷1.15 per notch, clamped `1 → 3.5`.
  - **Hover (mouse):** drives mode A/B highlight behavior.

## Interactions & Behavior

### Navigation (two styles, prop `navStyle`)
- **`fade-then-travel` (default):** set `pending` destination + `navigating=true`; after 720ms swap `cur` to the destination and crossfade the panorama; after a further 900ms clear `navigating`. During navigation, further clicks are ignored.
- **`snap`:** immediately set `cur` and crossfade — no travel delay.
- Yaw (facing) is **never** reset on location change — the pin keeps its heading.

### Panorama crossfade
Ping-pong: put the new room's image on the hidden layer (raised above), fade it in over the still-opaque outgoing layer, then flip which layer is "front". Guarantees the dark background never flashes. All four room images are preloaded on mount.

### Coordinate transforms (port these carefully)
- `screenToFloor(sx, sy)` — converts a point in the 360×360 minimap box back to floorplan coords by undoing pan → zoom → view-rotation → the base matrix. Used for hit-testing clicks.
- `roomAt(f)` — canvas `isPointInPath(Path2D(room.path), …)` hit-test to find which room polygon contains a floorplan point.
- `closestSnap(f)` — nearest defined snap point in the hit room (or nearest room if outside all polygons).
- Snap points are generated per room (`snapPoints()`), filtered to points that actually fall inside the room polygon, then clustered so the tightest neighbors overlap by exactly 30% of the 16px dot (`_overlapClusters`). Small rooms (bath) get a single centered point.

### Animations (keyframes)
- `mm-pinIn` — pin entrance: `opacity 0→1`, `translateY(6px) scale(.7) → 0/1`.
- `mm-ring` — pulsing ring: `scale(.55) opacity .55 → scale(2.3) opacity 0`, 1.1s ease-out infinite.
- `mm-dotIn` — snap dot pop-in: `scale(.2) opacity 0 → scale(1) opacity 1`, 0.28s spring.
- `mm-fade` / `mm-fade2` — panorama layer fade-in (alternating so the animation restarts on each swap), 1.1s ease.
- `mm-overlayOut` — mobile onboarding dismiss: `opacity 1 scale 1 → opacity 0 scale 1.06`, 0.32s.
- Pin-travel is a CSS transition on `left`/`top` (0.9s), not a keyframe.

## State Management
Key state (per the prototype's logic class):
- `cur` — current location `{x, y, name, area, pano, img}`.
- `pending` — in-flight destination during travel, else `null`.
- `navigating` — boolean lock while a travel is in progress.
- `yaw` — viewing direction in degrees; drives map rotation, pin cone rotation, and panorama parallax.
- `mmZoom` (1–3.5), `mmPanX`, `mmPanY` — minimap transform.
- `layers` (2), `front` (0|1), `fadeN` — panorama crossfade bookkeeping.
- `hoveredRoom`, `hoverDots`, `dotsOpacity` — hover highlight state.
- `mode` ('A'|'B'|'M') — prototype switcher only; drop in production.
- `tutorialSeen`, `tutorialClosing` — mobile onboarding overlay.

Data fetching: in production the room list, floorplan geometry, snap points, and panorama image URLs come from the tour dataset/API. Here they are hardcoded in `waypoints()` and `roomDefs()`.

## Design Tokens
Colors:
- Stage / letterbox background: `#0b0e13`
- Stage inner (behind panorama): `#1a222e`
- Ink (floorplan art, text on light): `rgb(26,34,46)` / `#1a222e`
- Accent (pins, hotspots) — prop, default `#1CB0F6`; options `#1CB0F6`, `#2A6FDB` (blue), `#1F8A5B` (green), `#E9615A` (red)
- Link: `#1CB0F6`, hover `#55c6f8`
- Notification dot: `#F2C94C`
- Glass control bg: `rgba(26,34,46,.28)` + `backdrop-filter: blur(6px)`
- Minimap disc: `rgba(20,26,36,.34)` + `blur(3px)`
- Hover tint: `rgba(26,34,46,.10)`, `mix-blend-mode: multiply`
- Room base fills: bedroom `rgb(249,231,219)`, living `rgb(236,233,214)`, kitchen `rgb(255,247,236)`, bath `rgb(221,228,240)`, hall `rgb(255,247,236)`
- Mode-B flat fill: `#E5E6E8`
- Label chip: bg `#fff`, text `#1a222e`

Typography:
- Family: `'Gilroy'` with system fallback stack (`-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif`)
- Floor number: 700 / 18px. Mode buttons: 700 / 15px. Label chip: 700 / 14px. Onboarding copy: 700 / 16px / 24px line-height.

Radii: glass buttons 8px, small buttons/switch keys 6px, label chip 6px, phone frame 46px.
Shadows: minimap disc `0 10px 30px rgba(11,14,19,.35)`; hotspot `0 4px 10px rgba(26,34,46,.4)`; phone frame `0 0 0 4px #0b0e13, 0 0 0 7px #2b3038, 0 30px 80px rgba(0,0,0,.6)`.

Timings: crossfade 1.1s ease; pin travel 0.9s `cubic-bezier(.4,0,.2,1)`; hover tint 0.18s; dot pop-in 0.28s spring; travel delay 720ms then 900ms unlock; onboarding dismiss 320ms.

Minimap sizes: box 360×360; disc fills box; dots 16px; pin 24×30; floorplan coordinate space 264.784×191.675; base matrix `matrix(0.506,-0.862,0.862,0.506,30.273,245.545)`; transform origin `180px 180px`.

## Assets
In `assets/`, `uploads/`, and `FloorplanArt.jsx` in this bundle:
- Panoramas: `assets/pano-living.png`, `pano-kitchen.png`, `pano-hall.png`, `pano-bedroom.png` (room 360° stills).
- Toolbar / control icons (SVG): `ic-info`, `ic-hotspot-off`, `ic-measure`, `ic-fullscreen`, `ic-map`, `ic-building`, `ic-down`, `ic-up`, `ic-floors`.
- In-scene hotspot icons: `assets/door_open.svg`, `assets/stairs_2.svg`.
- `assets/dot.svg` — snap-point dot. `uploads/Pin-location.svg` — location pin.
- `uploads/ced7565297fd102040c824b049bfe3d410afba13.gif` — mobile onboarding tap animation.
- `FloorplanArt.jsx` — static floorplan detail art (walls/doors/staircase/furniture), extracted from Figma. Coordinate space 264.784×191.675.

These came from the project's Figma source (icons/floorplan) and supplied room photography. In production, use the codebase's own icon set where equivalents exist; the panoramas and floorplan geometry are content data.

## Files
- `Virtual Tour Minimap.dc.html` — the full prototype (markup + logic). The logic class holds all geometry, navigation, and interaction code — the primary reference. (Runs in the design tool's component runtime via `support.js`; treat the `<x-dc>`/`{{ }}` template as JSX-like markup and the `class Component` as a React-style component.)
- `FloorplanArt.jsx` — static floorplan art component.
- `support.js` — the runtime that powers the `.dc.html` file (included only so the prototype opens in a browser for reference; not needed in production).

To view the prototype: open `Virtual Tour Minimap.dc.html` in a browser (it self-loads `support.js`).

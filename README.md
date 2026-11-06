# Berthwise — Container Port Terminal

> **World 03** of the World Prompt Pack · Teal `#0FA3A3` · Responsive (mobile + large screen)

🌐 **Live:** https://dbillion.github.io/berthwise/

---

## What It Is

Berthwise is a live 3D container port operations dashboard for **Constellation Port**. Built entirely in procedural Three.js geometry — no external model files. Everything moves autonomously on a continuous simulation loop, with a frosted-glass mission-control UI layered over the 3D scene.

---

## How the Polygons Were Exported Into This HTML File

Every object is **generated from raw Three.js primitives inside `index.html`** — zero `.gltf`, `.obj`, or texture files are loaded. The entire scene is defined as JavaScript geometry calls.

### Primitive palette

| Primitive | Three.js class | Used for |
|-----------|----------------|----------|
| Box | `BoxGeometry(w,h,d)` | Ship hulls, containers, crane beams, warehouses, gate booths, wagons |
| Cylinder | `CylinderGeometry(rt,rb,h,seg)` | Crane legs (rounded), lighthouse, funnel stacks, tug funnels, wheels |
| Cone | `ConeGeometry(r,h,seg)` | Ship bow (wedge), tug bow |
| Plane | `PlaneGeometry(w,h,wS,hS)` | Water surface (Gerstner animated), ground, quay apron |
| Torus | `TorusGeometry` | — |
| Points | `BufferGeometry + PointsMaterial` | Stars |

### Container colour system

```js
const CONT_COLORS = [0xe63946, 0x2a9d8f, 0xe9c46a, 0x264653,
                     0xf4a261, 0x0FA3A3, 0x6a4c93, 0xffffff, 0x118ab2];
const CMAT = CONT_COLORS.map(c => new THREE.MeshLambertMaterial({color:c}));
```

Each container on ship decks, in the yard, and on rail wagons randomly picks from this pre-built material pool — no per-frame allocation.

### Ship construction technique

```
BoxGeometry(60,6,16)        → hull body
CylinderGeometry(0,8,6,4)   → bow wedge (zero-top radius = pointed)
BoxGeometry(10,8,14)         → superstructure
BoxGeometry(8,4,10)          → bridge (glass material, transparent)
CylinderGeometry(1.2,1.5,4,8)→ funnel
BoxGeometry(3,2,2) × 96      → container stacks (4 rows × 8 cols × 3 high)
```

All parts added to a `THREE.Group` — the whole ship translates and rotates as one unit.

### STS Crane (Ship-to-Shore Gantry) construction

```
BoxGeometry(1.2,24,1.2) × 4   → 4 rail legs at corners
BoxGeometry(50,1.5,2)          → main girder (extends 25m over water)
BoxGeometry(10,5,8)            → machine room on portal
BoxGeometry(3,1.5,3)           → trolley (animated along girder)
BoxGeometry(3.5,0.4,2.2)       → spreader (child of trolley, hangs below)
CylinderGeometry(0.1,0.1,4,4) → rope (child of trolley)
```

The trolley's `position.x` is animated each frame by the crane state machine.

### Water animation (Gerstner-inspired)

```js
// Pre-cached original vertex positions
const waterOrig = waterPos.array.slice();

// In render loop (every 2nd frame):
waterPos.setY(i,
  Math.sin(ox * 0.25 + t) * 0.5 +
  Math.cos(oz * 0.18 + t * 1.4) * 0.4
);
```

Storm weather amplifies wave height to `×1.4` multiplier without any extra geometry.

---

## How the Autonomous Simulation Works

### STS Crane cycle (loops forever per crane)

```
toship → lower → pickup → raise → toyard → setdown → toship → …
```

- Trolley moves at 8 units/sec in THREE.js space
- On `pickup`: a coloured container mesh is spawned and parented to the trolley position
- On `setdown`: container is removed from scene (handed to truck)
- Each completed cycle: `totalMoves++`, KPI updates

### Terminal Truck routes

Each truck follows a pre-defined waypoint array:

```js
const route = [{x:-25,z:-44}, {x:-25,z:0}, {x:-25,z:28}, {x:-25,z:0}];
```

Frame by frame: `position += (target - position).normalized × speed × dt`  
No overlap — routes are intentionally non-intersecting.

### RTG Crane (rubber-tyred gantry)

Trolley oscillates ±7 units on X axis while the RTG itself drifts slowly via `Math.sin()`.

### Container train

Translates at constant `+2 units/sec` on X, wraps at ±80 — always moving across the rail siding.

---

## Responsive Design

| Screen | Layout |
|--------|--------|
| **≤ 640px (mobile)** | KPI horizontal scroll strip · Bottom sheet (5 tabs: Vessel Tracker, Live Telemetry, Berths, Equipment, Containers) · 5-button bottom nav · Desktop panels hidden |
| **641px – 1399px (tablet/laptop)** | Standard 3-panel: KPI top-left, detail top-right, table bottom-right, tracker bottom-left |
| **≥ 1400px (large desktop)** | Wider panels, larger fonts, more padding |
| **≥ 1800px (ultra-wide)** | Extra stats panel revealed on left side |

All tap targets min 44px height. `touch-action:none` on canvas for clean OrbitControls pinch-zoom.

---

## Weather System

| Weather | Effect |
|---------|--------|
| ☀️ Clear | Full sun, no fog, calm water |
| ⛈️ Monsoon | FogExp2, rain 2D canvas overlay, lightning flash |
| 🌫️ Fog | Dense Fog(30, 110), reduced sun/ambient |
| 🌊 Storm | FogExp2, rough water (1.4× wave height), crane wind hold alerts, rain |

---

## Controls

| Input | Action |
|-------|--------|
| Left-drag | Pan |
| Right-drag | Orbit / rotate |
| Scroll / pinch | Zoom |
| Click object | Select → glass detail panel + camera glide |
| Area switcher / bottom nav | Fly to Berths / Yard / Gate / Rail |
| Search + Enter | Keyword area fly |
| ☀️/⛈️/🌫️/🌊 | Weather |
| 🌙 / ☀️ | Day / Night toggle |

---

## Stack

- **Three.js r128** — local `js/three.min.js` (CDN fallback)
- **OrbitControls r128** — local `js/OrbitControls.js`
- **Vanilla CSS** — no Tailwind, no frameworks
- **Single `index.html`** (~62KB) — zero build step

---

## Performance

| Issue | Guard-rail |
|-------|-----------|
| `new THREE.Vector3()` in loop | Pre-allocated `_v3`, `_v3b` scratch objects |
| DOM reflow every frame | 500ms `domTimer` throttle |
| Water subdivision perf | Updated every 2nd frame (`frameCount%2===0`) |
| Rain on WebGL canvas | Drawn on separate 2D `<canvas>` overlay |
| Shadow overdraw | 1024×1024 max, one `DirectionalLight` |
| Spawned container GC | Removed immediately on `setdown`, not pooled (low rate) |

---

*Part of the [World Prompt Pack](https://x.com/sanjai.builds) — 17 interactive 3D strategy-game dashboards.*

| World | Name | URL |
|-------|------|-----|
| 01 | Gateflow — Airport | https://dbillion.github.io/gateflow/ |
| 02 | Orbitline — Spaceport | https://dbillion.github.io/orbitline/ |
| 03 | **Berthwise — Container Port** | https://dbillion.github.io/berthwise/ |

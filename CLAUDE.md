# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A cinematic scrollytelling site for the Millennium Falcon. Two files:
- `millennium-falcon.html` — the complete single-file app (~1,180 lines)
- `millennium_falcon.glb` — 24 MB real 3D model (10 meshes, 8 materials)

## Running

The GLB cannot be loaded via `file://` (browser security). A local HTTP server is required:

```bash
cd "/Users/veronika_zavgorodnia/Documents/AI 3D"
python3 -m http.server 3333
# then open http://localhost:3333/millennium-falcon.html
```

The `.claude/settings.local.json` allows `Bash(open *)`.

## External dependencies (CDN only)

| Library | CDN | Version |
|---|---|---|
| Three.js | cdnjs.cloudflare.com | r128 (global `THREE`) |
| GLTFLoader | cdn.jsdelivr.net/npm/three@0.128.0 | 0.128.0 |
| GSAP | cdnjs.cloudflare.com | 3.12.2 |
| ScrollTrigger | cdnjs.cloudflare.com | 3.12.2 (registered but only used for camera tweens — not for the main scroll pipeline) |
| Orbitron + IBM Plex Mono | fonts.googleapis.com | — |

No ES modules, no build step. All library globals (`THREE`, `gsap`) must come from CDN.

## Architecture

The `<script>` block is split into numbered sections §1–§14. Execution order matters; later sections depend on globals from earlier ones.

---

### GLB model coordinate system

The GLB node matrices rotate the model so its axes in scene space are:

| Axis | Direction | Range |
|---|---|---|
| Z | length — aft(−) → fwd(+) | ±5 units |
| X | width — port(−) → starboard(+) | ±4 units |
| Y | height — bottom(−) → top(+) | ±1.2 units |

This matters for annotation pin placement, camera keyframe positions, and engine light placement. The longest dimension is Z, not Y.

---

### Scroll → scene pipeline

**No GSAP ScrollTrigger for scroll tracking.** A native `window.addEventListener('scroll', handleScroll, { passive: true })` writes to the module-level `scrollProgress` (0–1). The animation loop reads it once per frame — this prevents multi-step opacity jumps when a trackpad fires many events per frame.

```
scroll event → scrollProgress (float 0–1)
                    ↓ (animation loop, once per frame)
             applyModelPhase(scrollProgress)   — opacity curves
             getCameraForScroll(scrollProgress) — camera position
```

`handleScroll` also detects the active phase (0–5) by threshold comparisons and updates the pill label, phase dots, nav items, and caption panel visibility via `animateCaptions(ph)`.

---

### Phase reveal — `applyModelPhase(p)`

Two opacity curves drive the building sequence:

| Curve | Range | Effect |
|---|---|---|
| `edgeOp` | fade-in [0→0.18], fade-out [0.32→0.52] | Blueprint skeleton (blue `EdgesGeometry` lines) |
| `solidOp` | fade-in [0.30→0.92] | Real solid GLB materials |

The cross-fade (skeleton out / hull in) happens during Phase 2 (p ≈ 0.30–0.52).

**EdgesGeometry is built asynchronously** — one mesh per `setTimeout(fn, 0)` tick — so heavy geometry computation never blocks the main thread. The `edgeLines` array grows over the first ~200 ms after load; `applyModelPhase` handles an empty array gracefully.

All GLB mesh materials start at `opacity: 0`. `solidMeshes` is an array of `{ mesh, mats[] }` populated at load time. Opacity lerps toward `solidOp` at factor 0.12 per frame.

---

### Camera system

**Scroll-scrubbed, no GSAP tweens.** Seven keyframes at phase boundaries (`CAM_KEYFRAMES`, `CAM_STOPS`). Every animation frame:

1. `getCameraForScroll(scrollProgress)` interpolates between the two surrounding keyframes using smoothstep.
2. `camTarget` lerps toward the goal at factor 0.14 (cinematic lag).
3. `camera.position` lerps toward `camTarget + mouseParallax` at factor 0.12.

To change a camera angle: edit the matching entry in `CAM_KEYFRAMES`. The stops are `[0, 0.12, 0.30, 0.48, 0.65, 0.82, 1.0]` — one per phase boundary plus a hold at 1.0. Never call `gsap.to(camTarget)` for scroll-driven movement; that breaks the scrub.

---

### Annotation pins

Each pin is an HTML `div.pin` inside `#ann` (fixed overlay). The authoritative position is a `THREE.Object3D` tracker parented to `falconGroup`.

**Positions are set at load time by raycasting** (`requestAnimationFrame` after `falconGroup.add(model)`): each pin fires a ray from outside the bounding box toward its feature zone; `hits[0].point` gives the exact surface coordinate; `falconGroup.worldToLocal(pt)` converts to local space. The `PINS` array holds fallback positions used only when a ray misses.

Every frame after `renderer.render()`, `updateAnnotations(activePhase)` calls `tracker.getWorldPosition(tmpVec)` (which forces `updateWorldMatrix`) then `tmpVec.project(camera)` to update `el.style.left/top`.

**Flip direction**: `flip: false` → label extends right (dot is left edge of the div); `flip: true` → `flex-direction: row-reverse` + CSS `transform: translate(-100%, -50%)` so the dot (rightmost element) lands on the projected point.

---

### Caption panels

Each section (`min-height: 150vh`, `align-items: flex-end`) positions its `.cap-wrap` at the bottom of the section. `position: sticky; top: calc(50vh - 170px)` catches the card at mid-screen as it scrolls upward, holding it there through most of the section, then releasing it off-screen top.

`animateCaptions(phase)` is called whenever `activePhase` changes — it adds/removes `.vis` on `#cap1`–`#cap5`. But visibility is also naturally governed by sticky: the card is only in the viewport while its section is.

---

### Cinematic overlays

- **Vignette** — CSS `radial-gradient` on `#vignette` (z-index 4)
- **Scanlines** — CSS `repeating-linear-gradient` on `#scanlines` (z-index 3)
- **Film grain** — 192×192 `<canvas>`, updated every ~75 ms, `mix-blend-mode: screen`, opacity 0.028
- **Canvas filter** — `filter: contrast(1.07) saturate(1.12)` on `#c`
- No engine glow sprites; engine point light (`engineLight`) and reactor light scale with `solidOp`

---

### Audio

Web Audio API only — no audio files. Three oscillators → LFO → lowpass → `masterGain`. `AUDIO.toggle()` ramps gain between 0 and 0.3. `AudioContext` created lazily on first click to satisfy autoplay policy.

---

## Design tokens

```
--bg: #07080A       background void
--surface: #0F1219  UI panels
--border: #1C2230   structural lines
--accent: #B3121F   CEC rebellion red
--text: #DDE2EE     primary text
--text2: #7888A4    secondary
--text3: #3E4A5E    muted / labels
```

Fonts: **Orbitron** (headings) and **IBM Plex Mono** (body, data, labels).

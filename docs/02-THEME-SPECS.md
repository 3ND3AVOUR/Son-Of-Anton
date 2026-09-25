# Theme Specifications

Five complete sensory skins. Each spec defines: palette, typography, geometry, ambient
background, cursor, hover behaviour, click effect, sounds, and the full-screen transition it
plays when you switch *away* from it.

All visual assets are **original SVG/CSS** — see `01-RESEARCH.md` §2 for why.

---

## 1 · `mark-vii` — Iron Man

> **Precision.** Everything is a readout. Nothing is decorative; it's all telemetry.

### Palette

| Token | Value | Use |
|---|---|---|
| `--bg` | `#0A0E14` | Deep carbon |
| `--bg-elev` | `#111823` | Panels |
| `--fg` | `#E7F6FF` | Body |
| `--fg-muted` | `#7C93A6` | Labels, telemetry |
| `--accent` | `#4FE3FF` | Arc-reactor cyan — HUD lines, cursor |
| `--accent-2` | `#FFB800` | Hot gold — CTAs, highlights |
| `--accent-3` | `#C41E3A` | Hotrod red — sparingly, for alerts |
| `--stroke` | `rgba(79,227,255,.15)` | Hairline grid |

Contrast: `#E7F6FF` on `#0A0E14` ≈ 15:1. Safe. Keep `--fg-muted` off small text.

### Typography

- Display: **Chakra Petch** or Rajdhani — wide, technical, slightly angular
- Body: **Inter** or IBM Plex Sans
- Mono: **Share Tech Mono** / JetBrains Mono — every number on screen is mono

`--radius: 2px` · `--border-w: 1px` · `--ease-signature: cubic-bezier(.16,1,.3,1)` (expo-out:
fast start, hard settle — mechanical) · `--dur-base: 220ms`

### Geometry & motifs

Hexagonal grid · concentric targeting reticles · 1px cyan hairlines · **corner brackets on
every card** (four L-shapes, not a full border) · scan lines · radar sweep · percentage
readouts next to everything · `[ ]` bracket decorations on headings.

The corner-bracket card is the signature component. Four absolutely-positioned 12px L-shapes
that animate outward 4px on hover.

### Ambient background

- A slow-rotating targeting reticle behind the hero (SVG, 60 s per revolution)
- Hex grid with 3-layer mouse parallax (0.02 / 0.05 / 0.09 translation factors)
- Arc-reactor breathing pulse: 2 s `ease-in-out` opacity 0.6 ↔ 1 on a radial gradient
- A diagnostic text ticker in the footer cycling fake telemetry
- Occasional single scanline sweeping top→bottom, every ~12 s

### Cursor

Two-part. Inner **4px cyan dot**, 0.08 s follow. Outer **28px hexagonal reticle** with corner
ticks, 0.45 s follow (the lag is the effect), rotating continuously at 8 s/turn.

- **Hover:** reticle expands to 56px, rotation stops and snaps to 0°, corner ticks extend, a
  mono label appears beside it — `ANALYZE` / `LINK` / `DEPLOY` per element type
- **Magnetic:** on `[data-magnetic]`, the reticle locks to the element's centre and the element
  itself translates 25% toward the pointer
- **Down:** reticle contracts to 20px, dot brightens to white

### Click — Repulsor blast

Six layers, 500 ms total:

1. **Charge trace** (0–60 ms) — a 2px cyan line from a fixed "palm" anchor (bottom-right) to the click point, drawn via `stroke-dashoffset`, then instantly gone
2. **Core flash** (0–180 ms) — radial-gradient div, `mix-blend-mode: screen`, `blur(6px)`, scale 0→3, white→cyan
3. **Primary ring** (0–420 ms) — 2px cyan circle, scale 0→22, opacity 1→0, `expo.out`
4. **Echo ring** (80–500 ms) — 1px, thinner, scale 0→30, 40% opacity
5. **Sparks** (0–600 ms) — 10–14 canvas particles, radial ejection, drag 0.92, additive blending, gold→cyan gradient over life
6. **Bloom lift** (0–120 ms) — brief global brightness +6% (rate-limited, see a11y)

### Hover

Corner brackets push outward 4px. Text runs a 220 ms **character-scramble decrypt** (GSAP
SplitText + random glyph substitution settling left to right). A 1px cyan underline draws from
left.

### Sound

| Event | Sound |
|---|---|
| Hover | 40 ms soft tick, ~2 kHz, very quiet |
| Click | *Synthesized:* sawtooth 180→900 Hz over 120 ms into a lowpass, then a filtered-noise burst + 60 Hz sine thump |
| Theme enter | HUD boot sequence — rising sweep with stepped ticks |
| Music | Dark cinematic synth loop, ~90 BPM, low pulse |

### Transition out

**HUD shutdown.** Grid lines retract to the edges, panels collapse to 1px outlines, a cyan
scanline closes vertically like an eye, then the new theme boots in.

---

## 2 · `web-head` — Spider-Man

> **Comic book print.** Ben-Day dots, thick ink, kinetic panels. Into-the-Spider-Verse energy.

### Palette

| Token | Value | Use |
|---|---|---|
| `--bg` | `#F5F2EC` | Newsprint white |
| `--bg-elev` | `#FFFFFF` | Panels |
| `--fg` | `#111111` | Ink black |
| `--accent` | `#E62429` | Spidey red |
| `--accent-2` | `#2B3A8C` | Web-head blue |
| `--accent-3` | `#FFD400` | SFX pop yellow |
| `--stroke-strong` | `#111111` | 3px panel borders |

This is the one **light** high-contrast theme. It's the visual shock in the switcher.
Contrast is trivially safe: black on newsprint.

### Typography

- Display: **Anton** or Archivo Black — heavy, condensed, all-caps headings
- SFX/pops: **Bangers** or Luckiest Guy
- Body: **Inter** at a slightly heavier weight than usual (500)

`--radius: 0` · `--border-w: 3px` · `--ease-signature: cubic-bezier(.34,1.3,.64,1)` (snappy,
slight overshoot) · `--dur-base: 180ms`

### Geometry & motifs

Halftone dot overlay (CSS `radial-gradient` tile, or an SVG pattern) · **thick 3px black
borders** on every card, drawn as comic panel gutters · cards rotated ±1.5° · speech-bubble
tooltips with a tail · hard offset shadows (`6px 6px 0 #111`, no blur) · web-pattern corner
ornaments · chromatic aberration on hover (red/cyan `text-shadow` offset by ±2px).

### Ambient background

- Halftone dots at 6% opacity over everything, scaled to viewport
- Thin web strands anchored to the four corners, swaying with a mouse-driven sine
- Sections **snap** in on scroll: scale 0.96→1 with a 1.06 overshoot, plus a 2° rotation settle
- Occasional floating "SFX" words drifting in the far background at low opacity

### Cursor

A **web-shooter crosshair**: a 24px circle with an internal web crosshair (4 spokes + 1 arc).
A fading **silk trail** follows the path — a canvas polyline of the last ~20 positions, tapering
in width, ~180 ms decay.

- **Hover:** morphs to a spider emblem (`MorphSVG` between two paths)
- **Down:** crosshair contracts, trail thickens

### Click — THWIP!

1. **Web line** (0–140 ms) — an SVG quadratic bezier from a wrist anchor (bottom-left) to the click point, with the control point offset perpendicular to the chord for a natural sag. Drawn via `stroke-dasharray` / `stroke-dashoffset`, `power2.in`
2. **Web splat** (140–800 ms) — a pre-authored SVG `<symbol>` (8 radial spokes + 4 concentric catenary arcs), random rotation 0–360°, scale 0→1 on `back.out(2.2)`, then fade
3. **"THWIP!"** (140–600 ms) — Bangers text pop at a random ±12° tilt, scale 0→1.2→1, drifting up 20px while fading. Rotate the word list: `THWIP` · `SNAP` · `BAM` · `KRAK`
4. **Line recoil** (200–320 ms) — the web line snaps back toward the anchor and vanishes
5. **Halftone burst** — a short radial ring of dots expanding from impact

### Hover

Card tilts 2°, hard shadow offsets to `10px 10px 0`, text chromatically splits red/cyan,
and a tiny halftone burst fires at the pointer entry point.

### Sound

| Event | Sound |
|---|---|
| Hover | Paper rustle, 50 ms |
| Click | *Sample:* short filtered-noise swoosh with a fast downward pitch bend + a click transient |
| Theme enter | Comic page-flip whoosh |
| Music | Upbeat funk/hip-hop loop, brass hits, ~105 BPM |

### Transition out

**Panel wipe.** Four vertical comic panels sweep across the viewport at staggered speeds with
thick black gutters between them, covering the screen; tokens flip; panels sweep off the
opposite side.

---

## 3 · `ki-surge` — Dragon Ball Z

> **Raw energy.** Aura, lightning, impact frames. The loudest theme.

### Palette

| Token | Value | Use |
|---|---|---|
| `--bg` | `#0B0B10` | Deep space |
| `--bg-elev` | `#15151F` | Panels |
| `--fg` | `#FFF6E0` | Warm white |
| `--accent` | `#FFD54A` | Super-saiyan gold |
| `--accent-2` | `#29B6F6` | Ki blue |
| `--accent-3` | `#FF6D00` | Energy orange |
| `--glow` | `rgba(255,213,74,.55)` | Aura bloom |

### Typography

- Display: **Teko** or Saira Condensed, heavy, with a `skewX(-6deg)` applied for that
  forward-leaning manga urgency
- Numbers: oversized, tabular — "POWER LEVEL" readouts are a design element
- Body: **Inter**

`--radius: 4px` · `--border-w: 2px` · `--ease-signature: cubic-bezier(.22,1.2,.36,1)`
(explosive out with overshoot) · `--dur-base: 260ms`

### Geometry & motifs

Radiating **manga speed lines** behind headings · aura flames (layered box-shadow flicker or
an SVG turbulence displacement outline) · lightning crackle paths · starfield · impact frames
(one-frame white invert + black radial burst) · angular clipped panel corners
(`clip-path: polygon(...)`) · big tabular power-level numbers.

### Ambient background

- **Rising aura particles**: ~40 canvas particles from the bottom edge, upward with a sine
  wobble, additive blending, gold→orange, halved on mobile
- Starfield with slow parallax drift
- A lightning crackle every ~15 s across the hero — an SVG polyline with jagged random
  midpoints, drawn on in 80 ms, flashed, gone. **Rate-limited**
- Headings arrive with a burst of speed lines drawn outward from the text
- Section reveal triggers a 4px screen shake

### Cursor

A **ki orb**: a 14px glowing sphere with an additive halo and a 6-particle tail.
Charges while held — grows to 34px, brightens toward white, and pulls the ambient aura
particles inward toward itself.

### Click — Ki blast (charge-and-release)

This is the only theme with a **hold mechanic**, and it's the most fun interaction on the site.

**On `pointerdown`:**
- Orb radius grows on a curve, clamped at 1.2 s
- Aura particles converge on the cursor
- A synthesized sine rises in pitch in real time — *the pitch is the charge meter*
- A thin charge ring tightens around the orb

**On `pointerup`,** all magnitudes scale with charge `t ∈ [0,1]`:

1. **Core** — white sphere, scale 0→`4+8t`, 200 ms
2. **Energy ring** — gold outer / blue inner, scale 0→`15+20t`, 500 ms, `expo.out`
3. **Speed lines** — `12+20t` SVG lines radiating out, random length/angle/width, drawn via dash-offset over 220 ms, staggered 6 ms
4. **Screen shake** — damped random walk, amplitude `3+6t` px, 180 ms, applied to a **wrapper div, never `body`** (shaking body breaks scrollbars and can trigger layout)
5. **Impact frame** — 33 ms full-screen white at `mix-blend-mode: difference`. **Rate-limited to 1 per 400 ms globally** (WCAG 2.3.1)
6. **Rising sparks** — `20+40t` particles, upward bias, gravity, additive

### Hover

A flickering gold aura outline — three stacked `box-shadow` layers at different blurs, each
with an independent ~80 ms opacity flicker, so the edge never sits still. Element lifts 3px.

### Sound

| Event | Sound |
|---|---|
| Hover | Low energy hum swell, 80 ms |
| Charge | *Synthesized:* sine, frequency tracking charge time 120→800 Hz + noise bed |
| Release | Deep 45 Hz thump + downward noise sweep, volume scaled by charge |
| Theme enter | Power-up surge |
| Music | Driving rock/orchestral hybrid, heavy percussion, ~140 BPM |

### Transition out

**Instant transmission.** Gold aura floods from the centre, the screen goes white, a horizontal
scanline splits it into two halves that slide apart, revealing the next theme.

---

## 4 · `grand-line` — One Piece

> **Adventure and paper.** Warm, hand-drawn, nautical. The analogue counterweight to the
> two sci-fi themes.

### Palette

| Token | Value | Use |
|---|---|---|
| `--bg` | `#F3E3C3` | Parchment |
| `--bg-elev` | `#EADAB8` | Aged panel |
| `--fg` | `#4A3420` | Ink brown — body text |
| `--fg-muted` | `#7A6247` | Faded ink |
| `--accent` | `#D7263D` | Straw-hat red |
| `--accent-2` | `#1B7A8C` | Ocean teal |
| `--accent-3` | `#E0A526` | Gold doubloon |
| `--ink-deep` | `#16324F` | Navy, for depth |

Contrast note: `#4A3420` on `#F3E3C3` ≈ 8.5:1 — safe. Do **not** put white text on parchment.

### Typography

- Display: **Alfa Slab One** or Cinzel — weighty, adventurous, slightly archaic
- Body: **Nunito** or a warm humanist serif — friendly, readable at length
- Accent/labels: a lightly condensed slab for "wanted poster" captions

`--radius: 6px` (soft, hand-cut) · `--border-w: 2px` · `--ease-signature: cubic-bezier(.3,.9,.4,1)`
(gentle, like paper settling) · `--dur-base: 380ms` — this theme is **slower** than the others,
deliberately.

### Geometry & motifs

Parchment grain (SVG `feTurbulence` baked to a static tile — never animated) · **dashed route
lines** connecting sections like a map path · compass rose · rope borders (a repeating SVG
pattern stroke) · **wanted-poster frames** for project cards, with torn edges via `clip-path` ·
wave dividers · wax-seal buttons · a Jolly-Roger-style personal stamp mark (draw your own).

### Ambient background

- Parchment grain overlay + a subtle vignette
- Animated wave divider between sections — an SVG path with two offset sine layers
- A slowly rotating compass in the corner
- A **log pose** indicator that physically points toward the next unread section
- Floating dust motes, very slow, very few (~15)
- A ship silhouette drifting across the horizon line every ~40 s

### Cursor

A **compass needle** — and it genuinely works: every frame it computes the nearest
`[data-interactive]` element and rotates to point at it, with a damped spring so it wobbles
and settles like a real needle.

- **Hover:** needle locks, and a small dashed "X" marker fades in under it
- **Down:** needle compresses

This is the most distinctive cursor of the five because it carries *information*, not just decoration.

### Click — Ink & treasure

1. **Ink splat** (0–600 ms) — a procedurally generated organic blob (12 points on a circle with randomized radii, joined by Catmull-Rom→bezier), filtered through `feTurbulence` + `feDisplacementMap` for a rough deckled edge. Scale 0→1 on `elastic.out(1, .6)`. Animate the turbulence `baseFrequency` for only the first ~400 ms, then freeze it — turbulence is the expensive filter
2. **Coin burst** (0–1200 ms) — 8–14 gold canvas particles launched in an arc with real gravity (`vy += 0.5` per frame), each spinning. Fake the disc spin by scaling the y-axis: `ctx.scale(1, Math.abs(Math.cos(rot)))` — the coin flattens and flips convincingly for almost no cost
3. **X stamp** (150–900 ms) — two dashed strokes draw in via dash-offset, hold, fade
4. **Parchment ripple** — a faint concentric ring, slow, low opacity

**Bonus interaction — the Gum-Gum stretch.** On buttons, `pointerdown` + drag stretches the
element toward the pointer (`translate(dx*.3, dy*.3)` plus a scale proportional to drag
distance and a `border-radius` skew); release springs it back with overshoot. The sound pitch
tracks the stretch distance live. It's the single most theme-appropriate interaction in the
whole project and costs almost nothing.

### Hover

Wanted-poster card lifts with a soft paper-curl shadow and rotates 1°. The wax seal spins 15°.
Dashed route lines to neighbouring sections brighten.

### Sound

| Event | Sound |
|---|---|
| Hover | Paper shuffle, 60 ms |
| Click | Wooden thunk + a short coin jingle layered on top |
| Stretch | *Synthesized:* pitch-bent sine tracking drag distance, released with a boing |
| Theme enter | Creaking ship timber + a gull |
| Music | Sea-shanty-flavoured acoustic loop, accordion/strings, ~110 BPM |

### Transition out

**Map unfurl.** Parchment scrolls open horizontally from the centre (or an ink wash floods
diagonally across the frame), covering the screen; tokens flip; the map rolls away.

---

## 5 · `banana-mode` — Minions

> **Chaos.** Toy-like, bouncy, squash-and-stretch. Pure comic relief — and the theme people
> will screenshot.

### Palette

| Token | Value | Use |
|---|---|---|
| `--bg` | `#FFD100` | Banana yellow |
| `--bg-elev` | `#2E5FA3` | Denim blue panels |
| `--fg` | `#1A1A1A` | Near-black — **required** for contrast on yellow |
| `--fg-on-elev` | `#FFFFFF` | White, only on the denim panels |
| `--accent` | `#2E5FA3` | Denim |
| `--accent-2` | `#7B2D8E` | Evil-minion purple |
| `--accent-3` | `#C9CDD2` | Goggle silver |

**Contrast warning:** this is the theme most likely to fail WCAG AA. White on `#FFD100` is
~1.7:1 — unusable. Body text on the yellow background must be `#1A1A1A` (≈13:1). White text
is allowed *only* on the denim panels (`#FFFFFF` on `#2E5FA3` ≈ 6.4:1). Enforce this with two
separate tokens rather than trusting yourself to remember.

### Typography

- Display: **Baloo 2** or Fredoka — chunky, rounded, friendly
- Pops: **Luckiest Guy**
- Body: **Nunito** — rounded terminals to match

`--radius: 999px` (everything is a pill or a circle) · `--border-w: 4px` (thick, with white
dashed "stitching" insets) · `--ease-signature: cubic-bezier(.34,1.56,.64,1)` (heavy overshoot)
· `--dur-base: 420ms` — slow enough that the overshoot reads.

### Geometry & motifs

**Goggle rings** as the universal framing device — two concentric circles plus a strap, wrapped
around avatars, images and icons · denim texture with white dashed stitching borders · rivets
in the corners · banana shapes · speech bubbles with gibberish (`BELLO!` `PAPOY!` `BANANAAA!`)
· **googly eyes that track the cursor** · everything rotated 1–3° off-axis so nothing sits square.

### Ambient background

- **Googly eyes on section headers** — pupil offset toward the pointer:
  `angle = atan2(my-ey, mx-ex)`, radius `min(maxR, dist*0.1)`. It's four lines of code and it's
  the single most memorable detail in the project
- Idle wobble on everything: ±2° rotation, 3 s, alternating, desynchronized per element
- A banana falls across the screen every ~20 s, tumbling
- A silhouette peeks in from a screen edge occasionally and ducks back out
- Every element enters with a spring overshoot (`stiffness: 400, damping: 10, mass: 0.8`)

### Cursor

A **banana** rotated to face the direction of travel (`atan2` on the velocity vector),
leaving a trail of tiny banana and confetti bits. On hover it becomes a **googly eye** whose
pupil looks in the direction of movement.

### Click — Banana blast

1. **Squash** (0–420 ms) — the clicked element squashes then springs back with **volume preservation**: `scaleY = 1/scaleX`, keyframed `1 → 1.25 → 0.9 → 1` on scaleX. Volume preservation is what separates real squash-and-stretch from "it got fatter"
2. **Confetti** (0–1400 ms) — `canvas-confetti` with custom shapes via `shapeFromText('🍌')` and `shapeFromPath(bananaPath)`, in yellow / denim / purple, with gravity and spin
3. **Word pop** (0–800 ms) — `BOING!` / `BELLO!` / `POP!` in Luckiest Guy, scaling in with heavy overshoot, wobbling on rotation as it drifts away
4. **Page wobble** — the whole wrapper shifts 2px with a spring. Tiny. Just enough to feel silly
5. **Long-press gag** — hold for 1.5 s and a rising "BANANAAA" builds, releasing into a bigger burst

### Hover

Element bounces on a spring, rotates 3°, nearby googly eyes swivel to look at it, and the
denim stitching border animates its dash offset (a marching-ants effect).

### Sound

| Event | Sound |
|---|---|
| Hover | Rubber squeak, 60 ms |
| Click | *Synthesized:* sine with a pitch envelope + LFO detune — a classic cartoon boing |
| Long press | Rising pitch build, then a pop |
| Theme enter | Crowd of gibberish chatter (synthesized formants — **not** clips) |
| Music | Goofy ukulele/brass loop, ~128 BPM |

### Transition out

**Crowd wipe.** A row of silhouettes runs across the viewport from left to right, filling the
screen mid-run; tokens flip; they keep running and clear the frame. Alternatively, a giant
banana peel unfurls across the screen.

---

## Cross-theme summary

| | `mark-vii` | `web-head` | `ki-surge` | `grand-line` | `banana-mode` |
|---|---|---|---|---|---|
| **Base** | Dark | Light | Dark | Light warm | Bright |
| **Radius** | 2px | 0 | 4px | 6px | 999px |
| **Border** | 1px hairline | 3px ink | 2px glow | 2px rope | 4px stitched |
| **Easing** | expo-out | snap | explosive | gentle | heavy overshoot |
| **Base dur** | 220ms | 180ms | 260ms | 380ms | 420ms |
| **Cursor** | Reticle + dot | Crosshair + silk | Ki orb | Compass needle | Banana / eye |
| **Click** | Repulsor | THWIP + splat | Charged blast | Ink + coins | Squash + confetti |
| **Render** | Canvas + DOM | SVG + canvas | Canvas (additive) | SVG + canvas | Confetti + spring |
| **Unique mechanic** | Magnetic snap | Web line draw | **Hold to charge** | Pointing needle | **Googly eyes** |
| **Sound** | Synth sci-fi | Funk | Rock/orchestral | Sea shanty | Ukulele |

**Design tension worth protecting:** the five easings and five radii vary more than the five
palettes do. If you ever have to cut scope, cut a colour, never a motion curve — the motion is
what makes them feel like different *places* rather than different *skins*.

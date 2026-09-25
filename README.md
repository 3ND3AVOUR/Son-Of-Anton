# Son-Of-Anton — Gamified Multi-Theme Portfolio

A portfolio that ships **five complete sensory skins**. Switching theme swaps the palette,
typography, motion language, ambient background, **mouse cursor**, click effects, interaction
sounds and background music — all at once.

| Codename | Inspiration | Cursor | Click effect |
|---|---|---|---|
| `mark-vii` | Iron Man | Targeting reticle + dot | Repulsor blast |
| `web-head` | Spider-Man | Crosshair + silk trail | THWIP + web splat |
| `ki-surge` | Dragon Ball Z | Ki orb | **Hold-to-charge** blast |
| `grand-line` | One Piece | Compass needle that points at links | Ink splat + coin burst |
| `banana-mode` | Minions | Banana / googly eye | Squash + banana confetti |

## Status

**Research and design complete. No implementation yet.**

## Documentation

| Doc | Contents |
|---|---|
| [`docs/01-RESEARCH.md`](docs/01-RESEARCH.md) | Concept · asset/IP strategy · stack choices with verified versions · theme-token architecture · performance budget · audio strategy · gamification · accessibility · roadmap |
| [`docs/02-THEME-SPECS.md`](docs/02-THEME-SPECS.md) | Full spec per theme: palette, typography, geometry, ambient background, cursor, hover, click effect, sounds, transition |
| [`docs/03-FX-COOKBOOK.md`](docs/03-FX-COOKBOOK.md) | Implementation recipes with code — the FX composer, cursor engine, pooled particle system, web lines, ink blobs, springs, audio manager, reduced-motion path |

## The four decisions that shape everything

1. **Original SVG assets, not ripped character art or soundtracks.** These are enforced
   properties and music rips get found automatically. Take the palettes, shapes and motion
   language; draw the rest yourself. It's legally clean, an order of magnitude lighter, and
   reads as design work rather than a fan page. — `01-RESEARCH.md` §2
2. **One data-driven FX composer, five config files.** Not five effect implementations. A
   sixth theme should cost a config, not a rewrite. — `03-FX-COOKBOOK.md` §1
3. **No Three.js in v1.** Every effect here is 2D canvas + SVG at a fraction of the
   weight. — `01-RESEARCH.md` §3
4. **Rate-limit every full-screen flash to one per 400 ms, globally.** WCAG 2.3.1 caps flashing
   at three per second and rapid clicking would blow straight through
   it. — `03-FX-COOKBOOK.md` §6.4

## Proposed stack

Vite 8 · React 19 · TypeScript 7 · Tailwind 4 · GSAP 3.15 (fully free since 2025, incl.
SplitText/MorphSVG/DrawSVG) · Motion 13 · Howler 2.2 · Zustand 5 · canvas-confetti 1.9

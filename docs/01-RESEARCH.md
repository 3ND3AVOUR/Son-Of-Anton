# Gamified Multi-Theme Portfolio — Research & Architecture

> Research date: **2026-09-25**. Versions verified against the npm registry on that date.

---

## 1. The concept in one paragraph

A single-page portfolio that ships **five complete sensory skins**. Switching the theme doesn't
just recolour the page — it swaps the typography, the motion language, the ambient background,
the mouse cursor, the click feedback, the UI chrome, the background music and every interaction
sound. Each theme is a small toy: the cursor *does* something, clicking *fires* something, and
exploring the site earns XP and unlockable achievements.

The five skins:

| # | Codename | Inspiration | Mood | Core verb |
|---|----------|-------------|------|-----------|
| 1 | `mark-vii`     | Iron Man        | Precision HUD, sci-fi engineering | **Target & fire** |
| 2 | `web-head`     | Spider-Man      | Comic book, Ben-Day dots, kinetic | **Shoot & stick** |
| 3 | `ki-surge`     | Dragon Ball Z   | Raw energy, aura, impact frames    | **Charge & blast** |
| 4 | `grand-line`   | One Piece       | Parchment, nautical, hand-drawn    | **Stamp & scatter** |
| 5 | `banana-mode`  | Minions         | Toy-like chaos, squash & stretch   | **Boing & burst** |

The five were chosen well: two are cold/sci-fi (**mark-vii**, **ki-surge**), one is
high-contrast print (**web-head**), one is warm/analogue (**grand-line**), one is pure
cartoon (**banana-mode**). That spread means the theme switcher actually *demonstrates*
range rather than showing off five variations of "glowy dark mode". Keep that spread —
it is the strongest thing about the concept.

---

## 2. Read this before you pick assets (it changes the design)

Iron Man, Spider-Man, Dragon Ball Z, One Piece and the Minions are all active, aggressively
enforced properties. Character art, logos, wordmarks and — above all — **soundtrack rips** are
protected. Music is the highest-risk item by a wide margin, because audio fingerprinting finds
it automatically; a portfolio playing the actual DBZ OST or a Minion voice clip is the thing
most likely to draw a takedown. Fan work in a portfolio sits in a grey zone that leans on fair
use, and fair use is a defence, not a shield.[^fanart]

**The design consequence, and it is a good one:** build an *aesthetic homage*, not a fan page.
Take the palette, the shapes, the typography and above all the **motion language** — and draw
every asset yourself as SVG. This route is:

- **Legally clean.** Colours, geometric motifs and easing curves aren't ownable.
- **Lighter.** A hand-drawn SVG web is ~2 KB and recolours from `currentColor`. A PNG render is 400 KB and is locked to one palette.
- **Better portfolio work.** "I designed five motion systems" reads as craft. "I downloaded five wallpapers" does not.

Concretely:

| Don't ship | Ship instead |
|---|---|
| Iron Man helmet render | An abstract glowing reactor ring + hex HUD grid you drew |
| The Spider-Man logo | Your own web geometry: 8 radial spokes + 4 catenary arcs |
| Goku sprites | An aura particle system + manga speed lines |
| Luffy / Jolly Roger art | A compass rose, rope borders, a wax seal, a wanted-poster frame |
| Minion character art | Goggle rings (2 circles + a strap), denim stitching, googly eyes |
| Any official OST | CC0 / royalty-free loops *in the genre*, or tracks you compose |
| Minion voice clips | Synthesized gibberish (pitch-shifted formants) or CC0 cartoon SFX |

Name the themes in the UI by codename — "Mark VII", "Web-Head", "Ki Surge", "Grand Line",
"Banana Mode". Evocative, instantly readable to anyone who knows the source, and not a
trademark use. The rest of these documents assume this approach.

If you decide you want literal character art anyway, that's your call to make — but keep the
music original regardless, and expect to swap assets if a notice ever arrives.

---

## 3. Recommended stack

Verified versions, 2026-09-25:

```
react           19.3.0     motion          13.4.3   (was framer-motion)
typescript       7.0.2     howler           2.2.4
vite             8.3.1     zustand          5.0.15
gsap            3.15.0     tailwindcss      4.3.3
@gsap/react      2.1.2     canvas-confetti  1.9.4
```

### Why each

**Vite + React + TypeScript, not Next.js.** This is a client-side, animation-heavy single page.
SSR buys you very little here and actively complicates the three systems that matter most —
the cursor, the canvas and the audio context — all of which are browser-only and need
`window` at mount. Vite's dev server and per-theme `import()` code-splitting are exactly
what this project needs. If you later add an SEO-sensitive blog, prerender that part.

**GSAP 3.15 as the primary timeline engine.** GSAP went **100% free in April 2025** after
Webflow acquired GreenSock — including every previously paid Club plugin: SplitText, MorphSVG,
DrawSVG, ScrollTrigger, InertiaPlugin.[^gsap] That is a big deal for this project specifically:

- `DrawSVG` → web lines, speed lines, HUD reticles drawing themselves on
- `MorphSVG` → cursor morphing between theme shapes
- `SplitText` (rewritten, ~50% smaller) → per-character text reveals, the "decrypt" effect
- `ScrollTrigger` → section-entry choreography
- `gsap.quickTo()` → the single most important API for the cursor (see below)

**Motion 13 (`motion/react`) alongside it, not instead of it.** Framer Motion became an
independent project and was renamed **Motion** in mid-2025; import from `motion/react`, not
`framer-motion`.[^motion] It is the same library by the same author, with a hybrid engine that
runs on the Web Animations API natively and falls back to JS for springs and gestures. Use it
for what it's genuinely better at: component mount/unmount (`AnimatePresence`), layout
animations, and **spring physics** — which is the entire soul of the Minion theme. Use GSAP for
timelines, scroll and SVG. They coexist fine; just never animate the same property of the same
element with both.

**Howler 2.2.4 for audio.** Howler is the standard for browser game audio: it wraps Web Audio
and HTML5 Audio behind one API, handles format fallback, handles autoplay policy, and supports
**audio sprites** — one file, many sounds, tight latency.[^howler] Tone.js is a synthesis and
scheduling library for building instruments; it's the wrong tool for "play a thwip." Use the raw
Web Audio API directly for the handful of *synthesized* effects (see §6).

**Zustand 5 for state.** Theme, audio settings, XP and unlocked achievements are read by
dozens of components every frame-ish. React Context would re-render the tree on every change.
Zustand gives selector-level subscriptions in ~1 KB.

**Tailwind 4 with CSS custom properties.** Tailwind v4's `@theme` block compiles directly to
CSS custom properties, which is exactly the primitive the theme system is built on (§4). No
config gymnastics needed.

**canvas-confetti 1.9 for the Minion theme only.** 6 KB gzipped, zero deps, one shared canvas
on `requestAnimationFrame`, and it supports `shapeFromPath()` and `shapeFromText()` — so you
get flying bananas for almost nothing.[^confetti]

### Explicitly rejected

**Three.js / React Three Fiber.** Every effect in this document is achievable with 2D canvas +
SVG at a fraction of the weight. Three.js core is ~600 KB before you add post-processing, and
bloom passes are a real mobile performance cost.[^three] Skip it for v1. Add it later *only* if
you want one genuinely 3D showpiece (a rotating arc reactor would be the obvious candidate).

**tsParticles.** Full package runs 20–100 KB depending on plugins and is configuration-driven
rather than bespoke.[^particles] You need five *different* particle behaviours with specific art
direction — a ~150-line custom engine with object pooling (§5) will be smaller, faster and do
exactly what you want.

**Rive / Lottie.** Rive is genuinely the better of the two for interactive UI in 2026 — it has
real state machines, data binding, and near-zero idle CPU, where Lottie has a per-instance
keyframe ticker that compounds linearly past ~20 simultaneous animations.[^rive] But the Rive
WASM runtime is a ~200 KB bundle cost. **Defer to v2**, where it would be the right call for one
thing: an animated mascot per theme that reacts to page state. Not needed for cursors or FX.

---

## 4. Theme system architecture

### 4.1 Token routing

Everything visual resolves through **semantic** CSS custom properties. Components never
reference a theme-specific value; they reference the semantic one, and the active theme
redefines it. Swapping `data-theme` on `<html>` repaints the entire site in one style
recalculation.[^theming]

```css
/* tokens.css — the contract every component codes against */
:root {
  --bg, --bg-elev, --fg, --fg-muted;
  --accent, --accent-2, --accent-glow;
  --stroke, --stroke-strong, --radius, --border-w;
  --font-display, --font-body, --font-mono;
  --ease-signature;          /* each theme has its OWN easing curve */
  --dur-fast, --dur-base, --dur-slow;
  --fx-primary, --fx-secondary, --fx-particle-count;
  --texture;                 /* url(#grain) | none */
}

[data-theme="mark-vii"] {
  --bg: #0A0E14;  --fg: #E7F6FF;  --accent: #4FE3FF;  --accent-2: #FFB800;
  --radius: 2px;  --border-w: 1px;
  --font-display: "Chakra Petch", sans-serif;
  --ease-signature: cubic-bezier(.16,1,.3,1);   /* sharp, mechanical */
  --dur-base: 220ms;
}

[data-theme="banana-mode"] {
  --bg: #FFD100;  --fg: #1A1A1A;  --accent: #2E5FA3;  --accent-2: #7B2D8E;
  --radius: 999px; --border-w: 4px;
  --font-display: "Baloo 2", cursive;
  --ease-signature: cubic-bezier(.34,1.56,.64,1);  /* overshoot */
  --dur-base: 420ms;
}
```

**`--ease-signature` and `--radius` are doing as much work as the colours.** Iron Man is
2px-radius and snaps; Minions is fully-round and overshoots. Get those two right and the themes
feel different even in greyscale.

### 4.2 Avoiding the theme flash

Apply the stored theme in a **blocking inline script in `<head>`**, before first paint —
otherwise the first frame renders in the default theme and visibly snaps.[^theming]

```html
<script>
  try {
    var t = localStorage.getItem('theme') || 'mark-vii';
    document.documentElement.dataset.theme = t;
  } catch (e) {}
</script>
```

Also disable transitions during the swap so tokens don't cross-fade through mud:

```js
root.dataset.switching = '1';
root.dataset.theme = next;
requestAnimationFrame(() => requestAnimationFrame(() => delete root.dataset.switching));
```
```css
[data-switching] *, [data-switching] *::before { transition: none !important; }
```

Never use `transition: all` on theme changes — it forces the browser to recalculate layout and
paint paths for properties that never changed.[^theming] Transition only `background-color`,
`color`, `border-color`, `fill`.

### 4.3 The switch is a set piece, not a toggle

Don't cross-fade. **Cover the swap.** Run the theme's own transition animation full-screen, flip
the tokens at its midpoint while the screen is obscured, then reveal. Each theme owns its exit:

- `mark-vii` → HUD boot: grid draws in, panels slide from the edges as 1px outlines then fill
- `web-head` → comic panel wipe: 4 panels sweep across with black gutters
- `ki-surge` → power-up: white flash + gold aura engulf, then a horizontal scanline split
- `grand-line` → map unfurl: parchment scrolls open, or an ink wash floods the frame
- `banana-mode` → a crowd of silhouettes runs across the viewport wiping it

Implementation: an overlay `<div>` on `z-index: 9999`, with the token flip fired from a GSAP
timeline callback at the halfway point.

**The View Transitions API is a progressive enhancement here, not the mechanism.**
Same-document transitions are widely supported (Firefox shipped them in 144), but
cross-document is still "Limited availability" and explicitly not Baseline.[^vt] Since this is
a single page you only need same-document — so wrap the swap in
`document.startViewTransition?.(...)` for free frame-interpolation where supported, and keep
your GSAP overlay as the actual choreography so the experience is identical everywhere.

### 4.4 Per-theme code splitting

Each theme's FX module, fonts and audio sprite are a separate lazily-imported chunk. A visitor
who never leaves Iron Man never downloads the One Piece ink shader or the Minion confetti.

```
src/themes/mark-vii/index.ts      →  dynamic import()
src/themes/web-head/index.ts      →  dynamic import()
```

Prefetch on hover/focus of the theme switcher so the switch still feels instant.

---

## 5. Performance budget

Animation is now judged by Core Web Vitals — INP in particular — so the 2026 bar is
"purposeful, restrained, reduced-motion aware", not "as much as the GPU will take".[^a11y]
Treat these as hard rules:

**Rules**

1. **Animate only `transform` and `opacity`.** Never `top`/`left`/`width`/`box-shadow` in a loop.
2. **One `requestAnimationFrame` loop for the whole site.** Use `gsap.ticker` and register the
   canvas engine and the cursor lerp onto it. Never run competing rAF loops.
3. **Stop the loop when idle.** If zero particles are alive, don't re-request the frame. Restart
   on the next emit. This is the difference between 0% and 4% idle CPU.
4. **Pool particles.** Pre-allocate ~500 objects and reuse them. Allocating in the hot loop
   causes GC pauses that read as stutter.
5. **Cap `devicePixelRatio` at 2.** A 3x-DPR phone rendering a full-screen canvas at native
   resolution is 2.25× the fill rate for no visible gain.
6. **`globalCompositeOperation = 'lighter'` is the expensive path.** Use additive blending only
   where it earns the look (repulsor sparks, ki aura) — not for ink blots or confetti.
7. **SVG filters: `feColorMatrix` is cheap, `feTurbulence` is expensive.**[^svg] Use turbulence
   for the One Piece ink roughness and the parchment grain — but bake the grain to a static
   pattern, and only ever animate one turbulence node at a time, for under ~400 ms.
8. **GSAP `quickTo` over manual lerp.** A hand-written `x += (target-x)*0.1` moves twice as fast
   at 120 Hz as at 60 Hz. `quickTo` is duration-based and therefore frame-rate independent —
   the problem never arises.[^quickto] If you do hand-roll a lerp, multiply by
   `gsap.ticker.deltaRatio(60)`.
9. **`gsap.ticker.add()` has no auto-cleanup.** It fires forever until `gsap.ticker.remove(fn)`.
   In React, that call goes in the `useGSAP` cleanup return.[^quickto]
10. **Store cursor coords and `quickTo` setters in refs.** Never in React state — that's a
    re-render per mouse move.

**Targets**

| Metric | Budget |
|---|---|
| Initial JS (gzip) | ≤ 200 KB |
| Per-theme chunk | ≤ 60 KB |
| Per-theme audio sprite | ≤ 150 KB (Opus) |
| Frame time, desktop | ≤ 8 ms (120 Hz headroom) |
| Frame time, mid-tier mobile | ≤ 16 ms |
| Idle CPU | ~0% |
| Live particles, desktop | ≤ 250 |
| Live particles, mobile | ≤ 100 |

**Mobile degradation** (branch on `(hover: hover) and (pointer: fine)`):
custom cursor off entirely; particle counts halved; screen shake reduced; ambient background
loops paused off-screen; FX fire at the touch point on `touchstart`.

---

## 6. Audio strategy

### Autoplay is a hard constraint, not an inconvenience

An `AudioContext` created before a user gesture starts **suspended**; you must call
`resume()` after a real interaction.[^autoplay] So:

- Nothing plays on load. Ever.
- Show a tasteful "🔊 Sound on" affordance in the corner.
- On the first `pointerdown` anywhere, call `Howler.ctx.resume()` and start the music.
- Persist mute state and volume to `localStorage`. Default to **muted** — a portfolio that
  blares music at a recruiter in an open-plan office is a portfolio that gets closed.

### Format and delivery

- **WebM/Opus with MP3 fallback.** Opus at 96 kbps matches or beats MP3 at 128 kbps at roughly
  75% of the size.[^audiofmt] Howler handles the fallback automatically from the `src` array.
- **SFX → one audio sprite per theme.** All of a theme's clicks, hovers and pops in a single
  file, played by offset. One request, no per-sound latency.[^audiofmt]
- **Music → separate file per theme**, loaded with `html5: true` so it streams rather than
  decoding the whole track into memory.
- Decode sprites during the theme-switch transition, not during interaction.

### Crossfading on theme change

```js
current.fade(current.volume(), 0, 600);
next.volume(0); next.play(); next.fade(0, targetVol, 600);
setTimeout(() => current.pause(), 620);   // pause, don't stop — resumes on return
```

### Synthesized effects (zero bytes)

Several effects are *better* synthesized, because they can vary continuously with the
interaction — which a fixed sample can't:

- **Repulsor charge** — a sawtooth ramping 180 → 900 Hz over 120 ms, into a lowpass
- **Ki charge** — a sine whose frequency tracks charge duration in real time, so the pitch
  literally is the charge meter
- **Minion boing** — a sine with a pitch envelope plus an LFO on the detune
- **Gum-Gum stretch** — pitch mapped live to drag distance

That's ~40 lines of raw Web Audio and it makes the interactions feel responsive in a way
samples cannot.

### Anti-fatigue rules

These matter more than the sounds themselves:

- **Randomize rate** on every SFX: `rate: 0.92 + Math.random() * 0.16`. Without this, repeated
  clicks sound like a machine gun and become unbearable in about eight seconds.
- **Debounce identical sounds** within 40 ms.
- **Cap concurrent SFX** at ~6; drop the oldest.
- **Duck the music** by ~25% for 150 ms under a big FX hit.
- Keep SFX **short** — 60–200 ms. Hover sounds especially: under 80 ms, quiet, or they'll
  drive people mad.

### Sourcing

CC0 (no attribution, commercial-safe): **Freesound** filtered to CC0 — roughly half of its
730k+ sounds; **Kenney** audio packs, all CC0; **Pixabay** and **Mixkit**, no credit required;
the annual **Sonniss GDC** bundle. ZapSplat's free tier allows commercial use *with*
attribution.[^sfx] Filter deliberately: CC-BY-NC is not usable here.

---

## 7. Gamification layer

**XP sources:** first visit to a section · opening a project case study · switching to each
theme · triggering N click effects · finding an easter egg · completing the contact form.

**Achievements** (theme-flavoured, persisted to `localStorage`):

| Achievement | Trigger |
|---|---|
| *Assemble* | Visit all five themes |
| *Mark I* | Fire 25 repulsor blasts |
| *With Great Power* | Click 50 times in Web-Head |
| *It's Over 9000* | Hold a ki charge for 9 seconds |
| *Pirate King* | Find the hidden treasure marker |
| *Bello!* | Click a banana 10 times |
| *Contra Kid* | Konami code — ↑↑↓↓←→←→BA[^konami] |
| *Completionist* | Every section + every theme + every egg |

**The HUD is one component with five skins** — the same progress primitive rendered as an arc
reactor fill / a web-fluid meter / a scouter power-level readout / a log pose needle / a banana
counter. Build it once against the token contract.

**The important caveat:** a recruiter has 90 seconds and wants to find your work. Gamification
must never be load-bearing for actual content. Two non-negotiables:

1. Every piece of real information is reachable in ≤ 2 clicks with the FX layer entirely off.
2. Ship a **"Focus mode"** toggle — pinned, always visible — that kills sound, particles,
   cursor FX and XP in one click, leaving a clean, fast, readable portfolio.

The game is the reward for people who want to play. It is never the toll for people who don't.

---

## 8. Accessibility — the parts that will actually bite

**Flashing is the real hazard.** WCAG 2.3.1 caps flashes at **three per second**. The DBZ
impact frame, the lightning crackle and the Iron Man muzzle flash can all violate this if
someone clicks rapidly. **Rate-limit every full-screen flash to one per 400 ms, globally**, and
cap its opacity. This is the single most important item in this document that is easy to get
wrong.

**`prefers-reduced-motion` is table stakes, not a nice-to-have.** Vestibular disorders and
migraine triggers are common, and motion that reads as smooth to you can cause nausea for
someone else.[^a11y] When reduced motion is set:

- No screen shake, no parallax, no full-screen flash, no auto-playing ambient loops
- Click feedback degrades to a simple 150 ms opacity pulse — **keep feedback, remove motion**
- Section entrances become cross-fades
- The custom cursor stays but stops trailing (it becomes a plain follower)

Provide an in-page toggle too — plenty of affected people are on a shared or work machine and
have never set the OS flag.[^a11y]

**Custom cursor rules.** `cursor: none` only inside `@media (hover: hover) and (pointer: fine)`.
Restore the native caret on `input`, `textarea` and `[contenteditable]`. Keyboard navigation and
visible focus rings must work with the cursor system entirely bypassed. All FX layers get
`aria-hidden="true"` and `pointer-events: none`.

**Contrast will fail in two themes if you're not careful.** Banana yellow `#FFD100` and
parchment `#F3E3C3` are light backgrounds; white or mid-grey text on them fails AA. Both themes
must use near-black body text (`#1A1A1A`, `#4A3420`) — this is specified per theme in
`02-THEME-SPECS.md`. Run every foreground/background pair through a contrast check before
shipping, including muted text and disabled states.

**Audio.** Never autoplay. Always mutable. Never the sole carrier of information.

---

## 9. Build roadmap

**Phase 0 — Foundation.** Vite + React + TS + Tailwind 4. Token contract in `tokens.css`.
Zustand stores. Zero-flash theme bootstrap. Ship with two themes' *colours only*, no FX.

**Phase 1 — The cursor engine.** One cursor component, state machine
(`idle | hover | down | text | disabled`), `gsap.quickTo` follow, magnetic snapping, mobile and
reduced-motion guards. Iron Man visuals only.

**Phase 2 — The FX composer.** The data-driven layer system from `03-FX-COOKBOOK.md` §1:
pooled canvas particles, SVG stamp layer, DOM ring/text layer, screen effects. Build
`mark-vii` and `web-head` end to end as proof the abstraction holds.

**Phase 3 — Audio.** Howler sprite loader, gesture gate, mute persistence, per-theme music
crossfade, rate randomization, the synthesized charge sounds.

**Phase 4 — Remaining themes.** `ki-surge`, `grand-line`, `banana-mode`. If the composer is
right, each should be a config file plus one or two bespoke draw functions.

**Phase 5 — Content and layout.** The actual portfolio: hero, about, projects, skills, contact.
Every section built against tokens so it reskins for free.

**Phase 6 — Gamification.** XP store, achievements, the five HUD skins, toasts, easter eggs.

**Phase 7 — Polish.** Theme transition set pieces. Focus mode. Accessibility audit. Contrast
pass. Performance profiling on a real mid-tier Android. Lighthouse.

**Order matters:** the cursor and FX engine come *before* content, because they determine the
token contract. Building the site first and retrofitting the FX means rewriting the site.

---

## References

[^gsap]: [GSAP is Now Completely Free, Even for Commercial Use — CSS-Tricks](https://css-tricks.com/gsap-is-now-completely-free-even-for-commercial-use/) · [Webflow makes GSAP 100% free](https://webflow.com/blog/gsap-becomes-free) · [From SplitText to MorphSVG: 5 Creative Demos Using Free GSAP Plugins — Codrops](https://tympanus.net/codrops/2025/05/14/from-splittext-to-morphsvg-5-creative-demos-using-free-gsap-plugins/)
[^motion]: [Motion (prev. Framer Motion)](https://motion.dev/) · [Motion for React docs](https://motion.dev/docs/react)
[^howler]: [howler.js](https://howlerjs.com/) · [howler vs tone.js vs wavesurfer 2026 — PkgPulse](https://www.pkgpulse.com/guides/howler-vs-tone-js-vs-wavesurfer-web-audio-javascript-2026) · [Tone.js vs Howler.js — Supadark](https://supadark.com/notes/tone-js-vs-howler-js)
[^confetti]: [canvas-confetti vs tsparticles vs party.js 2026 — PkgPulse](https://www.pkgpulse.com/guides/canvas-confetti-vs-tsparticles-vs-party-js-celebration-2026)
[^three]: [The Complete Guide to Three.js Post-Processing in 2026](https://threejsroadmap.com/blog/the-complete-guide-to-threejs-post-processing-in-2026) · [three.js UnrealBloom example](https://threejs.org/examples/webgl_postprocessing_unreal_bloom.html)
[^particles]: [tsParticles](https://particles.js.org/) · [10 Best Particles Animation JavaScript Libraries (2026) — CSS Script](https://www.cssscript.com/best-particles-animation/)
[^rive]: [Lottie vs Rive: File Size, Performance & Which to Use in 2026](https://unicornicons.com/blog/lottie-vs-rive-performance) · [Rive vs Lottie in 2026 — Rive Masterclass](https://www.rivemasterclass.com/blog/rive-vs-lottie-in-20260why-interactive-logic-data-binding-scripting-make-rive-the-future-of-ui-animation)
[^theming]: [Multi-Theme Design System: CSS Variables + Data Attributes](https://www.hirejeffgreen.com/blog/multi-theme-design-system-css-variables) · [How to create better themes with CSS variables — LogRocket](https://blog.logrocket.com/create-better-themes-with-css-variables/)
[^vt]: [View Transition API — MDN](https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API) · [The View Transitions API in 2026](https://brainstormsandraves.com/css/view-transitions-2026/) · [Cross-Document View Transitions: The Gotchas Nobody Mentions — CSS-Tricks](https://css-tricks.com/cross-document-view-transitions-part-1/)
[^autoplay]: [Web Audio, Autoplay Policy and Games — Chrome for Developers](https://developer.chrome.com/blog/web-audio-autoplay) · [Autoplay policy in Chrome](https://developer.chrome.com/blog/autoplay) · [Web Audio API best practices — MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Best_practices)
[^audiofmt]: [Web Audio Formats: What to Use in 2026 — AudioUtils](https://audioutils.com/guide/audio-for-web-developers) · [How to Optimize Web Audio Performance (2026) — Supadark](https://supadark.com/notes/how-to-optimize-audio-performance-on-my-website)
[^sfx]: [Free Sound Effects and Music for Games (2026) — Cinevva](https://app.cinevva.com/guides/free-sound-effects-music) · [25 Free Game Sound Effects & Music Libraries (2026)](https://gamineai.com/resources/25-free-game-sound-effects-music-libraries) · [ZapSplat CC0 1.0 Universal licence](https://www.zapsplat.com/license-type/cc0-1-0-universal/)
[^a11y]: [prefers-reduced-motion — MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/@media/prefers-reduced-motion) · [Design accessible animation and movement — Pope Tech](https://blog.pope.tech/2025/12/08/design-accessible-animation-and-movement/) · [Web Animation Trends 2026 — MotionKit](https://motionkit.io/blog/web-animation-trends-2026)
[^svg]: [SVG Filter Effects: Blur, Shadow, and Color Tricks (2026)](https://www.svg2png.org/blog/svg-filter-effects-guide) · [SVG Filters for Branding: Grain, Glows, and Duotone That Render Fast](https://theyellowflashlight.com/svg-filters-branding/)
[^quickto]: [gsap.quickTo() — GSAP docs](https://gsap.com/docs/v3/GSAP/gsap.quickTo()/) · [Cursor Follower — GSAP Demo Hub](https://demos.gsap.com/demo/cursor-follower/)
[^konami]: [How to create a Konami Code easter egg with vanilla JS — Go Make Things](https://gomakethings.com/how-to-create-a-konami-code-easter-egg-with-vanilla-js/)
[^fanart]: [Fanart for Sale: Navigating Copyright and Trademark Risks — Odin Law](https://odinlaw.com/blog-fanart-copyright-trademark-risks/) · [How to Avoid Copyright Infringement When Selling Fan Art — Nolo](https://www.nolo.com/legal-encyclopedia/can-you-legally-sell-fan-art-online.html)

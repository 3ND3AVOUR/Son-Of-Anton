# FX Cookbook — How Each Effect Is Actually Built

Implementation recipes for the cursor, the click effects and the supporting engines.
Code is illustrative TypeScript; it compiles conceptually, not necessarily literally.

---

## 1 · The core idea: one composer, five configs

**Do not write five separate effect implementations.** Write one effect *composer* that plays a
list of layers, and make each theme a data file. Adding a sixth theme then costs a config, not
a rewrite.

```ts
type FXLayer =
  | { kind: 'ring';       color: string; width: number; toScale: number; dur: number; ease: string; delay?: number }
  | { kind: 'flash';      color: string; blur: number; toScale: number; dur: number }
  | { kind: 'particles';  count: number; speed: [number, number]; gravity: number;
                          drag: number; life: [number, number]; draw: ParticleDrawFn; additive: boolean }
  | { kind: 'svgStamp';   symbol: string; toScale: number; ease: string; dur: number; randomRotate: boolean }
  | { kind: 'projectile'; from: AnchorFn; curve: number; color: string; width: number; dur: number }
  | { kind: 'lines';      count: number; len: [number, number]; dur: number; stagger: number }
  | { kind: 'text';       words: string[]; font: string; rise: number; dur: number }
  | { kind: 'screen';     effect: 'shake' | 'flash' | 'bloom'; amount: number; dur: number };

interface ClickRecipe {
  layers: FXLayer[];
  sound: string;
  haptic?: number[];
  charge?: { maxMs: number; scales: Array<keyof FXLayer> };
}
```

`mark-vii` then reduces to:

```ts
export const repulsor: ClickRecipe = {
  sound: 'repulsor',
  haptic: [8, 24],
  layers: [
    { kind: 'projectile', from: bottomRightPalm, curve: 0,  color: 'var(--accent)', width: 2, dur: 60 },
    { kind: 'flash',  color: '#fff', blur: 6, toScale: 3, dur: 180 },
    { kind: 'ring',   color: 'var(--accent)', width: 2, toScale: 22, dur: 420, ease: 'expo.out' },
    { kind: 'ring',   color: 'var(--accent)', width: 1, toScale: 30, dur: 420, ease: 'expo.out', delay: 80 },
    { kind: 'particles', count: 12, speed: [4, 11], gravity: 0, drag: 0.92,
      life: [300, 600], draw: drawSpark, additive: true },
    { kind: 'screen', effect: 'bloom', amount: 0.06, dur: 120 },
  ],
};
```

The composer dispatches each layer to the right renderer:

```ts
export function fire(recipe: ClickRecipe, x: number, y: number, charge = 0) {
  if (reducedMotion()) return fireReduced(x, y);      // opacity pulse only
  audio.play(recipe.sound, { rate: 0.92 + Math.random() * 0.16 });
  navigator.vibrate?.(recipe.haptic ?? []);
  for (const layer of recipe.layers) {
    switch (layer.kind) {
      case 'particles':  particles.emit(layer, x, y, charge); break;
      case 'ring':
      case 'flash':
      case 'text':       domFX.spawn(layer, x, y, charge);    break;
      case 'svgStamp':
      case 'projectile':
      case 'lines':      svgFX.spawn(layer, x, y, charge);    break;
      case 'screen':     screenFX.run(layer, charge);         break;
    }
  }
}
```

### Why three render layers, not one

| Layer | Used for | Why |
|---|---|---|
| **Canvas** (`z:9997`) | Particles, sparks, coins, silk trail | Thousands of cheap draws per frame; DOM can't |
| **SVG** (`z:9998`) | Web lines, ink blobs, speed lines, reticles, stamps | Free crisp scaling and `stroke-dashoffset` draw-on, which is exactly the "draws itself" look |
| **DOM** (`z:9999`) | Rings, flashes, word pops | Free text rendering and CSS transitions; only a handful alive at once |

All three are `position: fixed; inset: 0; pointer-events: none; aria-hidden="true"`.

---

## 2 · The cursor engine

### 2.1 Guards first

```ts
const canCustomCursor =
  window.matchMedia('(hover: hover) and (pointer: fine)').matches;
```

```css
@media (hover: hover) and (pointer: fine) {
  html[data-cursor="on"] * { cursor: none; }
  /* never steal the caret */
  html[data-cursor="on"] input,
  html[data-cursor="on"] textarea,
  html[data-cursor="on"] [contenteditable] { cursor: text; }
}
```

### 2.2 Follow with `gsap.quickTo`

`quickTo` is the correct primitive. It's duration-based, so it's frame-rate independent by
construction — a hand-rolled `x += (target - x) * 0.1` runs twice as fast on a 120 Hz display.

```ts
useGSAP(() => {
  const dotX  = gsap.quickTo('#cur-dot',  'x', { duration: 0.08, ease: 'power3' });
  const dotY  = gsap.quickTo('#cur-dot',  'y', { duration: 0.08, ease: 'power3' });
  const ringX = gsap.quickTo('#cur-ring', 'x', { duration: 0.45, ease: 'power3' });
  const ringY = gsap.quickTo('#cur-ring', 'y', { duration: 0.45, ease: 'power3' });

  const onMove = (e: PointerEvent) => {
    pos.current.x = e.clientX; pos.current.y = e.clientY;   // ref, NOT state
    dotX(e.clientX);  dotY(e.clientY);
    ringX(e.clientX); ringY(e.clientY);
  };

  window.addEventListener('pointermove', onMove, { passive: true });
  return () => window.removeEventListener('pointermove', onMove);
});
```

The **two different durations are the whole effect** — the ring lagging behind the dot is what
makes a cursor feel alive. 0.08 s and 0.45 s is a good starting ratio.

> If you ever hand-roll a lerp instead, multiply the step by `gsap.ticker.deltaRatio(60)`.
> And remember `gsap.ticker.add()` never auto-cleans — the callback fires forever until you
> call `gsap.ticker.remove(fn)` in the effect's cleanup.

### 2.3 State machine

```ts
type CursorState = 'idle' | 'hover' | 'down' | 'text' | 'disabled';
```

Drive it from delegated listeners on `document`, matching `closest()` against
`a, button, [data-interactive]`. Set `data-cursor-state` on the cursor root and let **CSS**
handle the visual morph — the JS should only ever change one attribute.

```css
#cur-ring { width:28px; height:28px; transition: width var(--dur-fast), height var(--dur-fast); }
[data-cursor-state="hover"] #cur-ring { width:56px; height:56px; }
[data-cursor-state="down"]  #cur-ring { width:20px; height:20px; }
```

### 2.4 Magnetic snapping

```ts
function magnetize(el: HTMLElement, mx: number, my: number, pull = 0.25) {
  const r = el.getBoundingClientRect();
  const cx = r.left + r.width / 2, cy = r.top + r.height / 2;
  const dx = mx - cx, dy = my - cy;
  const dist = Math.hypot(dx, dy);
  const radius = Math.max(r.width, r.height) * 0.9;
  if (dist > radius) return gsap.to(el, { x: 0, y: 0, duration: 0.4, ease: 'elastic.out(1,.4)' });
  gsap.to(el, { x: dx * pull, y: dy * pull, duration: 0.3, ease: 'power3.out' });
}
```

Cache the `getBoundingClientRect()` on `pointerenter` and invalidate on scroll/resize — calling
it every `pointermove` forces layout on every frame.

### 2.5 Direction-facing cursor (banana / compass)

```ts
const vx = x - prev.x, vy = y - prev.y;
if (Math.hypot(vx, vy) > 1.5) {                    // deadzone stops jitter at rest
  const target = Math.atan2(vy, vx) * 180 / Math.PI;
  rotTo(target);                                   // another gsap.quickTo
}
prev = { x, y };
```

### 2.6 The compass needle that points at things (One Piece)

The only cursor that carries information. Maintain a cached list of interactive rects; each
frame, find the nearest and spring the needle toward it.

```ts
gsap.ticker.add(() => {
  const { x, y } = pos.current;
  let best: DOMRect | null = null, bestD = Infinity;
  for (const r of cachedRects) {
    const d = Math.hypot(r.left + r.width/2 - x, r.top + r.height/2 - y);
    if (d < bestD) { bestD = d; best = r; }
  }
  const target = best && bestD < 600
    ? Math.atan2(best.top + best.height/2 - y, best.left + best.width/2 - x) * 180/Math.PI
    : needleAngle + 0.25;                          // idle slow drift
  // damped spring so it wobbles and settles like a real needle
  needleVel += shortestAngle(needleAngle, target) * 0.08;
  needleVel *= 0.82;
  needleAngle += needleVel;
  gsap.set('#needle', { rotate: needleAngle });
});
```

Rebuild `cachedRects` on scroll (throttled) and resize — never per frame.

---

## 3 · The particle engine (~150 lines, pooled)

The single most important detail is **object pooling**. Allocating particles in the hot loop
produces GC pauses that read as stutter.

```ts
interface P { x:number; y:number; vx:number; vy:number; life:number; max:number;
              size:number; rot:number; vrot:number; hue:number; alive:boolean; draw:number }

const POOL: P[] = Array.from({ length: 500 }, makeDeadParticle);
let active = 0;
let running = false;

export function emit(cfg: ParticleLayer, x: number, y: number, charge = 0) {
  const n = Math.round(cfg.count * (1 + charge) * densityScale());   // densityScale: .5 on mobile
  for (let i = 0; i < n; i++) {
    const p = POOL.find(p => !p.alive);
    if (!p) break;                                    // pool exhausted: drop, never allocate
    const a = Math.random() * Math.PI * 2;
    const s = rand(cfg.speed[0], cfg.speed[1]) * (1 + charge * 0.8);
    p.x = x; p.y = y;
    p.vx = Math.cos(a) * s; p.vy = Math.sin(a) * s;
    p.life = 0; p.max = rand(cfg.life[0], cfg.life[1]);
    p.rot = Math.random() * 6.28; p.vrot = rand(-0.2, 0.2);
    p.alive = true; active++;
  }
  if (!running) { running = true; gsap.ticker.add(tick); }
}

function tick() {
  ctx.clearRect(0, 0, W, H);
  if (additive) ctx.globalCompositeOperation = 'lighter';
  for (const p of POOL) {
    if (!p.alive) continue;
    p.vy += cfg.gravity;
    p.vx *= cfg.drag; p.vy *= cfg.drag;
    p.x += p.vx; p.y += p.vy; p.rot += p.vrot;
    p.life += gsap.ticker.deltaRatio(60) * 16.67;
    if (p.life >= p.max) { p.alive = false; active--; continue; }
    drawFns[p.draw](ctx, p, 1 - p.life / p.max);      // t=1 fresh → 0 dead
  }
  ctx.globalCompositeOperation = 'source-over';
  if (active === 0) { running = false; gsap.ticker.remove(tick); }   // ← idle CPU ≈ 0
}
```

That last line is the difference between a portfolio that idles at 0% CPU and one that melts
laptop batteries in a background tab.

### Canvas setup

```ts
const dpr = Math.min(window.devicePixelRatio || 1, 2);   // cap at 2
canvas.width  = innerWidth  * dpr;
canvas.height = innerHeight * dpr;
canvas.style.width  = innerWidth  + 'px';
canvas.style.height = innerHeight + 'px';
ctx.scale(dpr, dpr);
```

### Per-theme draw functions

```ts
// Iron Man spark — additive, gold→cyan over life
const drawSpark = (c, p, t) => {
  c.globalAlpha = t;
  c.fillStyle = `hsl(${lerp(190, 42, 1 - t)} 100% ${50 + t * 30}%)`;
  c.fillRect(p.x, p.y, 2, 2 + (1 - t) * 4);           // streak, not a dot
};

// One Piece coin — fake a spinning disc by flattening the y-axis
const drawCoin = (c, p, t) => {
  c.save(); c.translate(p.x, p.y);
  c.scale(1, Math.abs(Math.cos(p.rot)));               // <- the whole trick
  c.globalAlpha = t;
  c.fillStyle = '#E0A526'; c.beginPath();
  c.arc(0, 0, p.size, 0, 6.283); c.fill();
  c.strokeStyle = '#B8842A'; c.stroke();
  c.restore();
};

// DBZ ki mote — additive glow
const drawKi = (c, p, t) => {
  const g = c.createRadialGradient(p.x, p.y, 0, p.x, p.y, p.size * 3);
  g.addColorStop(0, `rgba(255,255,255,${t})`);
  g.addColorStop(0.4, `rgba(255,213,74,${t * 0.8})`);
  g.addColorStop(1, 'rgba(255,109,0,0)');
  c.fillStyle = g; c.beginPath(); c.arc(p.x, p.y, p.size * 3, 0, 6.283); c.fill();
};
```

---

## 4 · Spider-Man: the web line and the splat

### 4.1 The line

A quadratic bezier from a wrist anchor to the click point, with the control point pushed
**perpendicular to the chord** so the web sags naturally instead of arcing arbitrarily.

```ts
function webPath(ox: number, oy: number, tx: number, ty: number, sag = 0.18) {
  const mx = (ox + tx) / 2, my = (oy + ty) / 2;
  const dx = tx - ox, dy = ty - oy;
  const len = Math.hypot(dx, dy);
  const nx = -dy / len, ny = dx / len;               // unit perpendicular
  const k = len * sag * (Math.random() < 0.5 ? 1 : -1);
  return `M ${ox},${oy} Q ${mx + nx*k},${my + ny*k} ${tx},${ty}`;
}
```

Draw it on with `stroke-dashoffset` — the classic "draws itself" technique:

```ts
const line = mk('path', { d: webPath(ox, oy, x, y), stroke: 'var(--fg)', 'stroke-width': 2, fill: 'none' });
svgRoot.append(line);
const L = line.getTotalLength();
gsap.fromTo(line,
  { strokeDasharray: L, strokeDashoffset: L },
  { strokeDashoffset: 0, duration: 0.12, ease: 'power2.in',
    onComplete: () => gsap.to(line, {
      strokeDashoffset: -L, opacity: 0, duration: 0.14,   // recoil: keep going past 0
      onComplete: () => line.remove(),
    })
  });
```

> GSAP's `DrawSVGPlugin` does this more expressively and is now free — but raw dash-offset has
> zero dependencies and is three lines. Use the plugin when you need partial ranges
> (`drawSVG: "20% 80%"`), which the reticles and speed lines do want.

### 4.2 The splat

Author **one** SVG `<symbol>` — 8 radial spokes plus 4 concentric catenary arcs — and `<use>`
it. 2 KB, recolours from `currentColor`, works at any scale.

```html
<symbol id="web" viewBox="-50 -50 100 100">
  <g stroke="currentColor" stroke-width="1.6" fill="none">
    <!-- 8 spokes -->
    <path d="M0,0 L0,-46 M0,0 L32,-32 M0,0 L46,0 M0,0 L32,32
             M0,0 L0,46 M0,0 L-32,32 M0,0 L-46,0 M0,0 L-32,-32"/>
    <!-- concentric rings, each segment sagging inward between spokes -->
    <path d="M0,-12 Q8,-8 8.5,-8.5 Q12,-8 12,0 …Z" opacity=".9"/>
    <!-- …3 more rings at r=22, 32, 44 -->
  </g>
</symbol>
```

```ts
const use = mk('use', { href: '#web', x: -50, y: -50, width: 100, height: 100 });
const g = mk('g', { transform: `translate(${x},${y}) rotate(${Math.random()*360}) scale(0)` });
g.append(use); svgRoot.append(g);
gsap.timeline({ onComplete: () => g.remove() })
  .to(g, { scale: 1, duration: 0.42, ease: 'back.out(2.2)', transformOrigin: '50% 50%' })
  .to(g, { opacity: 0, duration: 0.3 }, '+=0.25');
```

### 4.3 The silk trail

Keep a ring buffer of the last ~20 pointer positions and stroke a tapering polyline. Drop
points older than ~180 ms.

```ts
trail.push({ x, y, t: performance.now() });
if (trail.length > 20) trail.shift();
// in tick():
const now = performance.now();
for (let i = 1; i < trail.length; i++) {
  const age = (now - trail[i].t) / 180;
  if (age > 1) continue;
  ctx.globalAlpha = 1 - age;
  ctx.lineWidth = (1 - age) * 3;
  ctx.beginPath(); ctx.moveTo(trail[i-1].x, trail[i-1].y); ctx.lineTo(trail[i].x, trail[i].y); ctx.stroke();
}
```

---

## 5 · Iron Man: rings and flashes without touching layout

Rings are pure CSS transforms on a fixed-position div — **no layout, no paint of new geometry**,
just compositing.

```css
.fx-ring {
  position: fixed; left: 0; top: 0; width: 12px; height: 12px;
  border: var(--w) solid var(--c); border-radius: 50%;
  transform: translate(-50%, -50%) scale(0);
  will-change: transform, opacity;
}
```

```ts
const ring = div('fx-ring');
ring.style.setProperty('--c', layer.color);
ring.style.setProperty('--w', layer.width + 'px');
gsap.set(ring, { x, y });
gsap.to(ring, {
  scale: layer.toScale, opacity: 0,
  duration: layer.dur / 1000, delay: (layer.delay ?? 0) / 1000,
  ease: layer.ease, onComplete: () => ring.remove(),
});
```

The core flash is a radial gradient with `mix-blend-mode: screen` — it brightens whatever is
under it instead of drawing an opaque white blob:

```css
.fx-flash {
  position: fixed; width: 60px; aspect-ratio: 1;
  background: radial-gradient(circle, #fff 0%, var(--accent) 40%, transparent 70%);
  mix-blend-mode: screen; filter: blur(6px);
  transform: translate(-50%, -50%) scale(0);
}
```

### The text-scramble hover

```ts
const GLYPHS = '█▓▒░<>/\\[]{}#@$%&*01';
function scramble(el: HTMLElement, dur = 220) {
  const target = el.dataset.text ?? el.textContent!;
  const start = performance.now();
  const step = () => {
    const p = Math.min(1, (performance.now() - start) / dur);
    const settled = Math.floor(target.length * p);          // settles left→right
    el.textContent = target.slice(0, settled) +
      [...target.slice(settled)].map(c => c === ' ' ? ' ' : GLYPHS[(Math.random()*GLYPHS.length)|0]).join('');
    if (p < 1) requestAnimationFrame(step); else el.textContent = target;
  };
  step();
}
```

Store the original in `data-text` up front, or a fast re-hover will latch onto scrambled text.

---

## 6 · DBZ: charge, shake and the impact frame

### 6.1 Charge-and-release

```ts
let chargeStart = 0, osc: OscillatorNode | null = null;

onPointerDown = (e) => {
  chargeStart = performance.now();
  osc = audio.startChargeTone();                 // synthesized, pitch tracks charge
  gsap.to('#ki-orb', { scale: 2.4, duration: 1.2, ease: 'power1.in' });
  auraConvergeOn(e.clientX, e.clientY);
};

onPointerUp = (e) => {
  const t = Math.min(1, (performance.now() - chargeStart) / 1200);   // 0..1
  audio.stopChargeTone(osc);
  audio.play('blast', { rate: 1.2 - t * 0.4, volume: 0.4 + t * 0.6 });
  fire(kiBlast, e.clientX, e.clientY, t);        // charge scales every layer
  gsap.to('#ki-orb', { scale: 1, duration: 0.3, ease: 'back.out(3)' });
};
```

Every magnitude in the recipe multiplies by `t`: ring scale, particle count, shake amplitude,
line count, sound volume and pitch. One scalar drives the whole escalation.

### 6.2 Speed lines

```ts
function speedLines(x: number, y: number, n: number) {
  for (let i = 0; i < n; i++) {
    const a = (i / n) * 6.283 + Math.random() * 0.3;
    const r0 = 30 + Math.random() * 20;
    const r1 = r0 + 60 + Math.random() * 180;
    const line = mk('line', {
      x1: x + Math.cos(a)*r0, y1: y + Math.sin(a)*r0,
      x2: x + Math.cos(a)*r1, y2: y + Math.sin(a)*r1,
      stroke: 'var(--accent)', 'stroke-width': 1 + Math.random()*2.5,
      'stroke-linecap': 'round',
    });
    svgRoot.append(line);
    const L = Math.hypot(line.x2.baseVal.value - line.x1.baseVal.value,
                         line.y2.baseVal.value - line.y1.baseVal.value);
    gsap.fromTo(line, { strokeDasharray: L, strokeDashoffset: L, opacity: 1 },
      { strokeDashoffset: 0, duration: 0.18, delay: i * 0.006, ease: 'power2.out',
        onComplete: () => gsap.to(line, { opacity: 0, duration: 0.18, onComplete: () => line.remove() }) });
  }
}
```

### 6.3 Screen shake — on a wrapper, never on `body`

Shaking `body` or `html` can trigger scrollbars, reflow the whole page and fight scroll
anchoring. Shake a dedicated `#shake-root` that wraps the content instead.

```ts
function shake(amp: number, dur = 180) {
  if (reducedMotion()) return;
  const el = document.getElementById('shake-root')!;
  const frames = Math.round(dur / 16.67);
  const kf = Array.from({ length: frames }, (_, i) => {
    const decay = 1 - i / frames;                       // damped
    return { x: (Math.random()*2-1) * amp * decay, y: (Math.random()*2-1) * amp * decay };
  });
  kf.push({ x: 0, y: 0 });
  gsap.to(el, { keyframes: kf, duration: dur/1000, ease: 'none' });
}
```

### 6.4 The impact frame — and the rate limiter

**This is the most safety-critical code in the project.** WCAG 2.3.1 caps flashing at three per
second; rapid clicking would otherwise blow straight through that.

```ts
let lastFlash = 0;
export function impactFrame(ms = 33) {
  if (reducedMotion()) return;
  const now = performance.now();
  if (now - lastFlash < 400) return;                    // hard global cap: ≤2.5/s
  lastFlash = now;
  const el = document.getElementById('fx-impact')!;
  el.style.opacity = '0.85';                            // never a full 1.0
  setTimeout(() => { el.style.opacity = '0'; }, ms);
}
```

```css
#fx-impact {
  position: fixed; inset: 0; background: #fff;
  mix-blend-mode: difference; opacity: 0;
  transition: opacity 60ms linear; pointer-events: none;
}
```

Apply the same limiter to the lightning crackle and the Iron Man bloom lift. One shared
limiter for all full-screen brightness changes, site-wide.

### 6.5 Flickering aura outline

Three stacked shadows at different blurs, each flickering on its own offset interval. Don't
animate `box-shadow` on a timeline — swap between two CSS classes on a ~80 ms interval, which
is cheaper and looks more electrical.

```css
.aura { box-shadow: 0 0 6px var(--accent), 0 0 18px var(--glow), 0 0 40px var(--glow); }
.aura.f2 { box-shadow: 0 0 9px var(--accent), 0 0 14px var(--glow), 0 0 52px var(--glow); }
```

---

## 7 · One Piece: procedural ink and rough edges

### 7.1 Generating an organic blob

12 points around a circle with randomized radii, closed with a smooth bezier chain:

```ts
function inkBlob(r = 40, pts = 12, jitter = 0.35) {
  const P = Array.from({ length: pts }, (_, i) => {
    const a = (i / pts) * 6.283;
    const rr = r * (1 - jitter + Math.random() * jitter * 2);
    return [Math.cos(a) * rr, Math.sin(a) * rr] as const;
  });
  let d = `M ${P[0][0]},${P[0][1]}`;
  for (let i = 0; i < pts; i++) {
    const c = P[(i + 1) % pts], n = P[(i + 2) % pts];
    d += ` Q ${c[0]},${c[1]} ${(c[0]+n[0])/2},${(c[1]+n[1])/2}`;   // midpoint smoothing
  }
  return d + ' Z';
}
```

### 7.2 The rough deckled edge

`feColorMatrix` is cheap; `feTurbulence` is expensive. So: use turbulence, but **animate it for
400 ms maximum, one node at a time, then freeze.**

```html
<filter id="inkRough" x="-30%" y="-30%" width="160%" height="160%" filterUnits="objectBoundingBox">
  <feTurbulence type="fractalNoise" baseFrequency="0.05" numOctaves="2" seed="3" result="n"/>
  <feDisplacementMap in="SourceGraphic" in2="n" scale="6" xChannelSelector="R" yChannelSelector="G"/>
</filter>
```

```ts
gsap.timeline({ onComplete: () => blob.style.filter = 'none' })   // freeze after the entrance
  .to(blob, { scale: 1, duration: 0.5, ease: 'elastic.out(1,.6)' })
  .to(turb, { attr: { baseFrequency: 0.09 }, duration: 0.4 }, 0)
  .to(blob, { opacity: 0, duration: 0.4 }, '+=0.6');
```

The **parchment grain** uses the same filter family but must be **baked**: render it once to a
static tile (or ship a pre-generated PNG/inline data URI) and repeat it. Never leave a live
`feTurbulence` covering the full viewport.

### 7.3 The Gum-Gum stretch button

The most theme-appropriate interaction in the project, and nearly free:

```ts
let dragging = false, origin = { x: 0, y: 0 };

el.addEventListener('pointerdown', e => {
  dragging = true; origin = { x: e.clientX, y: e.clientY };
  el.setPointerCapture(e.pointerId);
  audio.startStretchTone();
});

el.addEventListener('pointermove', e => {
  if (!dragging) return;
  const dx = e.clientX - origin.x, dy = e.clientY - origin.y;
  const d = Math.min(Math.hypot(dx, dy), 120);
  gsap.set(el, {
    x: dx * 0.3, y: dy * 0.3,
    scaleX: 1 + d * 0.002 * Math.abs(dx) / (Math.abs(dx) + Math.abs(dy) + 1),
    scaleY: 1 + d * 0.002 * Math.abs(dy) / (Math.abs(dx) + Math.abs(dy) + 1),
  });
  audio.setStretchPitch(d / 120);                   // pitch = distance, live
});

el.addEventListener('pointerup', () => {
  dragging = false;
  audio.releaseStretch();                            // boing
  gsap.to(el, { x: 0, y: 0, scaleX: 1, scaleY: 1, duration: 0.8, ease: 'elastic.out(1, .3)' });
});
```

---

## 8 · Minions: springs, squash and googly eyes

### 8.1 Volume-preserving squash and stretch

The rule that separates real cartoon animation from "the box got bigger": **when it stretches
on one axis it must compress on the other.**

```tsx
<motion.button
  whileTap={{ scaleX: [1, 1.25, 0.9, 1], scaleY: [1, 0.8, 1.11, 1] }}
  transition={{ type: 'spring', stiffness: 400, damping: 10, mass: 0.8 }}
/>
```

Note each scaleY keyframe is `1 / scaleX`: `1/1.25 = 0.8`, `1/0.9 ≈ 1.11`. Low damping (10) is
what produces the overshoot wobble; raise it and the theme dies.

### 8.2 Googly eyes

Four lines, and it's the detail people will remember:

```ts
gsap.ticker.add(() => {
  const { x: mx, y: my } = pos.current;
  for (const eye of eyes) {                         // rects cached, refreshed on scroll
    const { cx, cy, r } = eye;
    const a = Math.atan2(my - cy, mx - cx);
    const d = Math.min(r * 0.42, Math.hypot(mx - cx, my - cy) * 0.1);
    gsap.set(eye.pupil, { x: Math.cos(a) * d, y: Math.sin(a) * d });
  }
});
```

Clamp `d` to ~42% of the eye radius so the pupil never escapes the sclera.

### 8.3 Banana confetti

`canvas-confetti` supports custom shapes from an SVG path or from text — so flying bananas cost
essentially nothing:

```ts
import confetti from 'canvas-confetti';
const banana = confetti.shapeFromText({ text: '🍌', scalar: 2 });
const blob   = confetti.shapeFromPath({ path: 'M12 2c…' });   // your own SVG path

confetti({
  particleCount: 40, spread: 70, startVelocity: 35, scalar: 1.4, gravity: 1.1,
  shapes: [banana, blob, 'circle'],
  colors: ['#FFD100', '#2E5FA3', '#7B2D8E', '#FFFFFF'],
  origin: { x: x / innerWidth, y: y / innerHeight },
  disableForReducedMotion: true,                    // built in — use it
});
```

### 8.4 Idle wobble, desynchronized

```css
@keyframes wob { 0%,100% { rotate: -2deg } 50% { rotate: 2deg } }
.wobble { animation: wob 3s ease-in-out infinite; animation-delay: var(--d); }
```

Set `--d` to a random negative value per element (`--d: -1.7s`) so they start mid-cycle and
never move in lockstep. A negative delay starts the animation already in progress — that's the
trick.

---

## 9 · The audio manager

```ts
class AudioManager {
  private sfx = new Map<ThemeId, Howl>();
  private music = new Map<ThemeId, Howl>();
  private unlocked = false;
  private recent = new Map<string, number>();
  private live = 0;

  async unlock() {                                   // MUST be inside a user gesture
    if (this.unlocked) return;
    await Howler.ctx?.resume();
    this.unlocked = true;
    if (!store.muted) this.playMusic(store.theme);
  }

  loadTheme(id: ThemeId, sprite: Record<string, [number, number]>) {
    this.sfx.set(id, new Howl({
      src: [`/audio/${id}.webm`, `/audio/${id}.mp3`],   // Opus first, MP3 fallback
      sprite,
    }));
    this.music.set(id, new Howl({
      src: [`/audio/${id}-music.webm`, `/audio/${id}-music.mp3`],
      loop: true, volume: 0, html5: true,               // stream long tracks
    }));
  }

  play(name: string, { rate = 1, volume = 1 } = {}) {
    if (!this.unlocked || store.muted) return;
    const now = performance.now();
    if (now - (this.recent.get(name) ?? 0) < 40) return;   // debounce identical
    if (this.live > 6) return;                             // concurrency cap
    this.recent.set(name, now);
    const h = this.sfx.get(store.theme); if (!h) return;
    const id = h.play(name);
    h.rate(rate * (0.92 + Math.random() * 0.16), id);      // ← anti-machine-gun
    h.volume(volume * store.sfxVolume, id);
    this.live++; h.once('end', () => this.live--, id);
    this.duckMusic();
  }

  crossfade(from: ThemeId, to: ThemeId, ms = 600) {
    const a = this.music.get(from), b = this.music.get(to);
    a?.fade(a.volume(), 0, ms);
    if (b) { b.volume(0); b.play(); b.fade(0, store.musicVolume, ms); }
    setTimeout(() => a?.pause(), ms + 20);                 // pause, so it resumes in place
  }

  private duckMusic() {
    const m = this.music.get(store.theme); if (!m) return;
    m.fade(m.volume(), store.musicVolume * 0.75, 90);
    setTimeout(() => m.fade(m.volume(), store.musicVolume, 260), 150);
  }
}
```

### Synthesized effects

```ts
function repulsor(ctx: AudioContext, t = ctx.currentTime) {
  const osc = ctx.createOscillator(); osc.type = 'sawtooth';
  const lp  = ctx.createBiquadFilter(); lp.type = 'lowpass';
  const g   = ctx.createGain();
  osc.frequency.setValueAtTime(180, t);
  osc.frequency.exponentialRampToValueAtTime(900, t + 0.12);
  lp.frequency.setValueAtTime(400, t);
  lp.frequency.exponentialRampToValueAtTime(6000, t + 0.12);
  g.gain.setValueAtTime(0.0001, t);
  g.gain.exponentialRampToValueAtTime(0.35, t + 0.02);
  g.gain.exponentialRampToValueAtTime(0.0001, t + 0.34);
  osc.connect(lp).connect(g).connect(ctx.destination);
  osc.start(t); osc.stop(t + 0.36);
}

function boing(ctx: AudioContext, t = ctx.currentTime) {
  const osc = ctx.createOscillator(); osc.type = 'sine';
  const lfo = ctx.createOscillator(); lfo.frequency.value = 14;   // the wobble
  const lfoGain = ctx.createGain(); lfoGain.gain.value = 60;
  const g = ctx.createGain();
  osc.frequency.setValueAtTime(520, t);
  osc.frequency.exponentialRampToValueAtTime(120, t + 0.26);
  lfo.connect(lfoGain).connect(osc.frequency);
  g.gain.setValueAtTime(0.4, t);
  g.gain.exponentialRampToValueAtTime(0.0001, t + 0.3);
  osc.connect(g).connect(ctx.destination);
  osc.start(t); lfo.start(t); osc.stop(t + 0.32); lfo.stop(t + 0.32);
}
```

The **ki charge** is the payoff case for synthesis — a sample can't track a live charge:

```ts
function startCharge(ctx: AudioContext) {
  const osc = ctx.createOscillator(); osc.type = 'sine';
  const g = ctx.createGain(); g.gain.value = 0.0001;
  osc.connect(g).connect(ctx.destination); osc.start();
  g.gain.exponentialRampToValueAtTime(0.25, ctx.currentTime + 0.2);
  return {
    osc, g,
    setCharge: (t: number) => osc.frequency.setTargetAtTime(120 + t * 680, ctx.currentTime, 0.05),
    stop: () => { g.gain.exponentialRampToValueAtTime(0.0001, ctx.currentTime + 0.08);
                  osc.stop(ctx.currentTime + 0.1); },
  };
}
```

---

## 10 · The reduced-motion path

Not an afterthought and not a switch you check in ten places — one gate, checked in the
composer:

```ts
const mq = matchMedia('(prefers-reduced-motion: reduce)');
export const reducedMotion = () => mq.matches || store.forceReducedMotion;

function fireReduced(x: number, y: number) {
  // Keep the FEEDBACK, remove the MOTION.
  const d = div('fx-pulse');
  gsap.set(d, { x, y });
  gsap.fromTo(d, { opacity: 0.5, scale: 1 }, { opacity: 0, duration: 0.15, onComplete: () => d.remove() });
}
```

When reduced motion is on: no shake, no parallax, no impact frame, no ambient loops, no trail,
section entrances become cross-fades, and the cursor stops lagging (`duration: 0` on both
`quickTo` calls) but stays visible.

Expose `store.forceReducedMotion` in the settings panel — many affected people are on a shared
or work machine and have never set the OS flag.

---

## 11 · Suggested project structure

```
src/
  app/                      App.tsx, routes, layout
  themes/
    tokens.css              the semantic contract
    mark-vii/{index.ts,recipe.ts,cursor.tsx,ambient.tsx,tokens.css,sprite.json}
    web-head/…  ki-surge/…  grand-line/…  banana-mode/…
  fx/
    composer.ts             fire(recipe, x, y, charge)
    particles.ts            pooled canvas engine
    svgFX.ts                stamps, lines, projectiles
    domFX.ts                rings, flashes, text pops
    screenFX.ts             shake, impact frame, RATE LIMITER
    reduced.ts
  cursor/
    Cursor.tsx              root + state machine
    useMagnetic.ts
    variants/               one per theme
  audio/
    AudioManager.ts
    synth.ts                repulsor, boing, charge, stretch
  game/
    xp.ts  achievements.ts  konami.ts
    HUD.tsx                 one component, five skins
  store/
    theme.ts  audio.ts  game.ts  prefs.ts
  components/               Section, Card, Button, … (tokens only, never theme-specific)
public/
  audio/{theme}.webm|mp3 + {theme}-music.webm|mp3
  fonts/                    subset, preload active theme only
```

**The rule that keeps this maintainable:** nothing under `components/` may reference a theme by
name. Components consume tokens; themes define tokens and recipes. If a component ever needs an
`if (theme === 'banana-mode')`, that's a missing token.

---

## 12 · Build order, and the one trap to avoid

Build in this order: **tokens → cursor → composer → two themes end-to-end → audio → remaining
three themes → content → gamification → polish.**

The trap: building the portfolio content first and retrofitting the FX afterwards. The token
contract is determined by what the effects and cursor need, so writing the site first means
rewriting the site. Prove the abstraction on `mark-vii` and `web-head` — the most and least
similar pair — before committing to it. If those two both fall out of the same composer with
no special-casing, the remaining three will too.

# Animation Patterns Reference

Use this reference when generating presentations. Match animations to the intended feeling.

## Effect-to-Feeling Guide

| Feeling | Animations | Visual Cues |
|---------|-----------|-------------|
| **Dramatic / Cinematic** | Slow fade-ins (1-1.5s), large scale transitions (0.9 to 1), parallax scrolling | Dark backgrounds, spotlight effects, full-bleed images |
| **Techy / Futuristic** | Neon glow (box-shadow), glitch/scramble text, grid reveals | Particle systems (canvas), grid patterns, monospace accents, cyan/magenta/electric blue |
| **Playful / Friendly** | Bouncy easing (spring physics), floating/bobbing | Rounded corners, pastel/bright colors, hand-drawn elements |
| **Professional / Corporate** | Subtle fast animations (200-300ms), clean slides | Navy/slate/charcoal, precise spacing, data visualization focus |
| **Calm / Minimal** | Very slow subtle motion, gentle fades | High whitespace, muted palette, serif typography, generous padding |
| **Editorial / Magazine** | Staggered text reveals, image-text interplay | Strong type hierarchy, pull quotes, grid-breaking layouts, serif headlines + sans body |

## Entrance Animations

```css
/* Fade + Slide Up (most versatile) */
.reveal {
    opacity: 0;
    transform: translateY(30px);
    transition: opacity 0.6s var(--ease-out-expo),
                transform 0.6s var(--ease-out-expo);
}
.visible .reveal {
    opacity: 1;
    transform: translateY(0);
}

/* Scale In */
.reveal-scale {
    opacity: 0;
    transform: scale(0.9);
    transition: opacity 0.6s, transform 0.6s var(--ease-out-expo);
}

/* Slide from Left */
.reveal-left {
    opacity: 0;
    transform: translateX(-50px);
    transition: opacity 0.6s, transform 0.6s var(--ease-out-expo);
}

/* Blur In */
.reveal-blur {
    opacity: 0;
    filter: blur(10px);
    transition: opacity 0.8s, filter 0.8s var(--ease-out-expo);
}
```

## Background Effects

```css
/* Gradient Mesh — layered radial gradients for depth */
.gradient-bg {
    background:
        radial-gradient(ellipse at 20% 80%, rgba(120, 0, 255, 0.3) 0%, transparent 50%),
        radial-gradient(ellipse at 80% 20%, rgba(0, 255, 200, 0.2) 0%, transparent 50%),
        var(--bg-primary);
}

/* Noise Texture — inline SVG for grain */
.noise-bg {
    background-image: url("data:image/svg+xml,..."); /* Inline SVG noise */
}

/* Grid Pattern — subtle structural lines */
.grid-bg {
    background-image:
        linear-gradient(rgba(255,255,255,0.03) 1px, transparent 1px),
        linear-gradient(90deg, rgba(255,255,255,0.03) 1px, transparent 1px);
    background-size: 50px 50px;
}
```

## Interactive Effects

```javascript
/* 3D Tilt on Hover — adds depth to cards/panels */
class TiltEffect {
    constructor(element) {
        this.element = element;
        this.element.style.transformStyle = 'preserve-3d';
        this.element.style.perspective = '1000px';

        this.element.addEventListener('mousemove', (e) => {
            const rect = this.element.getBoundingClientRect();
            const x = (e.clientX - rect.left) / rect.width - 0.5;
            const y = (e.clientY - rect.top) / rect.height - 0.5;
            this.element.style.transform = `rotateY(${x * 10}deg) rotateX(${-y * 10}deg)`;
        });

        this.element.addEventListener('mouseleave', () => {
            this.element.style.transform = 'rotateY(0) rotateX(0)';
        });
    }
}
```

## Seamless Horizontal Parallax Scroll

Recipe for continuously-rolling patterns (ocean waves, cloud bands, scrolling stripes). Avoids the `background-position: %` no-op trap (see [css-gotchas.md](css-gotchas.md) #11) by using `transform: translateX` on a double-width pseudo-element.

**Pattern:** pseudo-element is `200%` wide, tile is sized to `50%` (so one tile = one container width), and translate shifts by `-50%` = one full tile. Because the SVG/image tiles via `repeat-x`, the loop is visually seamless.

```css
.container {
    position: relative;
    overflow: hidden;
}

.container::before {
    content: '';
    position: absolute;
    top: 0; bottom: 0; left: 0;
    width: 200%;                         /* 2× container width */
    background-image: url("data:image/svg+xml;utf8,...");
    background-size: 50% 100%;           /* 1 tile = 1 container */
    background-repeat: repeat-x;
    opacity: 0.7;
    pointer-events: none;
    z-index: 0;
    animation: seamlessScroll 30s linear infinite;
    will-change: transform;
}

/* Direction reverses with sign of translateX:
   -50% → 0  == left-to-right motion (tile appears to roll rightward)
    0 → -50% == right-to-left motion (tile appears to roll leftward) */
@keyframes seamlessScroll {
    from { transform: translateX(-50%); }
    to   { transform: translateX(0);    }
}
```

### Two-layer parallax + tidal easing

For "liquid" or "atmospheric" motion (waves, clouds), stack two layers moving at different speeds and add ease-in-out for natural acceleration:

```css
.container::before {                    /* far layer — slower, quieter */
    width: 200%;
    background-image: url(far-tile.svg);
    background-size: 50% 100%;
    background-repeat: repeat-x;
    opacity: 0.7;
    animation: scrollFar 48s cubic-bezier(0.45, 0, 0.55, 1) infinite;
}
.container::after {                     /* near layer — faster, more present */
    width: 200%;
    background-image: url(near-tile.svg);
    background-size: 50% 100%;
    background-repeat: repeat-x;
    opacity: 0.6;
    animation: scrollNear 32s cubic-bezier(0.5, 0, 0.5, 1) infinite;
}
```

Pick durations that don't share a clean divisor (e.g., 48s + 32s, not 30s + 60s) — the composite motion never repeats on-beat, which reads as organic.

### Compound keyframes (drift + bob in one animation)

When you want horizontal drift AND vertical bob on the same element, combine them in a single keyframe rather than declaring two animations. Two `animation:` entries both touching `transform` conflict — only the last wins (see [css-gotchas.md](css-gotchas.md) #13).

```css
@keyframes oceanLayer {
    0%   { transform: translate3d(-50%,  0px, 0); }
    25%  { transform: translate3d(-37%,  -4px, 0); }
    50%  { transform: translate3d(-25%,  0px, 0); }
    75%  { transform: translate3d(-13%,  -4px, 0); }
    100% { transform: translate3d(  0%,  0px, 0); }
}
```

Endpoints (`0%` and `100%`) must have identical Y values so the loop doesn't snap.

### SVG tileability

For `repeat-x` to join cleanly at tile boundaries:

- Every path's Y value at `x=0` must equal its Y value at `x=viewBoxWidth`
- Use `preserveAspectRatio="none"` if you want the tile to stretch to fit
- Use `preserveAspectRatio="xMidYMax slice"` if you want the pattern anchored to the bottom regardless of aspect ratio

Tile edges that don't match create visible vertical seams as the scroll cycles. Test by viewing the SVG alone in a browser and checking that pasting two copies side-by-side shows no discontinuity.

---

## Rotating card tints via `:nth-child`

Break up monochromatic card grids without HTML edits:

```css
/* Four HINOMC-palette tints cycle through card positions.
   Each card gets a subtle background + matching top-border accent.
   :not(.featured) preserves emphasis on dominant cards. */
.card:nth-child(4n+1):not(.featured) {
    background-color: rgba(0, 120, 144, 0.05);     /* teal */
    border-top: 3px solid rgba(0, 120, 144, 0.35);
}
.card:nth-child(4n+2):not(.featured) {
    background-color: rgba(240, 144, 48, 0.045);   /* amber */
    border-top: 3px solid rgba(240, 144, 48, 0.35);
}
.card:nth-child(4n+3):not(.featured) {
    background-color: rgba(192, 72, 48, 0.04);     /* rose */
    border-top: 3px solid rgba(192, 72, 48, 0.35);
}
.card:nth-child(4n+4):not(.featured) {
    background-color: rgba(100, 140, 100, 0.045);  /* sage */
    border-top: 3px solid rgba(100, 140, 100, 0.35);
}
```

**Why it works:**

- Zero HTML edits — every card in every `.g-2/3/4/6` grid gets a color automatically
- `:not(.featured)` lets emphasis cards keep their prominent treatment
- Alpha stays ≤6% so text readability is never impacted
- Top-border accent at 35% gives each card a confident accent stripe without heavy background fill

**Critical:** use `background-color` not `background` shorthand — otherwise you'll wipe any hover `background-image` layers applied earlier (see [css-gotchas.md](css-gotchas.md) #12).

Pair with a corner-arc pseudo-element on dense grids (`g-2x3`, `g-5x2`, `g-6-items`) for a decorative stroke:

```css
.g-2x3 > .card:not(.featured)::before {
    content: '';
    position: absolute; top: 6px; right: 6px;
    width: 14px; height: 14px;
    border-top: 1px solid rgba(0, 120, 144, 0.35);
    border-right: 1px solid rgba(0, 120, 144, 0.35);
    border-top-right-radius: 14px;
    z-index: 3;
}
/* override border color per nth-child position to match card tint */
```

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Fonts not loading | Check Fontshare/Google Fonts URL; ensure font names match in CSS |
| Animations not triggering | Verify Intersection Observer is running; check `.visible` class is being added |
| Scroll snap not working | Ensure `scroll-snap-type: y mandatory` on html; each slide needs `scroll-snap-align: start` |
| Mobile issues | Disable heavy effects at 768px breakpoint; test touch events; reduce particle count |
| Performance issues | Use `will-change` sparingly; prefer `transform`/`opacity` animations; throttle scroll handlers |
| `background-position` animation not moving | See [css-gotchas.md](css-gotchas.md) #11 — switch to transform-based seamless scroll |
| Two animations on `transform` conflict | Combine into one compound keyframe (see #13 + seamless-parallax recipe above) |

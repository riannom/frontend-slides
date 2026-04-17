# Brand Integration Recipe

How to weave a client logo + wordmark into a deck so it feels authored rather than pasted on. Three placement tiers, tuned for subtle-but-visible presence.

---

## 1. Prepare the logo

Resize to 256×256 (square) or ~400×160 (wordmark logos). If the logo ships with a solid white background, make near-white pixels transparent so it reads on any surface (cream paper, dark slate, tinted gradient).

```python
from PIL import Image

def prepare_brand_mark(src, dst, size=256):
    img = Image.open(src).convert('RGBA')
    img.thumbnail((size, size), Image.LANCZOS)
    # Fade near-white pixels to transparent; preserves antialiased edges
    px = img.load()
    w, h = img.size
    for y in range(h):
        for x in range(w):
            r, g, b, a = px[x, y]
            if r > 240 and g > 240 and b > 240:
                lightness = (r + g + b) / 3
                t = max(0.0, min(1.0, (lightness - 240) / 10))
                px[x, y] = (r, g, b, int(a * (1 - t)))
    img.save(dst, 'PNG', optimize=True)
```

Install with: `pip install Pillow`

---

## 2. Embed as a CSS variable (once)

Base64-encode the prepared PNG and store it once in `:root` so every branded placement pulls from the same source:

```css
:root {
    --brand-logo: url("data:image/png;base64,iVBORw0KGgoAAAA...");
}
```

This keeps the deck a single self-contained file, and every usage is a one-liner:

```css
background: var(--brand-logo) center/contain no-repeat;
```

---

## 3. Three placement tiers

### Tier A — Chrome mark (every slide, subtle)

A small logo fixed in the bottom-left corner. Always present, never blocks content, reads as "seal of authorship."

```html
<div class="brand-unit" aria-hidden="true" title="BRANDNAME">
    <div class="brand-chrome"></div>
    <span class="brand-wordmark">BRANDNAME</span>
</div>
```

```css
.brand-unit {
    position: fixed;
    bottom: clamp(0.8rem, 1.5vh, 1.5rem);
    left: clamp(1rem, 2vw, 2rem);
    display: flex;
    align-items: center;
    gap: clamp(0.45rem, 0.9vw, 0.75rem);
    z-index: 98;
    opacity: 0.68;             /* subtle but visible */
    transition: opacity 0.25s ease;
}
.brand-unit:hover { opacity: 1; }

.brand-chrome {
    width: clamp(22px, 2.8vh, 32px);
    height: clamp(22px, 2.8vh, 32px);
    background: var(--brand-logo) center/contain no-repeat;
    transition: transform 0.25s ease;
}
.brand-unit:hover .brand-chrome { transform: scale(1.08); }

.brand-wordmark {
    font-family: var(--font-mono);
    font-size: clamp(0.62rem, 0.85vw, 0.78rem);
    font-weight: 500;
    letter-spacing: 0.26em;
    color: var(--ink-2);
    text-transform: uppercase;
    white-space: nowrap;
    line-height: 1;
}
```

**Tune opacity per aesthetic:**
- Restrained (consulting/editorial): 0.62
- Modern (SaaS/tech): 0.72
- Dark theme: 0.85 and brighten text color

---

### Tier B — Integrated (title + closing slides)

Replace the decorative geometric circle on title and closing slides with the logo. Give it a soft drop shadow to feel elevated.

```css
.title-slide .shape-circle {
    background: var(--brand-logo) center/contain no-repeat;
    border: none !important;
    opacity: 0.9;
    filter: drop-shadow(0 3px 12px rgba(10, 10, 10, 0.08));
}
.title-slide .shape-circle::before { display: none !important; }
```

---

### Tier C — Animated (section/intermission slides)

Logo sits inside a rotating dashed ring with a slow pulse breathing outward. Gives each section transition a quiet "this is a chapter marker" moment.

```html
<div class="section-logo">
    <div class="section-logo-ring"></div>
    <div class="section-logo-pulse"></div>
    <div class="section-logo-mark"></div>
</div>
```

```css
.section-logo {
    position: relative;
    width: clamp(90px, 16vh, 200px);
    height: clamp(90px, 16vh, 200px);
    border-radius: 50%;
}
.section-logo-mark {
    position: absolute;
    inset: 14%;
    background: var(--brand-logo) center/contain no-repeat;
    border-radius: 50%;
    filter: drop-shadow(0 3px 14px rgba(10, 10, 10, 0.08));
    z-index: 3;
}
.section-logo-ring {
    position: absolute;
    inset: 0;
    border-radius: 50%;
    border: 1.5px dashed rgba(var(--accent-rgb), 0.38);
    animation: sectionRingSpin 18s linear infinite;
    z-index: 1;
}
.section-logo-pulse {
    position: absolute;
    inset: 8%;
    border-radius: 50%;
    border: 1px solid rgba(var(--accent-rgb), 0.5);
    animation: sectionPulse 4.8s cubic-bezier(0.25, 0.9, 0.3, 1) infinite;
    z-index: 2;
    pointer-events: none;
}
@keyframes sectionRingSpin { to { transform: rotate(360deg); } }
@keyframes sectionPulse {
    0%   { transform: scale(0.85); opacity: 0.55; }
    70%  { opacity: 0.12; }
    100% { transform: scale(1.45); opacity: 0; }
}

@media (prefers-reduced-motion: reduce) {
    .section-logo-ring, .section-logo-pulse { animation: none; }
    .section-logo-ring { border-style: dotted; }
}
```

**Pair with a typographic section numeral** on the left for wayfinding:

```css
.section-num {
    font-family: var(--font-display);
    font-size: clamp(2.6rem, 9vw, 6.8rem);
    font-weight: 300;
    color: var(--accent);
    opacity: 0.32;
    letter-spacing: -0.04em;
}
```

Use JS to inject the section numeral + logo structure into every `.intermission` slide's `.decorative` container by extracting `§ NN` from the rule-top.

---

## 4. Accessibility

- `aria-hidden="true"` on decorative brand marks so screen readers skip them
- `title="BRANDNAME"` for a mouse-hover tooltip
- Never rely on the logo alone for meaning — it's decorative
- Respect `prefers-reduced-motion: reduce` by killing ring/pulse animations

---

## 5. File-size budget

- 256×256 base64 PNG ≈ 30–40 KB (negligible)
- 512×512 base64 PNG ≈ 100+ KB (feels the weight; prefer 256 unless logo has fine detail)
- Two variants of the same deck → embed in each; per-file overhead is worth the "one file" portability

For decks with many screenshots or diagrams, switch to a relative `assets/` folder for those images while keeping the logo embedded.

---

## 6. Deck-wide brand palette hook

If the brand has signature colors beyond the accent, expose them as tokens:

```css
:root {
    --brand-primary: #1e3a5f;   /* mapped to --accent */
    --brand-secondary: #c4844f;
    --brand-surface: #faf8f4;   /* maybe mapped to --bg-soft */
}
```

Let the user know these exist — they become obvious tuning knobs if the client later wants to adjust tone.

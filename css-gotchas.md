# CSS Gotchas

Traps that silently break presentations. Each has bitten real decks — check your generated output against this list before declaring done.

---

## 1. `flex: 0` collapses grid content to zero height

**Symptom:** A grid of cards inside a flex column appears completely invisible or clipped. The surrounding content looks fine; the grid cells are simply gone.

**Cause:** The shorthand `flex: 0` expands to `flex-grow: 0; flex-shrink: 1; flex-basis: 0`. The element starts at height 0, can only shrink, never grows. Combined with `min-height: 0` on the grid, it collapses to zero and any content is either clipped by the parent's `overflow: hidden` or never rendered.

**When it bites:** Tight two-column layouts where a grid of small pill-cards sits below a stat or subtitle. Font metric differences (e.g., switching Nunito → Geist or IBM Plex) can push preceding content slightly taller, eating the slack that kept the grid visible.

**Fix:** Use `flex: 0 0 auto` — don't grow, don't shrink, size to content.

```html
<!-- ✗ WRONG — element collapses to 0 height -->
<div class="g-cols g-3" style="flex:0;">...</div>

<!-- ✓ RIGHT — sized by its content -->
<div class="g-cols g-3" style="flex:0 0 auto;">...</div>
```

---

## 2. Pseudo-element backgrounds leak outside the rectangle

**Symptom:** A visible gradient, shine, or colored background shows up OUTSIDE an element you added a `::before` or `::after` to — often covering an adjacent card or spilling across the slide.

**Cause:** `position: absolute` on a pseudo-element attaches to the nearest positioned ancestor. If the parent element lacks `position: relative`, the pseudo attaches to a grandparent and is sized to THAT element. `translateX(-120%)` then moves 120% of the grandparent's width, not the element you intended.

**When it bites:** Adding the shine-sweep hover pattern or any background-layer pseudo-element to cards that weren't originally designed for it. `.dt-leaf`, `.dt-box`, `.diagram .node`, and `.vs-row` are common offenders.

**Fix:** Any element receiving a `::before`/`::after` with absolute positioning MUST have both:

```css
.element {
    position: relative;   /* anchor the pseudo-element */
    overflow: hidden;     /* clip any gradient overshoot */
}
```

**Rule of thumb:** Before appending a hover/shine pseudo-element to an existing class, grep for its declaration. If `position: relative` and `overflow: hidden` aren't both present, add them explicitly.

---

## 3. Negating CSS functions silently fails

Browsers ignore `-clamp()`, `-min()`, `-max()`:

```css
/* ✗ WRONG — silently ignored by browsers, no console error */
right: -clamp(28px, 3.5vw, 44px);
margin-left: -min(10vw, 100px);

/* ✓ RIGHT */
right: calc(-1 * clamp(28px, 3.5vw, 44px));
margin-left: calc(-1 * min(10vw, 100px));
```

---

## 4. Font metrics cascade into layout

Switching display/body fonts can push content past `overflow: hidden` boundaries, making cards disappear or text clip. Wider fonts (serif, humanist sans) render taller; narrower fonts (geometric sans like Nunito) leave slack.

**When offering style swaps on an existing deck:**

- Re-verify tight 2-column content slides at 1280×720 **and** at short viewports (700/600/500 px media queries)
- Watch for content clipping in nested `.col` containers with `overflow: hidden`
- Consider reducing `clamp()` max sizes if switching to a wider font
- Flag slides that combine a stat-big, subtitle, AND a card grid in the same column — these are the highest-risk for overflow

---

## 5. Serif display fonts look broken in uppercase

Serif typefaces (Fraunces, Tiempos, Cormorant, Playfair) have distinctive italics, optical sizing, and character widths that uppercase flattens. Uppercased serif headlines often look amateur — letterspacing fights with the font's inherent rhythm.

**Rule:** When generating or swapping to a serif display preset, set `text-transform: none` on:

- `h1.page-title`, `h2.page-title`
- `.title-slide .main h1`, `.intermission h1`
- `.card h3`, `.step h4`, `.persona h4`
- Any other place uppercase was applied for a sans-serif display font

Keep uppercase only on small mono/sans labels (chips, tags, kickers, rule-top/bottom).

---

## 6. Multi-variant `localStorage` key collisions

When generating multiple style variants of the same deck (e.g., Swiss Modern + Consulting + SaaS), the inline editor's `localStorage` key must be unique per variant. Default is `vibe-coding-slides-edits-v1`; if two HTML files share it, edits made on one variant show up on the other when opened in the same browser (same origin for `file://` URLs, or same domain for hosted).

**Fix:** Suffix the key with the variant name:

```js
// ✗ WRONG — all variants share one storage
this.storageKey = 'slides-edits-v1';

// ✓ RIGHT — per-variant keys
this.storageKey = 'slides-edits-consulting-v1';
this.storageKey = 'slides-edits-saas-v1';
```

Also update the download filename in `exportFile()` to match the variant.

---

## 7. FOUT (Flash of Unstyled Text) with Google/Fontshare

Fonts loaded via `<link>` take a moment to download. Some browsers (Safari, strict Chrome configs) render nothing at all until the font arrives — the slide appears blank on first load. Subsequent refreshes use the browser cache and render instantly.

**Mitigation:**

- Always include a concrete fallback stack: `'Geist', system-ui, sans-serif` — never just `'Geist'`
- Prefer `&display=swap` in the Google Fonts URL (shows fallback while loading). This is the default, but double-check custom URLs
- For critical first-paint fonts, add `<link rel="preload" as="font" crossorigin ...>` hints in `<head>`
- If the user reports "no text on first load," ask them to refresh — don't try to debug the code unless it persists after cache is warm

---

## 8. `overflow: hidden` with scroll-snap: silent clipping

Every `.slide` has `overflow: hidden` to enforce viewport-fitting. This means:

- Content that extends beyond the slide IS rendered but not visible
- It will not trigger any layout warning in the browser
- You won't catch it unless you visually inspect every slide at multiple viewport sizes

**Always verify:** Open the generated deck, step through every slide with arrow keys, resize the window to 1280×720 and 1024×600 at minimum. Look for truncated text or missing cards.

---

## 9. `align-content: stretch` doesn't stretch auto rows

`.g-cols` declares `align-content: stretch`, but auto-sized grid rows (the default when `grid-template-rows` isn't specified) DO NOT stretch to fill extra space. They size to max-content.

**If rows look short:** Specify `grid-template-rows: 1fr 1fr` (or similar) explicitly, or switch the grid from `display: grid` to `display: flex` with `flex: 1` on children.

---

## 10. `data:` URI base64 images bloat the HTML but preserve portability

Embedding a logo as a base64 `data:` URI keeps the deck a single self-contained file (good), but inflates file size (a 256×256 PNG is ~30–40 KB; the base64 encoding is ~33% larger). For decks with >3 large images, prefer a local `assets/` folder and relative paths.

**Rule of thumb:**

- Logo + favicon + ≤2 small icons → embed as base64
- Screenshots, diagrams, larger images → `assets/` folder with relative paths
- Warn the user: "Deploying the deck will need to include the `assets/` folder too"

---

## 11. `background-position: %` is a no-op when `background-size: 100% 100%`

**Symptom:** You animate `background-position` from `0` to `-100%` (or any percentage) expecting the background image to scroll, but nothing moves. The animation declares motion and plays silently.

**Cause:** Per spec, percentage-based `background-position` is calculated as `percentage × (container_size − image_size)`. When `background-size: 100% 100%` sets the image to exactly match the container, the difference is zero. Any percentage value × 0 = no movement.

**When it bites:** Animating scrolling backgrounds (ocean waves, parallax patterns, infinite-scroll stripes). Feels like the animation "just isn't working."

**Fix:** Either use an image smaller than the container (e.g., `background-size: 50% 100%` so tiles can actually slide), OR use viewport/pixel units (`background-size: 100vw 100%` with `background-position: 0 → -100vw`), OR animate a `transform: translateX` on a pseudo-element wider than the container. The transform approach is most reliable and GPU-accelerated:

```css
.container { position: relative; overflow: hidden; }
.container::before {
    content: '';
    position: absolute; top: 0; bottom: 0; left: 0;
    width: 200%;                              /* double container */
    background: url(tile.svg) repeat-x;
    background-size: 50% 100%;                /* 1 tile = 1 container */
    animation: scroll 30s linear infinite;
}
@keyframes scroll {
    from { transform: translateX(0); }
    to   { transform: translateX(-50%); }     /* shift one tile width */
}
```

See [animation-patterns.md](animation-patterns.md) for the full seamless-parallax recipe.

---

## 12. `background: rgba(...)` shorthand silently resets `background-image`

**Symptom:** You set `background-image: radial-gradient(...)` on `.card` in one rule, then later add a tint override with `background: rgba(0,120,144,0.05)` elsewhere. The gradient disappears.

**Cause:** The `background` shorthand resets every sub-property it doesn't explicitly mention — including `background-image`, which gets set to `none`. A value like `background: rgba(...)` only sets `background-color`, but the shorthand implicitly clears the other sub-properties.

**When it bites:** When layering a hover effect (like a cursor-tracking radial-gradient on `background-image`) and later adding a color-tint override for the same element. The new tint wins and wipes the gradient even though they seem to target different properties.

**Fix:** Use `background-color` explicitly when you only want to touch the color layer:

```css
/* ✗ WRONG — wipes any background-image set earlier */
.card:nth-child(4n+1) { background: rgba(0, 120, 144, 0.05); }

/* ✓ RIGHT — color only, gradient preserved */
.card:nth-child(4n+1) { background-color: rgba(0, 120, 144, 0.05); }
```

Alternative for hover layers: put the hover effect on a pseudo-element (`::after`) so it's isolated from the element's own background stack entirely.

---

## 13. Multiple animations on the same property — only the last wins

**Symptom:** You declare two animations in the shorthand: `animation: drift 40s infinite, bob 10s infinite alternate`. You expect drift to animate `transform: translateX` and bob to animate `transform: translateY` independently — but only the bob motion is visible; the drift seems stuck.

**Cause:** When multiple animations target the same property (`transform` is a single property, not per-axis), the one declared LATEST in the list wins at every keyframe. There's no additive composition unless you use `@property` with explicit CSS custom properties, which isn't widely supported yet.

**When it bites:** Any time you want "drift horizontally AND bob vertically" on one element, and both effects touch `transform`.

**Fix:** Either:
1. **Combine into one compound keyframe** (simplest, most compatible):
   ```css
   @keyframes drift_and_bob {
       0%   { transform: translate3d(-50%,  0px, 0); }
       25%  { transform: translate3d(-37%,  -4px, 0); }
       50%  { transform: translate3d(-25%,  0px, 0); }
       75%  { transform: translate3d(-13%,  -4px, 0); }
       100% { transform: translate3d(  0%,  0px, 0); }
   }
   ```
2. **Split across two nested elements** — one child does X, its parent does Y.
3. **Use `@property` with custom properties** (modern browsers):
   ```css
   @property --x { syntax: '<percentage>'; inherits: false; initial-value: 0%; }
   @property --y { syntax: '<length>'; inherits: false; initial-value: 0px; }
   .element {
       transform: translate3d(var(--x), var(--y), 0);
       animation: drift-x 40s linear infinite, bob-y 10s ease alternate infinite;
   }
   @keyframes drift-x { to { --x: -50%; } }
   @keyframes bob-y  { to { --y:  -4px; } }
   ```

Option 1 covers 98% of use cases. Use compound keyframes unless you need the axes to cycle at wildly different rates.

---

## Pre-delivery CSS checklist

Before calling a deck done, grep the generated file for these red flags:

| Pattern | Concern |
|---|---|
| `style="flex:0;"` | Collapsing grid (see #1) |
| `::before` or `::after` without matching `position: relative` on parent | Leaking pseudo-elements (see #2) |
| `-clamp(`, `-min(`, `-max(` | Negated CSS functions (see #3) |
| `text-transform: uppercase` + serif `font-family` | Broken serif headlines (see #5) |
| Identical `storageKey` across variants | Editor cross-contamination (see #6) |
| Single-name font-family with no fallback | FOUT risk (see #7) |
| `background-position: -N%` with `background-size: 100% 100%` | No-op animation (see #11) |
| `background: rgba(...)` on element that also has `background-image` elsewhere | Wiped gradient (see #12) |
| Two `animation:` entries both touching `transform` | Conflicting animations (see #13) |

## Page-number audit

When a deck displays the slide number, keep exactly **one** source of truth:

- Fixed corner counter (`.slide-counter`) OR rule-chrome text (`<span>NN / 52</span>`), not both
- A content deck generated naively can end up with 2–3 redundant page-number elements per slide (fixed counter + rule-top right span + rule-bottom left span). Grep for `>\s*\d+\s*/\s*\d+\s*<` across the generated file to catch duplicates
- If the fixed counter is present, remove the `NN / 52` spans from all `.rule-top` and `.rule-bottom` chrome — they're pure visual noise and can collide with fixed brand marks in the corner

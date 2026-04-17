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

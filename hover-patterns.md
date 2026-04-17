# Hover Patterns

Recipes for mouse-over effects that draw the audience's eye where the presenter points. Combines shine, spotlight, lift, and tint into a coherent "wow without noise" treatment.

---

## The "Wow" Recipe (four layers)

When a presenter hovers a card mid-presentation, the audience should instantly know where to look. Four stacked techniques:

1. **Shine sweep** — a diagonal accent-tinted band travels across the element once per hover
2. **Cursor spotlight** — a subtle radial glow follows the mouse inside the element (Linear/Vercel style)
3. **Lift + colored shadow** — element rises 3–4 px with an accent-tinted drop shadow
4. **Tinted background + accent border** — subtle color wash + highlighted edge

---

## Prerequisites

Every target element must have:

```css
.target {
    position: relative;
    overflow: hidden;
}
```

Without these, pseudo-element gradients leak outside the rectangle (see `css-gotchas.md` #2).

And an RGB channel variable for the accent color:

```css
:root { --accent-rgb: 30, 58, 95; }
```

`rgb()` / `rgba()` need channels; a single hex doesn't work for the glow layers.

---

## Layer 1: Shine Sweep (CSS only)

```css
.target::after {
    content: '';
    position: absolute;
    inset: 0;
    background: linear-gradient(115deg,
        transparent 20%,
        rgba(var(--accent-rgb), 0.12) 50%,
        transparent 80%);
    transform: translateX(-120%);
    pointer-events: none;
    z-index: 1;
    transition: transform 0s;          /* instant reset on leave */
    border-radius: inherit;
}

.target:hover::after {
    transform: translateX(120%);
    transition: transform 0.75s cubic-bezier(0.4, 0, 0.2, 1);
}

/* Children sit above the shine layer */
.target > * {
    position: relative;
    z-index: 2;
}
```

**How it works:** Default state has `transition: 0s` so re-entering the element instantly resets the pseudo back to `-120%`. Hover state has a 0.75s transition so the sweep plays once on each mouse-enter.

---

## Layer 2: Cursor Spotlight (JS + CSS)

A radial glow that tracks the cursor's position inside the element. Feels "alive" without being distracting.

```css
.target {
    background-image: radial-gradient(
        circle 220px at var(--mx, 50%) var(--my, 50%),
        rgba(var(--accent-rgb), 0) 0%,
        rgba(var(--accent-rgb), 0) 100%);
    transition: background-image 0s,
                transform 0.3s cubic-bezier(0.2, 0.8, 0.2, 1),
                box-shadow 0.3s cubic-bezier(0.2, 0.8, 0.2, 1),
                border-color 0.3s ease;
}
.target:hover {
    background-image: radial-gradient(
        circle 220px at var(--mx, 50%) var(--my, 50%),
        rgba(var(--accent-rgb), 0.18) 0%,
        rgba(var(--accent-rgb), 0.05) 60%);
}
```

```js
document.querySelectorAll('.target').forEach(el => {
    el.addEventListener('mousemove', (e) => {
        const r = el.getBoundingClientRect();
        const mx = ((e.clientX - r.left) / r.width) * 100;
        const my = ((e.clientY - r.top) / r.height) * 100;
        el.style.setProperty('--mx', mx + '%');
        el.style.setProperty('--my', my + '%');
    });
    el.addEventListener('mouseleave', () => {
        el.style.setProperty('--mx', '50%');
        el.style.setProperty('--my', '50%');
    });
});
```

**Note:** Only apply spotlight to primary cards (main content elements). Skip on row-style elements (table rows, agenda items) — the effect looks weird on elongated thin shapes.

---

## Layer 3: Lift + Glow

```css
.target:hover {
    transform: translateY(-3px);
    border-color: var(--accent);
    box-shadow: 0 10px 28px rgba(var(--accent-rgb), 0.15),
                0 0 0 1px var(--accent);
}
```

The `0 0 0 1px` shadow acts as a "second border" when the original border is thin — gives a crisp accent edge without affecting layout.

---

## Layer 4: Row-style elements

Grid/table rows can't reliably host `::after` pseudo-elements due to row-layout semantics. Use background tint + inset bar instead:

```css
.vs-row:hover {
    background-color: rgba(var(--accent-rgb), 0.06);
}

.data-table tr:hover {
    background-color: rgba(var(--accent-rgb), 0.06) !important;
    box-shadow: inset 3px 0 0 var(--accent);
}

.agenda:hover {
    background-color: rgba(var(--accent-rgb), 0.05);
    transform: translateX(4px);                    /* tiny slide-right */
    border-bottom-color: var(--accent);
}
```

The `translateX(4px)` on agenda rows gives a subtle "page flips forward" hint without using pseudo-elements.

---

## Reduced motion

```css
@media (prefers-reduced-motion: reduce) {
    .target::after { display: none; }
    .target:hover,
    .agenda:hover { transform: none; }
}
```

Opacity and color changes still apply — users who've opted out of motion still get color feedback.

---

## Tuning per style aesthetic

| Aesthetic | Shine alpha | Lift | Glow alpha | Spotlight peak |
|---|---|---|---|---|
| **Restrained** (consulting, editorial) | 0.10 | 3px | 0.12 | 0.14 |
| **Balanced** (default, Swiss Modern) | 0.13 | 3px | 0.15 | 0.16 |
| **Punchy** (SaaS, modern) | 0.16 | 4px | 0.18 | 0.20 |
| **Dark theme** | 0.22 | 3px | 0.25 | 0.28 |

Dark themes need higher alphas because the accent needs to read against a darker field.

---

## Target list

Typical elements to apply the wow-recipe to:

| Element | Shine | Spotlight | Lift | Notes |
|---|---|---|---|---|
| `.card` | ✓ | ✓ | ✓ | Primary cards — full recipe |
| `.step` | ✓ |   | ✓ | Numbered steps |
| `.persona` | ✓ |   | ✓ | Persona cards |
| `.dt-leaf` | ✓ |   | ✓ | Decision tree outcomes |
| `.dt-box` | ✓ |   | ✓ | Decision tree questions |
| `.diagram .node` | ✓ |   |   | Small diagram nodes |
| `.vs-row` |   |   |   | Background tint only |
| `.agenda` |   |   |   | Tint + slide-right |
| `.data-table tr` |   |   |   | Tint + inset bar |

---

## Why this works

The combination creates **three levels of feedback** at different distances:

1. **Peripheral** — the lift + shadow signals "something changed" from across the room
2. **Mid-range** — the shine sweep + tint draws the eye to the specific element
3. **Close-up** — the cursor spotlight confirms "this is where I'm pointing" to anyone looking at the projector

Presenters don't need to point physically — their cursor becomes the pointer. The audience's eyes follow naturally.

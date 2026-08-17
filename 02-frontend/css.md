# 🎨 CSS Architecture & Modern Layout

> CSS is a constraint-solving language with a cascade. Most CSS pain comes from fighting the cascade or from using the wrong layout algorithm for the job.

---

## 📑 Table of Contents

1. [Mental Model](#1-mental-model)
2. [The Cascade](#2-the-cascade)
3. [Specificity and Modern Escape Hatches](#3-specificity)
4. [The Box Model](#4-the-box-model)
5. [Formatting Contexts](#5-formatting-contexts)
6. [Flexbox](#6-flexbox)
7. [Grid](#7-grid)
8. [Positioning and Stacking](#8-positioning-and-stacking)
9. [Responsive Design](#9-responsive-design)
10. [Modern CSS Features](#10-modern-css-features)
11. [Design Tokens and Theming](#11-design-tokens-and-theming)
12. [Architecture Methodologies](#12-architecture)
13. [Animation](#13-animation)
14. [Accessibility in CSS](#14-accessibility)
15. [Interview Section](#15-interview-section)
16. [Cheat Sheet](#16-cheat-sheet)

---

## 1. Mental Model

```
   Every element gets its style from a resolution process:

   ┌────────────────────────────────────────────────────┐
   │ 1. COLLECT   all declarations matching the element │
   ├────────────────────────────────────────────────────┤
   │ 2. CASCADE   sort by origin → layer → specificity  │
   │              → source order. One winner per prop.  │
   ├────────────────────────────────────────────────────┤
   │ 3. INHERIT   unset inheritable props from parent   │
   ├────────────────────────────────────────────────────┤
   │ 4. DEFAULT   remaining props get initial values    │
   ├────────────────────────────────────────────────────┤
   │ 5. COMPUTE   relative units → absolute (em→px)     │
   ├────────────────────────────────────────────────────┤
   │ 6. LAYOUT    formatting context decides geometry   │
   └────────────────────────────────────────────────────┘
```

The two questions that resolve nearly every CSS bug:
1. **Which declaration won, and why?** (cascade)
2. **Which formatting context is this element in?** (layout)

---

## 2. The Cascade

```
   Sorted top to bottom — the first difference decides:

   ┌──────────────────────────────────────────────────────────┐
   │ 1. ORIGIN & IMPORTANCE                                   │
   │    transition declarations              ← highest        │
   │    !important user-agent                                 │
   │    !important user                                       │
   │    !important AUTHOR (your !important)                   │
   │    animation declarations                                │
   │    normal AUTHOR (your normal CSS)                       │
   │    normal user                                           │
   │    normal user-agent                    ← lowest         │
   ├──────────────────────────────────────────────────────────┤
   │ 2. CASCADE LAYERS  @layer reset, base, components, utils │
   │    Later layers win. !important REVERSES layer order.    │
   ├──────────────────────────────────────────────────────────┤
   │ 3. SPECIFICITY     (inline, ids, classes, elements)      │
   ├──────────────────────────────────────────────────────────┤
   │ 4. SOURCE ORDER    last one wins                         │
   └──────────────────────────────────────────────────────────┘
```

⚠️ Note that `!important` **inverts** layer order. Inside `@layer`, an `!important` in an *earlier* layer beats an `!important` in a later one — deliberately, so a reset layer can enforce non-negotiable rules.

### Inheritance

```
   Inherited by default:  color, font-*, line-height, text-align,
                          letter-spacing, visibility, cursor,
                          list-style, white-space, direction

   NOT inherited:         display, position, width/height, margin,
                          padding, border, background, overflow, z-index

   Force it:
     inherit    → take the parent's computed value
     initial    → the spec's initial value (often not what you want)
     unset      → inherit if inheritable, else initial
     revert     → back to the user-agent/user value  ← usually the best reset
     revert-layer → back to the previous cascade layer
```

---

## 3. Specificity

```
   Specificity is a 3-tuple: (ID, CLASS, TYPE)
   Compared left to right — a higher ID count beats ANY number of classes.

   ┌──────────────────────────────┬───────────┐
   │ Selector                     │ (I, C, T) │
   ├──────────────────────────────┼───────────┤
   │ *                            │ (0,0,0)   │
   │ div                          │ (0,0,1)   │
   │ div p                        │ (0,0,2)   │
   │ .card                        │ (0,1,0)   │
   │ .card:hover                  │ (0,2,0)   │  pseudo-CLASS counts
   │ .card::before                │ (0,1,1)   │  pseudo-ELEMENT is type
   │ [type="text"]                │ (0,1,0)   │
   │ #main                        │ (1,0,0)   │
   │ #main .card p                │ (1,1,1)   │
   │ inline style="…"             │ overrides all selectors │
   │ !important                   │ separate origin tier    │
   └──────────────────────────────┴───────────┘

   Special:
   :is(a, #b)     → takes the HIGHEST specificity of its arguments
   :where(a, #b)  → ALWAYS (0,0,0)  ⭐ the modern escape hatch
   :not(.a)       → takes its argument's specificity
   :has(.a)       → takes its argument's specificity
```

```css
/* ❌ Specificity arms race */
#sidebar .widget ul li a.link { color: red; }        /* (1,3,3) */

/* ✅ Zero-specificity defaults that anything can override */
:where(.prose) :where(h1, h2, h3) { margin-block: 1em; }   /* (0,0,0) */

/* ✅ Layers remove the need to think about specificity at all */
@layer reset, base, components, utilities;
@layer components { .btn { padding: 8px; } }
@layer utilities  { .p-0 { padding: 0; } }   /* wins regardless of specificity */
```

🏭 **Modern best practice:** define layers up front, keep selectors flat (one class), use `:where()` for defaults, and never use IDs or `!important` for styling.

---

## 4. The Box Model

```
   ┌─────────────────── margin ────────────────────┐
   │  ┌──────────────── border ─────────────────┐  │
   │  │  ┌────────────── padding ────────────┐  │  │
   │  │  │  ┌──────────── content ────────┐  │  │  │
   │  │  │  │                             │  │  │  │
   │  │  │  └─────────────────────────────┘  │  │  │
   │  │  └───────────────────────────────────┘  │  │
   │  └─────────────────────────────────────────┘  │
   └───────────────────────────────────────────────┘

   box-sizing: content-box  (default)
      width = content only.  Total = width + padding + border

   box-sizing: border-box   ⭐ what you almost always want
      width = content + padding + border.  Total = width
```

```css
*, *::before, *::after { box-sizing: border-box; }
```

### Margin Collapsing

```
   Adjacent VERTICAL margins collapse to the LARGER of the two.
   Horizontal margins never collapse.

   ┌──────────┐  margin-bottom: 30px
   │    A     │  ┐
   └──────────┘  ├── result: 30px gap, NOT 50px
   ┌──────────┐  ┘
   │    B     │  margin-top: 20px
   └──────────┘

   Three cases:
   1. Adjacent siblings
   2. Parent and first/last child (margin "escapes" the parent)
   3. Empty block (its own top and bottom collapse together)

   Prevented by: flex/grid containers, padding or border on the parent,
                 overflow != visible, position: absolute, float,
                 or a new BFC.
```

Using `gap` in flex/grid sidesteps collapsing entirely, which is one reason modern layouts are less fiddly.

---

## 5. Formatting Contexts

The layout algorithm applied to a box's children. Knowing which one you're in explains most "why doesn't this work" moments.

| Context | Trigger | Children laid out |
|---|---|---|
| **BFC** (Block) | `display: block`, root, `overflow != visible`, `display: flow-root` | Vertically, full width, margins collapse |
| **IFC** (Inline) | Inline-level content | Horizontally in line boxes |
| **FFC** (Flex) | `display: flex` | Along one axis |
| **GFC** (Grid) | `display: grid` | In a 2D grid |
| **TFC** (Table) | `display: table` | Table algorithm |

### The BFC toolkit

A new Block Formatting Context: contains floats, stops margin collapse through it, and doesn't overlap floats.

```css
/* The clean, side-effect-free way to create one */
.container { display: flow-root; }

/* Older triggers (each has side effects) */
.a { overflow: hidden; }        /* clips content */
.b { display: inline-block; }   /* shrinks to fit */
.c { float: left; }
```

```css
/* Classic use: parent collapses to zero height around floated children */
.parent { display: flow-root; }   /* ✅ contains the floats */
```

---

## 6. Flexbox

**One-dimensional.** Content-driven distribution along a single axis.

```
   flex-direction: row  (default)

   main axis ──────────────────────────────────▶
   ┌──────────────────────────────────────────┐  ▲
   │ ┌──────┐  ┌──────┐  ┌──────┐             │  │ cross
   │ │ item │  │ item │  │ item │             │  │ axis
   │ └──────┘  └──────┘  └──────┘             │  │
   └──────────────────────────────────────────┘  ▼
     justify-content controls ──▶ (main)
     align-items controls ▲▼ (cross)
```

```css
.container {
  display: flex;
  flex-direction: row | row-reverse | column | column-reverse;
  flex-wrap: nowrap | wrap | wrap-reverse;
  justify-content: flex-start | center | space-between | space-around | space-evenly;
  align-items: stretch | flex-start | center | baseline | flex-end;
  align-content: /* only matters with wrap — spaces the LINES */;
  gap: 16px;                     /* row-gap column-gap */
}

.item {
  flex: <grow> <shrink> <basis>;
  /* the three you actually need: */
  flex: 1;        /* 1 1 0%    — equal widths, ignore content size */
  flex: auto;     /* 1 1 auto  — grow from content size */
  flex: none;     /* 0 0 auto  — fixed at content size */
  align-self: center;            /* override align-items for one item */
  order: 2;                      /* ⚠️ visual only — breaks tab order */
}
```

### 6.1 The `flex-basis` vs `width` distinction

```
   flex: 1 1 0%      →  all items EQUAL width regardless of content
   flex: 1 1 auto    →  items sized by content, then share extra space
   
   ┌────────┬────────┬────────┐   flex: 1     (0% basis)
   │  short │  much  │ medium │   equal thirds
   │        │ longer │        │
   └────────┴────────┴────────┘

   ┌────┬──────────────┬───────┐  flex: auto   (auto basis)
   │short│ much longer │medium │  proportional to content
   └────┴──────────────┴───────┘
```

### 6.2 The min-width gotcha

```css
/* ⚠️ A flex item's default min-width is `auto`, meaning it refuses
       to shrink below its content — long text or a wide child overflows */
.item { min-width: 0; }        /* ✅ allows shrinking */
.item { overflow: hidden; }     /* also works — implies min-width: 0 */
```

This is *the* most common flexbox bug. Any time text overflows a flex container instead of truncating, this is why.

### 6.3 Common patterns

```css
/* Perfect centering */
.center { display: flex; place-items: center; min-height: 100dvh; }

/* Sticky footer */
body { min-height: 100dvh; display: flex; flex-direction: column; }
main { flex: 1; }

/* Sidebar that wraps on small screens — no media query */
.holy-grail { display: flex; flex-wrap: wrap; gap: 1rem; }
.sidebar { flex: 1 1 250px; }
.main    { flex: 999 1 60%; }   /* huge grow ratio → takes all space until it can't */

/* Nav with a pushed-right item */
.nav { display: flex; gap: 1rem; }
.nav .right { margin-left: auto; }
```

---

## 7. Grid

**Two-dimensional.** Container-driven placement in rows and columns.

```
        col 1     col 2     col 3
      1─────────2─────────3─────────4   ← column lines
   1  ┌─────────┬─────────┬─────────┐
      │ header  │ header  │ header  │   grid-area: header
   2  ├─────────┼─────────┼─────────┤   ← row lines
      │ sidebar │  main   │  main   │
   3  ├─────────┼─────────┼─────────┤
      │ footer  │ footer  │ footer  │
   4  └─────────┴─────────┴─────────┘
```

```css
.grid {
  display: grid;
  grid-template-columns: 200px 1fr 1fr;
  grid-template-rows: auto 1fr auto;
  grid-template-areas:
    "header  header header"
    "sidebar main   main"
    "footer  footer footer";
  gap: 1rem;
}
.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
```

### 7.1 Sizing functions

```css
grid-template-columns: repeat(3, 1fr);              /* 3 equal */
grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));  /* keeps empty tracks */
grid-template-columns: repeat(auto-fit,  minmax(200px, 1fr));  /* collapses empty tracks */
grid-template-columns: minmax(150px, 300px) 1fr;
grid-template-columns: fit-content(300px) 1fr;
grid-template-columns: subgrid;                     /* inherit parent's tracks */
```

```
   auto-fill vs auto-fit with 2 items in a 1000px container:

   auto-fill: │item│item│    │    │      (empty tracks kept)
   auto-fit:  │  item   │   item   │     (empty tracks collapsed, items grow)
```

### 7.2 The famous one-liner

```css
/* Responsive card grid with NO media queries */
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(min(280px, 100%), 1fr));
  gap: 1.5rem;
}
```

The `min(280px, 100%)` prevents overflow on viewports narrower than 280px.

### 7.3 Placement

```css
.item {
  grid-column: 1 / 3;          /* line 1 to line 3 */
  grid-column: 1 / span 2;     /* same, relative */
  grid-column: span 2;         /* auto placement, 2 wide */
  grid-column: 1 / -1;         /* full width — -1 is the last line */
  grid-row: 2 / 4;
  grid-area: 2 / 1 / 4 / 3;    /* row-start / col-start / row-end / col-end */
}

.container {
  grid-auto-flow: row | column | dense;   /* dense backfills holes */
  grid-auto-rows: minmax(100px, auto);    /* implicit row sizing */
  justify-items / align-items;            /* items within their cell */
  justify-content / align-content;        /* the whole grid within the container */
  place-items: center;                    /* align + justify shorthand */
}
```

### 7.4 Flexbox vs Grid — the decision

```
              Do you need to control BOTH rows and columns?
                             │
                 ┌───────────┴────────────┐
                YES                       NO
                 │                        │
                 ▼                        ▼
               GRID                    FLEXBOX
                 │                        │
      "I define the layout,     "Content determines the layout,
       content flows into it"    I distribute the space"

   Page layouts, card grids,   Navs, toolbars, button rows,
   dashboards, forms with      centering, anything that should
   aligned labels              wrap naturally
```

They compose — a grid cell is commonly a flex container.

---

## 8. Positioning and Stacking

```css
position: static;    /* default; top/left ignored */
position: relative;  /* offset from its normal position; SPACE IS RESERVED */
position: absolute;  /* removed from flow; positioned vs nearest positioned ancestor */
position: fixed;     /* vs the viewport (unless an ancestor has transform/filter!) */
position: sticky;    /* relative until a scroll threshold, then fixed */
```

```css
/* Sticky requires: a threshold AND a scrollable ancestor that isn't
   overflow:hidden, AND the parent must be taller than the sticky element */
.sticky-header { position: sticky; top: 0; z-index: 10; }
```

⚠️ **`position: fixed` breaks inside a transformed ancestor.** Any ancestor with `transform`, `filter`, `perspective`, `backdrop-filter`, `contain: paint`, or `will-change` on those becomes the containing block instead of the viewport. This silently breaks modals inside animated containers.

### 8.1 Stacking Contexts

```
   A stacking context is a self-contained z-index universe.
   A child's z-index NEVER escapes its parent's context.

   Created by:
     • root element
     • position != static WITH a z-index other than auto
     • opacity < 1
     • transform, filter, perspective, clip-path, mask
     • will-change on any of the above
     • isolation: isolate          ⭐ explicit, no side effects
     • flex/grid ITEM with z-index
     • position: fixed / sticky (always)
```

```
   ❌ The classic bug:

   .parent { opacity: 0.99; }      ← creates a stacking context!
     .child { z-index: 9999; }     ← trapped inside; can't beat .sibling
   .sibling { z-index: 1; }        ← wins anyway

   ✅ Fix: remove the context, or move .child out, or raise .parent
```

Within a stacking context, the paint order is:

```
   1. background/borders of the context root
   2. negative z-index children
   3. block-level, non-positioned descendants
   4. floats
   5. inline content
   6. z-index: 0 / auto positioned
   7. positive z-index children
```

🏭 Manage z-index with tokens, never magic numbers:

```css
:root {
  --z-base: 0; --z-dropdown: 100; --z-sticky: 200;
  --z-overlay: 300; --z-modal: 400; --z-toast: 500;
}
```

---

## 9. Responsive Design

### 9.1 Units

| Unit | Relative to | Use |
|---|---|---|
| `px` | Absolute | Borders, tiny fixed details |
| `rem` | Root font-size | **Default for spacing/type** — respects user settings |
| `em` | Own/parent font-size | Component-relative spacing |
| `%` | Parent dimension | Fluid widths |
| `vw`/`vh` | Viewport | Full-screen sections |
| `dvh`/`svh`/`lvh` | Dynamic/small/large viewport | ⭐ Mobile — accounts for the collapsing URL bar |
| `ch` / `ex` | Character width / x-height | Line length (`max-width: 65ch`) |
| `fr` | Grid free space | Grid tracks |
| `cqw`/`cqi`/`cqb` | Container query units | Container-relative sizing |

⚠️ Never set a root font-size in `px` and never disable zoom. Users who set a larger default font size must be respected — that's an accessibility requirement, not a preference.

### 9.2 Fluid Typography

```css
/* clamp(MIN, PREFERRED, MAX) — scales smoothly, no media queries, and
   because the preferred value includes a rem term, zoom still works */
h1 { font-size: clamp(1.75rem, 1.2rem + 2.5vw, 3.5rem); }
.container { width: min(100% - 2rem, 72rem); margin-inline: auto; }
.section { padding-block: clamp(2rem, 6vw, 6rem); }
```

### 9.3 Container Queries

The most important CSS addition in years: components respond to *their own* size, not the viewport's.

```css
.card-wrapper { container-type: inline-size; container-name: card; }

@container card (min-width: 400px) {
  .card { display: grid; grid-template-columns: 150px 1fr; }
}

/* Container query units */
.card h2 { font-size: clamp(1rem, 5cqi, 2rem); }   /* 5% of container inline size */
```

This finally makes components truly reusable — the same card works in a sidebar and a full-width hero without knowing where it is.

### 9.4 Media Queries

```css
/* Mobile-first: base styles are mobile, then add complexity */
.grid { display: grid; gap: 1rem; }
@media (min-width: 48rem) { .grid { grid-template-columns: repeat(2, 1fr); } }
@media (min-width: 64rem) { .grid { grid-template-columns: repeat(3, 1fr); } }

/* Range syntax (modern) */
@media (400px <= width <= 700px) { }

/* Preference queries — respect the user */
@media (prefers-reduced-motion: reduce) { *, *::before, *::after {
  animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; } }
@media (prefers-color-scheme: dark) { }
@media (prefers-contrast: more) { }
@media (hover: hover) and (pointer: fine) { .card:hover { transform: scale(1.02); } }
```

Guarding hover effects with `@media (hover: hover)` prevents sticky hover states on touch devices.

---

## 10. Modern CSS Features

```css
/* ── Nesting (native, no preprocessor) ────────────── */
.card {
  padding: 1rem;
  & .title { font-weight: 600; }
  &:hover { background: var(--surface-2); }
  @media (min-width: 40rem) { padding: 2rem; }
}

/* ── :has() — the "parent selector" ───────────────── */
.card:has(img) { grid-template-rows: 200px auto; }
form:has(:invalid) .submit { opacity: 0.5; pointer-events: none; }
body:has(dialog[open]) { overflow: hidden; }        /* scroll lock, no JS */
.label:has(+ input:focus) { color: var(--accent); } /* "previous sibling" */

/* ── :is() / :where() ─────────────────────────────── */
:is(h1, h2, h3) + p { margin-top: 0.5em; }
:where(ul, ol) :where(ul, ol) { margin-block: 0; }   /* zero specificity */

/* ── Logical properties (i18n-ready) ──────────────── */
.box {
  margin-inline: auto;          /* left+right in LTR, flips in RTL */
  padding-block: 1rem;          /* top+bottom */
  border-inline-start: 2px solid;
  inset-inline-start: 0;
}

/* ── Color functions ──────────────────────────────── */
:root {
  --brand: oklch(60% 0.15 250);                     /* perceptually uniform */
  --brand-light: oklch(from var(--brand) 80% c h);  /* relative color */
  --overlay: rgb(0 0 0 / 50%);                      /* modern slash syntax */
}
.mix { background: color-mix(in oklch, var(--brand) 70%, white); }

/* ── @supports ────────────────────────────────────── */
@supports (container-type: inline-size) { /* progressive enhancement */ }
@supports not (display: grid) { /* fallback */ }

/* ── Scroll-driven animation (no JS) ──────────────── */
@keyframes progress { from { scale: 0 1; } to { scale: 1 1; } }
.progress-bar { animation: progress linear; animation-timeline: scroll(root); }

/* ── View transitions ─────────────────────────────── */
@view-transition { navigation: auto; }
.hero { view-transition-name: hero; }

/* ── Anchor positioning ───────────────────────────── */
.tooltip { position: absolute; position-anchor: --btn; inset-area: block-start; }

/* ── Others worth knowing ─────────────────────────── */
.truncate { text-wrap: balance; }        /* balanced headline line lengths */
p { text-wrap: pretty; }                 /* avoids orphans */
dialog::backdrop { background: rgb(0 0 0 / 60%); }
.field { field-sizing: content; }        /* textarea auto-grows */
@starting-style { .modal { opacity: 0; } }  /* entry animations for display:none */
```

---

## 11. Design Tokens and Theming

```css
@layer tokens {
  :root {
    /* ── Primitive tokens: raw values ──────────────── */
    --gray-50:  oklch(98% 0.005 250);
    --gray-900: oklch(20% 0.02 250);
    --blue-500: oklch(60% 0.18 250);

    --space-1: 0.25rem; --space-2: 0.5rem;  --space-3: 0.75rem;
    --space-4: 1rem;    --space-6: 1.5rem;  --space-8: 2rem;

    --text-sm: clamp(0.875rem, 0.85rem + 0.1vw, 0.9375rem);
    --text-base: clamp(1rem, 0.95rem + 0.25vw, 1.125rem);
    --text-xl: clamp(1.5rem, 1.3rem + 1vw, 2rem);

    --radius-sm: 4px; --radius-md: 8px; --radius-full: 9999px;
    --shadow-sm: 0 1px 2px rgb(0 0 0 / 0.05);
    --shadow-lg: 0 10px 25px rgb(0 0 0 / 0.15);

    /* ── Semantic tokens: meaning, mapped to primitives ── */
    --color-bg: var(--gray-50);
    --color-surface: white;
    --color-text: var(--gray-900);
    --color-text-muted: oklch(50% 0.02 250);
    --color-accent: var(--blue-500);
    --color-border: oklch(90% 0.01 250);
  }

  /* Dark mode = remap SEMANTIC tokens only */
  @media (prefers-color-scheme: dark) {
    :root:not([data-theme='light']) {
      --color-bg: var(--gray-900);
      --color-surface: oklch(25% 0.02 250);
      --color-text: var(--gray-50);
      --color-border: oklch(35% 0.02 250);
    }
  }
  :root[data-theme='dark'] { /* same overrides, for the explicit toggle */ }
}
```

**The two-tier rule:** components reference only semantic tokens. Primitives change rarely; semantics are the theming surface. This means adding a new theme touches one block, not every component.

### Typed custom properties

```css
@property --gradient-angle {
  syntax: '<angle>';
  initial-value: 0deg;
  inherits: false;
}
/* Now this ANIMATES — untyped custom properties cannot */
.card { background: linear-gradient(var(--gradient-angle), a, b);
        transition: --gradient-angle 400ms; }
```

---

## 12. Architecture

| Methodology | Idea | Best for |
|---|---|---|
| **BEM** | `.block__element--modifier` | Team consistency without tooling |
| **Utility-first** (Tailwind) | Compose from atomic classes | Fast iteration, no naming debates |
| **CSS Modules** | Locally scoped class names | Component frameworks |
| **CSS-in-JS** | Styles as JS | Dynamic theming (⚠️ runtime cost) |
| **Cascade Layers** | Explicit priority order | Taming third-party CSS |
| **ITCSS** | Specificity increases down the file | Large legacy codebases |

```css
/* Layer architecture — the modern foundation regardless of methodology */
@layer reset, tokens, base, layout, components, utilities, overrides;

@layer reset {
  *, *::before, *::after { box-sizing: border-box; }
  * { margin: 0; }
  body { -webkit-font-smoothing: antialiased; line-height: 1.5; }
  img, picture, video, svg { display: block; max-width: 100%; }
  input, button, textarea, select { font: inherit; color: inherit; }
  p, h1, h2, h3, h4 { overflow-wrap: break-word; }
  :target { scroll-margin-block: 5ex; }
}
```

Because layers are ordered explicitly, a one-class utility in the `utilities` layer beats a five-selector rule in `components` — specificity stops mattering.

---

## 13. Animation

```css
/* Transitions — for state changes */
.btn {
  transition: background-color 200ms ease-out, transform 150ms ease-out;
}
.btn:hover { transform: translateY(-2px); }

/* Keyframes — for sequences */
@keyframes slide-in {
  from { opacity: 0; transform: translateY(20px); }
  to   { opacity: 1; transform: translateY(0); }
}
.panel { animation: slide-in 300ms cubic-bezier(0.16, 1, 0.3, 1) both; }
```

### Performance rules

```
   ✅ ONLY animate: transform, opacity, filter
      → compositor thread, no layout, no paint, 60fps even when JS is busy

   ⚠️ Avoid: width, height, top, left, margin, padding, font-size
      → layout on every frame

   Easing:
      ease-out       → entrances (fast start, gentle stop) ← most UI
      ease-in        → exits
      cubic-bezier(0.16, 1, 0.3, 1)  → "expo out", feels snappy
      linear         → only for continuous motion (spinners, progress)

   Duration:
      100-150ms  micro (hover, focus)
      200-300ms  standard (dropdowns, toggles)
      300-500ms  large (page/modal transitions)
      >500ms     feels sluggish unless it's a deliberate reveal
```

### Always respect motion preferences

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

This is not optional — vestibular disorders make large motion genuinely painful for some users.

---

## 14. Accessibility

```css
/* Visually hidden but available to screen readers */
.sr-only {
  position: absolute; width: 1px; height: 1px;
  padding: 0; margin: -1px; overflow: hidden;
  clip-path: inset(50%); white-space: nowrap; border: 0;
}

/* Focus: never remove it — replace it */
:focus-visible {                        /* keyboard focus only */
  outline: 2px solid var(--color-accent);
  outline-offset: 2px;
}
:focus:not(:focus-visible) { outline: none; }   /* no ring on mouse click */

/* Minimum touch target: 44×44 CSS px */
.icon-btn { min-block-size: 44px; min-inline-size: 44px; }

/* Readable line length */
.prose { max-width: 65ch; line-height: 1.6; }

/* Don't convey meaning by color alone */
.error { color: var(--red); }
.error::before { content: '⚠ '; }
```

**Contrast requirements (WCAG AA):** 4.5:1 for normal text, 3:1 for large text (≥24px, or ≥19px bold) and for UI components/graphical objects.

---

## 15. Interview Section

<details>
<summary><b>Q1. Explain the cascade and specificity.</b></summary>

The cascade decides which declaration wins when several target the same property. It sorts by origin and importance first — user-agent, user, and author styles, each with a normal and an `!important` tier where important reverses the order. Then cascade layers, then specificity, then source order.

Specificity is a three-part tuple counting IDs, classes (plus attributes and pseudo-classes), and element types (plus pseudo-elements). It compares left to right, so one ID beats any number of classes — it's not a base-10 sum.

The modern approach is to avoid the whole competition: use `@layer` to declare priority explicitly, keep selectors to a single class, and use `:where()` for defaults since it contributes zero specificity.
</details>

<details>
<summary><b>Q2. Flexbox vs Grid — how do you choose?</b></summary>

Grid when you're defining the layout structure and content flows into it — you control rows and columns simultaneously. Flexbox when content determines the layout and you're distributing space along one axis.

Practically: page shells, dashboards, card grids, and any form where labels must align across rows are Grid. Navigation bars, toolbars, button groups, and anything that should wrap naturally based on content width are Flexbox.

They compose constantly — a grid cell is very often a flex container.

The other distinction: Grid can overlap items and create explicit named areas; Flexbox can't do either cleanly.
</details>

<details>
<summary><b>Q3. What creates a stacking context, and why does it matter?</b></summary>

`position` with a non-auto `z-index`, `opacity` below 1, `transform`, `filter`, `perspective`, `clip-path`, `mask`, `isolation: isolate`, `will-change` on any of those, and `position: fixed` or `sticky` unconditionally.

It matters because z-index is scoped to the stacking context. A child with `z-index: 9999` inside a context whose root has `z-index: 1` can never paint above a sibling of that root with `z-index: 2`. This is why modals disappear behind things for no apparent reason — usually an ancestor has `opacity: 0.99` or a transform.

`isolation: isolate` is the clean way to create one deliberately, since it has no other visual effect.
</details>

<details>
<summary><b>Q4. What are container queries and why do they matter more than media queries?</b></summary>

Container queries let a component style itself based on the size of its container rather than the viewport. You mark an ancestor with `container-type: inline-size` and then write `@container (min-width: 400px)`.

They matter because media queries couple a component to page context. The same card in a sidebar and in a full-width region needs different layouts, but the viewport is identical — so you end up with variant classes or context-specific overrides that break encapsulation.

With container queries a component is genuinely portable: drop it anywhere and it adapts. Plus container query units (`cqi`, `cqw`) let typography scale with the container too.
</details>

<details>
<summary><b>Q5. Explain margin collapsing.</b></summary>

Adjacent vertical margins combine into one margin equal to the larger of the two, rather than summing. It happens between siblings, between a parent and its first or last child (where the child's margin appears to escape the parent), and within an empty block.

It's prevented by anything that establishes a new formatting context or puts something between the margins: padding or a border on the parent, `overflow` other than visible, `display: flow-root`, flex or grid containers, floats, and absolute positioning.

In modern layouts it mostly stops being an issue because `gap` in flex and grid doesn't collapse — which is a good reason to prefer `gap` over margins for spacing between siblings.
</details>

<details>
<summary><b>Q6. How would you implement dark mode?</b></summary>

Two-tier custom properties. Primitive tokens hold raw values; semantic tokens hold meaning — `--color-surface`, `--color-text`. Components reference only semantic tokens.

Dark mode then remaps the semantic layer in one block, under `@media (prefers-color-scheme: dark)` for the system preference and under a `[data-theme="dark"]` attribute selector for an explicit toggle. Both are needed, because the default "system" setting stamps no attribute at all.

Details that matter: put the attribute on `<html>` and set it in a blocking inline script before first paint to avoid a flash; set `color-scheme: light dark` so form controls and scrollbars follow; and don't just invert — dark surfaces usually need reduced saturation and elevation expressed by lighter surfaces rather than shadows.
</details>

<details>
<summary><b>Q7. What does `:has()` unlock?</b></summary>

It's a parent and sibling selector — the thing CSS lacked for twenty years. `.card:has(img)` styles a card differently when it contains an image. `form:has(:invalid)` disables a submit button without JavaScript. `body:has(dialog[open])` locks scroll when a modal is open.

It also gives previous-sibling selection: `.label:has(+ input:focus)` styles a label based on the input that follows it.

The caveat is that it takes the specificity of its most specific argument, and it's a genuinely expensive selector for browsers to evaluate, so I'd avoid it in very hot, very large DOMs — though in practice modern engines handle it well.
</details>

<details>
<summary><b>Q8. Why animate only `transform` and `opacity`?</b></summary>

Because they're handled on the compositor thread. Changing them skips style, layout, and paint entirely, so the animation runs at 60fps even when the main thread is blocked by JavaScript.

Anything geometric — width, height, top, margin — triggers layout, which forces paint and composite too, all on the main thread, every frame. On a complex page layout alone can exceed the ~10ms budget you have per frame.

You can promote an element with `will-change: transform`, but each composited layer costs GPU memory, so it should be applied just before an animation and removed after, not left on globally.
</details>

<details>
<summary><b>Q9. What are cascade layers and what problem do they solve?</b></summary>

`@layer` lets you declare an explicit priority order for groups of styles. Layer order beats specificity entirely — a single class in a later layer wins over a five-selector rule in an earlier one.

The problem it solves is the specificity arms race, especially with third-party CSS. Previously, overriding a library's `#widget .item a` meant writing something even more specific or reaching for `!important`. Now you put the library in an early layer and your overrides in a later one, and you're done.

The subtlety: `!important` reverses layer order, so an important declaration in your reset layer beats an important one in utilities. That's deliberate, so foundational rules can be non-negotiable.
</details>

<details>
<summary><b>Q10. How do you make CSS maintainable at scale?</b></summary>

Four things. First, cascade layers to make priority explicit rather than emergent. Second, design tokens in two tiers — primitives and semantics — so theming is one file, not a search-and-replace. Third, flat selectors: one class per rule, no IDs, no `!important`, `:where()` for defaults. Fourth, some form of scoping — CSS Modules, or a utility framework, or strict BEM — so a class name can't collide across the codebase.

Beyond mechanics, the discipline that matters most is deleting. CSS is append-only by default because nobody's sure what a rule affects. Tooling that reports unused CSS, plus components that own their styles, is what actually keeps a stylesheet from growing forever.
</details>

---

## 16. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                           CSS — ONE PAGE                             ║
╠══════════════════════════════════════════════════════════════════════╣
║ CASCADE: origin/importance → LAYER → specificity → source order      ║
║   specificity = (ID, CLASS, TYPE), compared left to right            ║
║   :where() = 0 specificity   :is() = highest of its args             ║
║   !important REVERSES layer order                                    ║
╠══════════════════════════════════════════════════════════════════════╣
║ LAYOUT CHOICE                                                        ║
║   Grid    → 2D, you define structure, content flows in               ║
║   Flexbox → 1D, content defines size, you distribute space           ║
║   flex:1 = "1 1 0%" equal | flex:auto = "1 1 auto" content-sized     ║
║   ⚠️ flex items need min-width:0 to shrink below content              ║
╠══════════════════════════════════════════════════════════════════════╣
║ STACKING CONTEXT created by: z-index+position · opacity<1 ·          ║
║   transform · filter · isolation:isolate · fixed/sticky              ║
║   → child z-index NEVER escapes its parent context                   ║
║ position:fixed BREAKS inside a transformed ancestor                   ║
╠══════════════════════════════════════════════════════════════════════╣
║ RESPONSIVE                                                           ║
║   rem for type/space (respects user settings) · dvh not vh on mobile ║
║   clamp(min, preferred+rem, max) for fluid type                      ║
║   @container > @media for components                                 ║
║   repeat(auto-fit, minmax(min(280px,100%), 1fr))  ← the one-liner    ║
╠══════════════════════════════════════════════════════════════════════╣
║ ANIMATE ONLY transform + opacity (compositor thread)                 ║
║   ALWAYS honor prefers-reduced-motion                                ║
╠══════════════════════════════════════════════════════════════════════╣
║ MODERN: :has() · nesting · @layer · logical props · oklch ·          ║
║   color-mix · @property (animatable vars) · subgrid · @starting-style║
╠══════════════════════════════════════════════════════════════════════╣
║ A11Y: :focus-visible never removed · 44px targets · 4.5:1 contrast   ║
║   65ch line length · never color alone                               ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [React](react.md) · [Browser & Performance](browser-performance.md) · [Next.js](nextjs.md)

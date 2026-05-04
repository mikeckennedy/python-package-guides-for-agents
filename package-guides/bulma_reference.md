# Bulma — Comprehensive Reference (v1.0.4)

> Modern, Sass-powered, no-JavaScript CSS framework based on flexbox, CSS Grid, and CSS variables.
> A library of class-based modifiers (`is-*`, `has-*`) layered on top of unstyled HTML elements.
> RTL-friendly via logical properties. Dark-mode and theming built in via CSS custom properties.

This reference is generated from the **Bulma 1.0.4 source** (`bulma.scss`, the `sass/` directory, and the official documentation in `docs/`). Class names, modifiers, breakpoints, and CSS variables match the shipped version exactly.

---

## Table of Contents

- [Installation](#installation)
- [HTML Requirements & Starter Template](#html-requirements--starter-template)
- [Modifier Syntax (`is-*`, `has-*`)](#modifier-syntax-is--has-)
- [Color System](#color-system)
- [Sizes & Typography Scale](#sizes--typography-scale)
- [Responsive Breakpoints](#responsive-breakpoints)
- [Dark Mode & Theming](#dark-mode--theming)
- [CSS Variables](#css-variables)
- [Layout](#layout)
  - [Container](#container)
  - [Section](#section)
  - [Hero](#hero)
  - [Level](#level)
  - [Media object](#media-object)
  - [Footer](#footer)
- [Columns (12-column flexbox grid)](#columns-12-column-flexbox-grid)
- [Grid (CSS Grid)](#grid-css-grid)
- [Elements](#elements)
  - [Block](#block)
  - [Box](#box)
  - [Button](#button)
  - [Content](#content)
  - [Delete](#delete)
  - [Icon / Icon Text](#icon--icon-text)
  - [Image](#image)
  - [Notification](#notification)
  - [Progress](#progress)
  - [Table](#table)
  - [Tag](#tag)
  - [Title / Subtitle](#title--subtitle)
- [Components](#components)
  - [Breadcrumb](#breadcrumb)
  - [Card](#card)
  - [Dropdown](#dropdown)
  - [Menu](#menu)
  - [Message](#message)
  - [Modal](#modal)
  - [Navbar](#navbar)
  - [Pagination](#pagination)
  - [Panel](#panel)
  - [Tabs](#tabs)
- [Form Controls](#form-controls)
  - [Field, Label, Help, Control](#field-label-help-control)
  - [Input, Textarea](#input-textarea)
  - [Select](#select)
  - [Checkbox, Radio](#checkbox-radio)
  - [File](#file)
- [Helpers (utility classes)](#helpers-utility-classes)
  - [Color helpers](#color-helpers)
  - [Spacing helpers](#spacing-helpers)
  - [Typography helpers](#typography-helpers)
  - [Flexbox helpers](#flexbox-helpers)
  - [Visibility / Display helpers](#visibility--display-helpers)
  - [Border / Aspect / Float / Gap / Position / Overflow / Other](#border--aspect--float--gap--position--overflow--other)
- [Skeletons](#skeletons)
- [Sass Customization](#sass-customization)
- [JavaScript Notes (Bulma ships none)](#javascript-notes-bulma-ships-none)
- [Migration Notes (v0 → v1)](#migration-notes-v0--v1)

---

## Installation

Bulma is distributed as plain CSS *and* as Sass source.

### Option 1 — CDN (jsDelivr)

```html
<link rel="stylesheet"
      href="https://cdn.jsdelivr.net/npm/bulma@1.0.4/css/bulma.min.css">
```

Or via `@import` inside your own CSS:

```css
@import "https://cdn.jsdelivr.net/npm/bulma@1.0.4/css/bulma.min.css";
```

> Since v1, the main version works in **RTL** contexts thanks to logical properties — there is no separate `bulma-rtl.min.css` anymore.

### Option 2 — npm (Sass build)

```bash
npm install bulma
```

```scss
@use "bulma/bulma";
```

The package's `main` is `bulma.scss`; `unpkg`/`style` is `css/bulma.min.css`.

### Pre-built variants (in `css/versions/`)

| Variant | What it omits / changes |
|---|---|
| `bulma-no-dark-mode.css` | No `prefers-color-scheme: dark` block |
| `bulma-no-helpers.css` | No `.is-*` / `.has-*` utility classes |
| `bulma-no-helpers-prefixed.css` | No helpers, classes prefixed |
| `bulma-prefixed.css` | All classes prefixed |

---

## HTML Requirements & Starter Template

For Bulma to render correctly your page must be modern + responsive:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>My Bulma App</title>
  <link rel="stylesheet"
        href="https://cdn.jsdelivr.net/npm/bulma@1.0.4/css/bulma.min.css">
</head>
<body>
  <section class="section">
    <div class="container">
      <h1 class="title">Hello world</h1>
      <p class="subtitle">My first <strong>Bulma</strong> page</p>
    </div>
  </section>
</body>
</html>
```

---

## Modifier Syntax (`is-*`, `has-*`)

Bulma is built around modifier classes added next to a base class:

```html
<button class="button">Default</button>
<button class="button is-primary">Colored</button>
<button class="button is-primary is-large is-outlined">All three</button>
<button class="button is-primary is-loading">Loading</button>
<button class="button is-danger" disabled>Disabled (HTML attribute)</button>
```

Conventions used everywhere:

- `is-*` — applies a state, color, size, or style to **the element it's on** (e.g. `is-primary`, `is-small`, `is-active`, `is-rounded`, `is-fullwidth`).
- `has-*` — describes what the element **contains** or has a relationship with (e.g. `has-addons`, `has-icons-left`, `has-text-centered`, `has-background-light`).
- `are-*` — applies a size to **all children** of a container (e.g. `<div class="buttons are-medium">` makes every `.button` inside medium).

Modifiers stack freely. Order does not matter to the CSS, but convention is `base color size style state`.

---

## Color System

### The 6 main colors + neutrals

Defined in `sass/utilities/derived-variables.scss`:

| Name | Default | Sass variable | Use |
|---|---|---|---|
| `primary` | `turquoise` | `$primary` | Brand / main action |
| `link` | `blue` | `$link` | Hyperlinks, focused state |
| `info` | `cyan` | `$info` | Informational |
| `success` | `green` | `$success` | Success state |
| `warning` | `yellow` | `$warning` | Warning state |
| `danger` | `red` | `$danger` | Errors / destructive |
| `light` | `white-ter` | `$light` | Light surface |
| `dark` | `grey-darker` | `$dark` | Dark surface |
| `text` | `grey-dark` | `$text` | Body text |
| `white`, `black` | — | — | Absolutes |

These names appear consistently across components: `is-primary`, `has-text-primary`, `has-background-primary`, `is-primary is-light`, etc.

### Shades (neutral palette)

`black-bis`, `black-ter`, `grey-darker`, `grey-dark`, `grey`, `grey-light`, `grey-lighter`, `white-ter`, `white-bis`. Available as `has-text-grey-light`, `has-background-white-ter`, etc.

### Per-color variants on a single class

Each named color produces these variants:

```
is-{name}                 e.g. is-primary
is-{name}-light           softer background, dark on light
is-{name}-dark
is-{name}-soft            theme-aware tinted
is-{name}-bold            theme-aware strong
has-text-{name}
has-text-{name}-invert
has-text-{name}-on-scheme       always readable on the page background
has-text-{name}-light / -dark
has-text-{name}-soft / -bold
has-background-{name}
has-background-{name}-invert
has-background-{name}-on-scheme
has-background-{name}-light / -dark
has-background-{name}-soft / -bold
```

### 21-step shade palette

Each main color exposes 21 lightness shades (`00`, `05`, `10`, `15`, …, `95`, `100`). Use them like:

```html
<span class="has-text-primary-30">very dark primary</span>
<span class="has-background-info-90">very light info background</span>
<span class="has-text-primary-50-invert">accessible on bg-50</span>
```

Each shade also has an `-invert` companion that auto-picks an accessible foreground.

### Palette helpers (`is-palette-{name}`)

`is-palette-primary` exposes the entire palette as local CSS variables (`--color`, `--color-50`, …, `--h`, `--s`, `--l`) so you can build custom widgets that re-use Bulma's color science.

---

## Sizes & Typography Scale

Seven font sizes (`$size-1` … `$size-7`):

| Class | Size variable | rem |
|---|---|---|
| `is-size-1` | `$size-1` | 3rem |
| `is-size-2` | `$size-2` | 2.5rem |
| `is-size-3` | `$size-3` | 2rem |
| `is-size-4` | `$size-4` | 1.5rem |
| `is-size-5` | `$size-5` | 1.25rem |
| `is-size-6` | `$size-6` | 1rem (body) |
| `is-size-7` | `$size-7` | 0.75rem |

Aliased size names used on components:

| Class | = |
|---|---|
| `is-small` | `$size-7` (0.75rem) |
| `is-normal` | `$size-6` (1rem) |
| `is-medium` | `$size-5` (1.25rem) |
| `is-large` | `$size-4` (1.5rem) |

Responsive variants: `is-size-N-mobile`, `-tablet`, `-tablet-only`, `-touch`, `-desktop`, `-desktop-only`, `-widescreen`, `-widescreen-only`, `-fullhd`.

Font weight helpers: `has-text-weight-light` (300), `-normal` (400), `-medium` (500), `-semibold` (600), `-bold` (700), `-extrabold` (800).

---

## Responsive Breakpoints

Defined in `sass/utilities/initial-variables.scss`:

| Name | Range | Pixels |
|---|---|---|
| `mobile` | `until $tablet` | `< 769px` |
| `tablet` | `from $tablet` | `≥ 769px` |
| `tablet-only` | `$tablet → $desktop` | `769–1023px` |
| `touch` | `until $desktop` | `< 1024px` (mobile + tablet) |
| `desktop` | `from $desktop` | `≥ 1024px` |
| `desktop-only` | `$desktop → $widescreen` | `1024–1215px` |
| `until-widescreen` | `until $widescreen` | `< 1216px` |
| `widescreen` | `from $widescreen` | `≥ 1216px` |
| `widescreen-only` | `$widescreen → $fullhd` | `1216–1407px` |
| `until-fullhd` | `until $fullhd` | `< 1408px` |
| `fullhd` | `from $fullhd` | `≥ 1408px` |

Note: the gap (`$gap`) is `32px`; `desktop = 960 + 2*gap = 1024px`, `widescreen = 1152 + 2*gap = 1216px`, `fullhd = 1344 + 2*gap = 1408px`.

These suffixes appear on every responsive class: columns, helpers (display, visibility, text alignment, sizes, etc.), grid cells.

---

## Dark Mode & Theming

Bulma includes light + dark themes built from CSS custom properties.

**Three modes of activation:**

1. **System preference (default)** — Bulma honors `prefers-color-scheme`. No code needed.
2. **Explicit per-element override** via attribute or class:
   ```html
   <html data-theme="dark">…</html>
   <body class="theme-dark">…</body>
   ```
3. **Compile out** — use `bulma-no-dark-mode.css` if you don't want any dark mode.

The themes are wired up in `sass/themes/_index.scss`:

```scss
:root                     { @include light-theme; @include setup-theme; }
@media (prefers-color-scheme: light) { :root { @include light-theme; } }
@media (prefers-color-scheme: dark)  { :root { @include dark-theme; } }
[data-theme="light"], .theme-light  { @include light-theme; @include setup-theme; }
[data-theme="dark"],  .theme-dark   { @include dark-theme;  @include setup-theme; }
```

Every component reads its colors through CSS variables, so toggling `data-theme` re-skins the page instantly without recompiling Sass.

---

## CSS Variables

Bulma 1 is **CSS-variables-first**. Every component registers a tree of variables, all prefixed `--bulma-` (controlled by `$cssvars-prefix`). The most useful root-level variables:

### Colors (`hsla` form, plus `-h`, `-s`, `-l`, `-rgb`)

For each main color (`primary`, `link`, `info`, `success`, `warning`, `danger`, `light`, `dark`):

```
--bulma-primary           /* hsla() */
--bulma-primary-base
--bulma-primary-rgb       /* "r, g, b" tuple */
--bulma-primary-h
--bulma-primary-s
--bulma-primary-l
--bulma-primary-invert
--bulma-primary-invert-l
--bulma-primary-on-scheme
--bulma-primary-on-scheme-l
--bulma-primary-light, --bulma-primary-light-invert
--bulma-primary-dark,  --bulma-primary-dark-invert
--bulma-primary-soft,  --bulma-primary-soft-invert
--bulma-primary-bold,  --bulma-primary-bold-invert
--bulma-primary-00, --bulma-primary-05, … --bulma-primary-100
--bulma-primary-NN-l, --bulma-primary-NN-invert, --bulma-primary-NN-invert-l
```

### Scheme & surface

```
--bulma-scheme-h, --bulma-scheme-s
--bulma-scheme-main, --bulma-scheme-main-bis, --bulma-scheme-main-ter
--bulma-scheme-invert, --bulma-scheme-invert-bis, --bulma-scheme-invert-ter
--bulma-background, --bulma-background-hover, --bulma-background-active
--bulma-border-weak, --bulma-border, --bulma-border-hover, --bulma-border-active
--bulma-text, --bulma-text-weak, --bulma-text-strong
--bulma-link, --bulma-link-text, --bulma-link-text-hover, --bulma-link-text-active
--bulma-code, --bulma-code-background, --bulma-pre, --bulma-pre-background
--bulma-shadow
```

### Typography

```
--bulma-family-primary, --bulma-family-secondary, --bulma-family-code
--bulma-size-small, --bulma-size-normal, --bulma-size-medium, --bulma-size-large
--bulma-size-1 … --bulma-size-7
--bulma-weight-light/normal/medium/semibold/bold/extrabold
```

### Geometry & motion

```
--bulma-block-spacing
--bulma-radius-small, --bulma-radius, --bulma-radius-medium, --bulma-radius-large, --bulma-radius-rounded
--bulma-duration   (294ms)
--bulma-easing     (ease-out)
--bulma-speed      (86ms)
--bulma-control-radius, --bulma-control-radius-small
--bulma-control-border-width, --bulma-control-height (2.5em)
--bulma-control-line-height, --bulma-control-padding-vertical, --bulma-control-padding-horizontal
--bulma-control-size, --bulma-control-focus-shadow-l
```

### Focus

```
--bulma-focus-h, --bulma-focus-s, --bulma-focus-l
--bulma-focus-offset, --bulma-focus-style, --bulma-focus-width
--bulma-focus-shadow-size, --bulma-focus-shadow-alpha
```

Component-scoped variables follow `--bulma-{component}-*` naming (e.g. `--bulma-button-padding-horizontal`, `--bulma-card-radius`, `--bulma-navbar-height`). Override them at any selector to retheme that component:

```css
.my-section .button {
  --bulma-button-padding-horizontal: 2em;
  --bulma-button-weight: 700;
}
```

---

## Layout

### Container

Centers content with responsive max-widths.

```html
<div class="container">…</div>
<div class="container is-fluid">… edge-to-edge with 32px gap …</div>
<div class="container is-widescreen">caps at widescreen</div>
<div class="container is-fullhd">caps at fullhd</div>
<div class="container is-max-tablet">never wider than tablet</div>
<div class="container is-max-desktop">never wider than desktop</div>
<div class="container is-max-widescreen">never wider than widescreen</div>
```

### Section

Page-level vertical spacing.

```html
<section class="section">…</section>
<section class="section is-medium">…</section>
<section class="section is-large">…</section>
<section class="section is-fullheight">100vh tall</section>
```

### Hero

Full-width banner with optional head/body/foot.

```html
<section class="hero is-primary is-bold is-medium">
  <div class="hero-head">…optional top, often a navbar/tabs…</div>
  <div class="hero-body">
    <p class="title">Title</p>
    <p class="subtitle">Subtitle</p>
  </div>
  <div class="hero-foot">…optional bottom…</div>
</section>
```

Color: `is-{primary|link|info|success|warning|danger|light|dark|black|white}`.
Size: `is-small`, `is-medium`, `is-large`.
Height: `is-halfheight`, `is-fullheight`, `is-fullheight-with-navbar` (subtracts `--bulma-navbar-height`).
Style: `is-bold` (gradient background).

Sub-elements: `.hero-head`, `.hero-body`, `.hero-foot`, `.hero-buttons`, `.hero-video` (with optional `.is-transparent`).

### Level

Horizontally aligned, vertically centered row of items.

```html
<nav class="level">
  <div class="level-left">
    <div class="level-item"><p class="title is-5">123 posts</p></div>
    <div class="level-item">
      <input class="input" type="text" placeholder="Find a post">
    </div>
  </div>
  <div class="level-right">
    <p class="level-item"><a class="button is-success">New</a></p>
  </div>
</nav>
```

Modifiers: `is-mobile` (force horizontal even on mobile). Item: `is-narrow`, `is-flexible`.

### Media object

```html
<article class="media">
  <figure class="media-left">
    <p class="image is-64x64"><img src="avatar.png"></p>
  </figure>
  <div class="media-content">
    <div class="content">…</div>
  </div>
  <div class="media-right">
    <button class="delete"></button>
  </div>
</article>
```

Size modifier: `<article class="media is-large">…</article>`.

### Footer

```html
<footer class="footer">
  <div class="content has-text-centered">…</div>
</footer>
```

---

## Columns (12-column flexbox grid)

Bulma's classic flexbox grid (separate from `.grid` below).

```html
<div class="columns">
  <div class="column">First</div>
  <div class="column">Second</div>
  <div class="column">Third</div>
</div>
```

### Column sizes

Fractional names: `is-full`, `is-three-quarters`, `is-two-thirds`, `is-half`, `is-one-third`, `is-one-quarter`, `is-one-fifth`, `is-two-fifths`, `is-three-fifths`, `is-four-fifths`.

Numeric (1–12): `is-1`, `is-2`, … `is-12`.

`is-narrow` — column takes only the width it needs.

```html
<div class="columns">
  <div class="column is-half">half</div>
  <div class="column is-one-quarter">a quarter</div>
  <div class="column">fills the rest</div>
</div>
```

### Offsets

`is-offset-half`, `is-offset-one-quarter`, …, `is-offset-1` … `is-offset-12`.

### Responsive columns

Append a breakpoint suffix: `-mobile`, `-tablet`, `-touch`, `-desktop`, `-widescreen`, `-fullhd`.

```html
<div class="column is-half-mobile is-one-third-tablet is-one-quarter-desktop">
```

Note: a plain `is-half` (no suffix) only kicks in `from tablet up`. For mobile-first you usually need `is-half-mobile` too.

### Container modifiers (`.columns.is-*`)

| Modifier | Effect |
|---|---|
| `is-mobile` | Activate the flex layout on **all** screen sizes, including mobile |
| `is-desktop` | Activate **only** from desktop up |
| `is-multiline` | Allow column wrapping |
| `is-centered` | `justify-content: center` |
| `is-vcentered` | `align-items: center` |
| `is-gapless` | Remove the gap between columns |
| `is-0` … `is-8` | Set the column gap to `N * 0.25rem`. Responsive: `is-N-mobile`, `is-N-tablet`, `is-N-tablet-only`, `is-N-touch`, `is-N-desktop`, `is-N-desktop-only`, `is-N-widescreen`, `is-N-widescreen-only`, `is-N-fullhd` |

Default gap is `0.75rem` (CSS var `--bulma-column-gap`).

Nesting columns: any `.column` can hold another `.columns` — but you generally need `is-multiline` on the outer one for predictable wrapping.

---

## Grid (CSS Grid)

Bulma 1 added a real CSS Grid system in two flavors.

### Smart grid — auto-fit columns

```html
<div class="grid">
  <div class="cell">1</div>
  <div class="cell">2</div>
  <div class="cell">3</div>
  <!-- … -->
</div>
```

Columns auto-fit at min `--bulma-grid-column-min` (default 9rem). Tweak with `is-col-min-1` … `is-col-min-32` (multiples of `1.5rem` = 0.5 of the base):

```html
<div class="grid is-col-min-12">…12 * 1.5rem = 18rem min column…</div>
```

`is-auto-fill` on `.grid` swaps `auto-fit` for `auto-fill`.

Cell spans (1–12), supported on `.cell`:

```
is-col-start-N, is-col-end-N, is-col-from-end-N, is-col-span-N
is-row-start-N, is-row-end-N, is-row-from-end-N, is-row-span-N
is-col-start-end, is-row-start-end   /* shorthand for "to end" */
```

All cell modifiers are responsive (`-mobile`, `-tablet`, `-tablet-only`, `-desktop`, `-desktop-only`, `-widescreen`, `-widescreen-only`, `-fullhd`).

### Fixed grid — fixed column count via container queries

```html
<div class="fixed-grid has-4-cols">
  <div class="grid">
    <div class="cell">…</div>
  </div>
</div>
```

Counts: `has-1-cols` … `has-12-cols`. Responsive at the **container** level (uses CSS container queries internally, container name `bulma-fixed-grid`): `has-N-cols-mobile`, `-tablet`, `-desktop`, `-widescreen`, `-fullhd`.

`has-auto-count` on `.fixed-grid` produces 2 / 4 / 8 / 12 / 16 columns at mobile / tablet / desktop / widescreen / fullhd container widths.

Gap variables: `--bulma-grid-gap` (default 0.75rem), with optional `--bulma-grid-column-gap`, `--bulma-grid-row-gap` overrides.

---

## Elements

### Block

A simple wrapper that adds a consistent bottom margin (`--bulma-block-spacing`, 1.5rem) — the same spacing every primary component uses.

```html
<div class="block">First</div>
<div class="block">Second</div>
<div class="block">Third (no extra margin: it's the last child)</div>
```

### Box

A white card with a subtle shadow and rounded corners.

```html
<div class="box">…</div>
<a class="box" href="#">Hoverable / clickable box</a>
```

### Button

Workhorse element. Use the `<button>`, `<a>`, or `<input type="submit">` tag — Bulma styles them identically.

```html
<button class="button">Default</button>
<button class="button is-primary">Primary</button>
<button class="button is-link">Link</button>
<button class="button is-info is-light">Info light</button>
<button class="button is-warning is-dark">Warning dark</button>
<button class="button is-success is-soft">Success soft</button>
<button class="button is-danger is-bold">Danger bold</button>
```

**Colors**: `is-{primary|link|info|success|warning|danger|light|dark|black|white|text|ghost}`. The variants `is-light`, `is-dark`, `is-soft`, `is-bold` apply to colored buttons.

**Styles**: `is-outlined`, `is-inverted`, `is-text`, `is-ghost`.

**Sizes**: `is-small`, `is-normal`, `is-medium`, `is-large`. Add `is-responsive` so the button shrinks on mobile (mobile/tablet-only sizes are scaled-down).

**States**: `is-hovered`, `is-focused`, `is-active`, `is-loading` (transparent text + spinner), `is-static` (display-only), the HTML `disabled` attribute.

**Other modifiers**: `is-fullwidth`, `is-rounded`, `is-selected` (used in `.has-addons` groups).

#### Button group (`.buttons`)

```html
<div class="buttons">
  <button class="button is-primary">Save</button>
  <button class="button is-light">Cancel</button>
</div>

<div class="buttons are-medium">  <!-- size all children -->
  <button class="button">A</button>
  <button class="button">B</button>
</div>

<div class="buttons has-addons">  <!-- joined edges -->
  <button class="button is-selected is-info">Active</button>
  <button class="button">B</button>
</div>

<div class="buttons is-centered">…</div>
<div class="buttons is-right">…</div>
```

Sizes for `.buttons`: `are-small`, `are-medium`, `are-large`. A `.button.is-expanded` inside `.has-addons` grows to fill remaining space.

### Content

Reset for "WYSIWYG" / Markdown-style HTML. Apply to a wrapper to get sensible default styles for `h1–h6`, `p`, `ul`, `ol`, `dl`, `blockquote`, `table`, `figure`, `pre`, etc.

```html
<div class="content">
  <h1>Page title</h1>
  <p>Body copy with <strong>strong</strong> and <em>emphasis</em>.</p>
  <ul><li>One</li><li>Two</li></ul>
  <blockquote>A pull-quote</blockquote>
</div>
```

Sizes: `is-small`, `is-normal`, `is-medium`, `is-large`.
Ordered list types (override default `decimal`): `is-lower-alpha`, `is-lower-roman`, `is-upper-alpha`, `is-upper-roman`.

### Delete

Stylized "X" close button.

```html
<button class="delete"></button>
<button class="delete is-small"></button>
<button class="delete is-medium"></button>
<button class="delete is-large"></button>
```

### Icon / Icon Text

Wraps a font icon (Font Awesome, MDI, etc.).

```html
<span class="icon"><i class="fas fa-home"></i></span>
<span class="icon is-small"><i class="fas fa-info-circle"></i></span>
<span class="icon is-medium"><i class="fas fa-rocket"></i></span>
<span class="icon is-large has-text-info"><i class="fab fa-github fa-2x"></i></span>
```

For text + icon together use `.icon-text`:

```html
<span class="icon-text">
  <span class="icon has-text-success"><i class="fas fa-check"></i></span>
  <span>Saved</span>
</span>
```

`.icon-text` is `inline-flex` by default. Use a `<div class="icon-text">` for block-level layout.

### Image

A figure-style wrapper with built-in aspect ratios.

```html
<figure class="image is-128x128">
  <img src="…">
</figure>
<figure class="image is-16by9">
  <img src="…">  <!-- crops to 16:9 -->
</figure>
<figure class="image is-square">
  <img src="…">
</figure>
<img class="is-rounded" src="avatar.png">
```

**Sizes** (square): `is-16x16`, `is-24x24`, `is-32x32`, `is-48x48`, `is-64x64`, `is-96x96`, `is-128x128`.

**Aspect ratios**: `is-square` (1:1), `is-1by1`, `is-5by4`, `is-4by3`, `is-3by2`, `is-5by3`, `is-16by9`, `is-2by1`, `is-3by1`, `is-4by5`, `is-3by4`, `is-2by3`, `is-3by5`, `is-9by16`, `is-1by2`, `is-1by3`.

`is-fullwidth` makes the figure 100% wide. On the inner `<img>`, `is-rounded` makes it a circle. For non-`<img>` content (e.g. iframes) put `.has-ratio` on the child.

### Notification

Banner-style alert, often paired with `<button class="delete">`.

```html
<div class="notification is-primary">
  <button class="delete"></button>
  <strong>Heads up!</strong> Something happened.
</div>
```

Colors: all main colors. `is-light` and `is-dark` variants supported.

### Progress

Native `<progress>` element styled.

```html
<progress class="progress" value="30" max="100">30%</progress>
<progress class="progress is-info" value="60" max="100"></progress>
<progress class="progress is-success is-large" value="80" max="100"></progress>
<progress class="progress is-primary"></progress>  <!-- indeterminate -->
```

Colors: `is-{primary|link|info|success|warning|danger}`.
Sizes: `is-small`, `is-medium`, `is-large`.
Omit `value`/`max` for an indeterminate animation.

### Table

Wrap with `.table-container` for horizontal-scroll on mobile.

```html
<div class="table-container">
  <table class="table is-bordered is-striped is-narrow is-hoverable is-fullwidth">
    <thead><tr><th>Name</th><th>Age</th></tr></thead>
    <tbody>
      <tr><td>Alice</td><td>30</td></tr>
      <tr class="is-selected"><td>Bob</td><td>28</td></tr>
    </tbody>
    <tfoot><tr><th>Name</th><th>Age</th></tr></tfoot>
  </table>
</div>
```

Table modifiers: `is-bordered`, `is-striped`, `is-narrow`, `is-hoverable`, `is-fullwidth`.
Row / cell: `is-selected` colors a row, `is-narrow` shrinks a cell to content width, `is-vcentered` vertical-centers cell content. Cells and rows accept color modifiers (`is-primary`, `is-info`, etc.).

### Tag

Compact pill / chip.

```html
<span class="tag">Default</span>
<span class="tag is-primary">Primary</span>
<span class="tag is-warning is-light">Warning light</span>
<span class="tag is-medium">Medium</span>
<span class="tag is-rounded">Rounded</span>

<span class="tag is-success">
  Bulma
  <button class="delete is-small"></button>
</span>
```

Colors: all main colors + `is-light`. Sizes: `is-normal` (default), `is-medium`, `is-large`.
Modifiers: `is-rounded`, `is-delete` (turn a tag into an X).

`<a class="tag">` and `<button class="tag">` get hover/active styles automatically; for arbitrary tags add `.is-hoverable`.

#### Tag group

```html
<div class="tags">
  <span class="tag is-primary">v1.0.4</span>
  <span class="tag">latest</span>
</div>

<div class="tags has-addons">
  <span class="tag is-dark">version</span>
  <span class="tag is-info">1.0.4</span>
</div>

<div class="tags are-medium">  <!-- size all children -->
  <span class="tag">a</span><span class="tag">b</span>
</div>
```

Group modifiers: `are-medium`, `are-large`, `is-centered`, `is-right`, `has-addons`.

### Title / Subtitle

Headings replacement.

```html
<h1 class="title">Title</h1>
<h2 class="subtitle">Subtitle</h2>

<h1 class="title is-1">Biggest</h1>
<h6 class="title is-6">Smallest</h6>

<h1 class="title is-spaced">Title</h1>
<h2 class="subtitle">…still gets full margin between this and the title…</h2>
```

Sizes: `is-1` … `is-7`. By default a `.title` immediately followed by a `.subtitle` collapses its bottom margin — add `is-spaced` to keep it.

---

## Components

### Breadcrumb

```html
<nav class="breadcrumb has-arrow-separator" aria-label="breadcrumbs">
  <ul>
    <li><a href="#">Bulma</a></li>
    <li><a href="#">Documentation</a></li>
    <li class="is-active"><a href="#" aria-current="page">Breadcrumb</a></li>
  </ul>
</nav>
```

Alignment: `is-centered`, `is-right`.
Sizes: `is-small`, `is-medium`, `is-large`.
Separator styles: `has-arrow-separator` (→), `has-bullet-separator` (•), `has-dot-separator` (·), `has-succeeds-separator` (≻).
Mark current page with `is-active` on the `<li>`.

### Card

```html
<div class="card">
  <header class="card-header">
    <p class="card-header-title">Component</p>
    <button class="card-header-icon" aria-label="more">
      <span class="icon"><i class="fas fa-angle-down"></i></span>
    </button>
  </header>
  <div class="card-image">
    <figure class="image is-4by3"><img src="…"></figure>
  </div>
  <div class="card-content">
    <div class="content">Lorem ipsum…</div>
  </div>
  <footer class="card-footer">
    <a href="#" class="card-footer-item">Save</a>
    <a href="#" class="card-footer-item">Edit</a>
    <a href="#" class="card-footer-item">Delete</a>
  </footer>
</div>
```

Sub-elements: `.card-header`, `.card-header-title` (`is-centered` to center), `.card-header-icon`, `.card-image`, `.card-content`, `.card-footer`, `.card-footer-item`.

### Dropdown

```html
<div class="dropdown is-active">
  <div class="dropdown-trigger">
    <button class="button" aria-haspopup="true" aria-controls="dropdown-menu">
      <span>Choose</span>
      <span class="icon is-small"><i class="fas fa-angle-down"></i></span>
    </button>
  </div>
  <div class="dropdown-menu" id="dropdown-menu" role="menu">
    <div class="dropdown-content">
      <a href="#" class="dropdown-item">Option A</a>
      <a href="#" class="dropdown-item is-active">Option B</a>
      <hr class="dropdown-divider">
      <a href="#" class="dropdown-item">Option C</a>
    </div>
  </div>
</div>
```

Modifiers on `.dropdown`: `is-active` (open), `is-hoverable` (open on hover, no JS), `is-right` (right-aligned), `is-up` (open upward).
Items: `.dropdown-item`, `<a>`/`<button>` items support `:hover`, `is-active`/`is-selected`. `<hr class="dropdown-divider">`.

### Menu

Vertical sidebar navigation.

```html
<aside class="menu">
  <p class="menu-label">General</p>
  <ul class="menu-list">
    <li><a class="is-active">Dashboard</a></li>
    <li><a>Customers</a></li>
  </ul>
  <p class="menu-label">Administration</p>
  <ul class="menu-list">
    <li><a>Team Settings</a>
      <ul>
        <li><a>Members</a></li>
        <li><a>Plugins</a></li>
      </ul>
    </li>
  </ul>
</aside>
```

Sizes: `is-small`, `is-medium`, `is-large` on `.menu`. Items support `is-active`/`is-selected`.

### Message

```html
<article class="message is-info">
  <div class="message-header">
    <p>Heads up</p>
    <button class="delete" aria-label="delete"></button>
  </div>
  <div class="message-body">
    Lorem ipsum dolor sit amet…
  </div>
</article>
```

Header is optional. Colors: all main colors. Sizes: `is-small`, `is-medium`, `is-large`.

### Modal

```html
<div class="modal is-active">
  <div class="modal-background"></div>
  <div class="modal-content">
    <!-- arbitrary content (image, card, …) -->
  </div>
  <button class="modal-close is-large" aria-label="close"></button>
</div>
```

Or a structured "card" modal:

```html
<div class="modal is-active">
  <div class="modal-background"></div>
  <div class="modal-card">
    <header class="modal-card-head">
      <p class="modal-card-title">Modal title</p>
      <button class="delete" aria-label="close"></button>
    </header>
    <section class="modal-card-body">…</section>
    <footer class="modal-card-foot">
      <div class="buttons">
        <button class="button is-success">Save</button>
        <button class="button">Cancel</button>
      </div>
    </footer>
  </div>
</div>
```

`.modal.is-active` shows it. **Bulma ships no JS** — opening/closing is your job. To prevent body scroll while open, add `is-clipped` to `<html>`.

### Navbar

The most complex component.

```html
<nav class="navbar is-light has-shadow" role="navigation" aria-label="main navigation">
  <div class="navbar-brand">
    <a class="navbar-item" href="/"><img src="logo.png"></a>
    <a role="button" class="navbar-burger" aria-label="menu"
       aria-expanded="false" data-target="mainMenu">
      <span aria-hidden="true"></span>
      <span aria-hidden="true"></span>
      <span aria-hidden="true"></span>
      <span aria-hidden="true"></span>
    </a>
  </div>
  <div id="mainMenu" class="navbar-menu">
    <div class="navbar-start">
      <a class="navbar-item">Home</a>
      <a class="navbar-item">Docs</a>
      <div class="navbar-item has-dropdown is-hoverable">
        <a class="navbar-link">More</a>
        <div class="navbar-dropdown">
          <a class="navbar-item">About</a>
          <a class="navbar-item">Jobs</a>
          <hr class="navbar-divider">
          <a class="navbar-item">Report an issue</a>
        </div>
      </div>
    </div>
    <div class="navbar-end">
      <div class="navbar-item">
        <div class="buttons">
          <a class="button is-primary"><strong>Sign up</strong></a>
          <a class="button is-light">Log in</a>
        </div>
      </div>
    </div>
  </div>
</nav>
```

**Container modifiers** (`.navbar.is-*`):
- Color: every main color (sets background + auto text color).
- `has-shadow` — bottom box-shadow.
- `is-spaced` — pads brand/menu and rounds items.
- `is-transparent` — transparent items (use inside `.hero`).
- `is-fixed-top`, `is-fixed-bottom` — pin to top/bottom. Pair with `<html class="has-navbar-fixed-top">` (or `…-bottom`) to push body content. Variants per breakpoint: `is-fixed-top-touch`, `is-fixed-top-desktop`, etc.

**Items**:
- `.navbar-item` — generic clickable item. Sub-modifier `is-tab`, `is-active`/`is-selected`, `is-expanded`, `has-dropdown`, `has-dropdown-up`.
- `.navbar-link` — same as `.navbar-item` but with arrow indicator. `is-arrowless` removes it.
- `.navbar-burger` — mobile hamburger; toggle `is-active` on it AND on `.navbar-menu` to open. Bulma supplies the visual, you supply the JS.
- `.navbar-dropdown` — appears under a `.has-dropdown` parent. `is-boxed` (boxed-style dropdown), `is-right` (right-aligned).
- `.has-dropdown.is-hoverable` — open on hover (JS-free).
- `.navbar-divider` — horizontal rule between items.

**Layout sub-elements**: `.navbar-brand`, `.navbar-tabs`, `.navbar-menu`, `.navbar-start`, `.navbar-end`, `.navbar-content`.

`--bulma-navbar-height` defaults to `3.25rem` and is exposed at `:root`.

### Pagination

```html
<nav class="pagination is-centered" role="navigation" aria-label="pagination">
  <a class="pagination-previous">Previous</a>
  <a class="pagination-next">Next page</a>
  <ul class="pagination-list">
    <li><a class="pagination-link" aria-label="Goto page 1">1</a></li>
    <li><span class="pagination-ellipsis">&hellip;</span></li>
    <li><a class="pagination-link is-current"
           aria-label="Page 46" aria-current="page">46</a></li>
    <li><a class="pagination-link">47</a></li>
    <li><span class="pagination-ellipsis">&hellip;</span></li>
    <li><a class="pagination-link">86</a></li>
  </ul>
</nav>
```

Modifiers: `is-centered`, `is-right`, `is-rounded`. Sizes: `is-small`, `is-medium`, `is-large`. Disabled: `is-disabled` or HTML `disabled`. Current page: `is-current`/`is-selected`.

### Panel

A more compact, decorated container.

```html
<nav class="panel is-primary">
  <p class="panel-heading">Repositories</p>
  <div class="panel-block">
    <p class="control has-icons-left">
      <input class="input" type="text" placeholder="Search">
      <span class="icon is-left"><i class="fas fa-search"></i></span>
    </p>
  </div>
  <p class="panel-tabs">
    <a class="is-active">All</a>
    <a>Public</a>
    <a>Private</a>
  </p>
  <a class="panel-block is-active">
    <span class="panel-icon"><i class="fas fa-book"></i></span>
    bulma
  </a>
  <a class="panel-block">
    <span class="panel-icon"><i class="fas fa-book"></i></span>
    minireset.css
  </a>
</nav>
```

Colors on `.panel`: every main color. Sub-elements: `.panel-heading`, `.panel-tabs` (mark active `<a>` with `is-active`), `.panel-block` (`a`/`label`/`div` — use `is-active` to highlight, `is-wrapped` for wrapping content), `.panel-icon`, `.panel-list`.

### Tabs

```html
<div class="tabs is-centered is-boxed is-medium">
  <ul>
    <li class="is-active"><a>Pictures</a></li>
    <li><a>Music</a></li>
    <li><a>Videos</a></li>
  </ul>
</div>
```

Style: `is-boxed`, `is-toggle`, `is-toggle-rounded`.
Alignment: `is-centered`, `is-right`.
Other: `is-fullwidth`. Sizes: `is-small`, `is-medium`, `is-large`. Mark active with `is-active` on the `<li>`.

---

## Form Controls

### Field, Label, Help, Control

`.field` is the wrapper that gives form controls vertical rhythm. `.control` wraps an input plus optional icons.

```html
<div class="field">
  <label class="label">Name</label>
  <div class="control has-icons-left has-icons-right">
    <input class="input is-success" type="text" placeholder="Alice" value="Alice">
    <span class="icon is-small is-left"><i class="fas fa-user"></i></span>
    <span class="icon is-small is-right"><i class="fas fa-check"></i></span>
  </div>
  <p class="help is-success">This name is available</p>
</div>
```

**`.field` modifiers**:

| Modifier | Effect |
|---|---|
| `has-addons` | Joined controls (button + input + button). Inner `.control.is-expanded` grows. Variants: `has-addons-centered`, `has-addons-right`, `has-addons-fullwidth` |
| `is-grouped` | Side-by-side controls with gap. Variants: `is-grouped-centered`, `is-grouped-right`, `is-grouped-multiline` |
| `is-horizontal` | Label on the left, body on the right (uses `.field-label` and `.field-body` children, tablet+) |

**`.control` modifiers**:

| Modifier | Effect |
|---|---|
| `has-icons-left` / `has-icons-right` | Reserve room for an `.icon.is-left` / `.icon.is-right` |
| `is-loading` | Show a spinner inside the field. Sizes: `is-small`, `is-medium`, `is-large` |
| `is-expanded` | Grow inside `.has-addons` / `.is-grouped` |

**`.label`**: sizes `is-small`, `is-medium`, `is-large`.
**`.help`**: colors per main color (`is-info`, `is-danger`, …).

### Input, Textarea

```html
<input class="input" type="text" placeholder="Plain">
<input class="input is-primary is-medium is-rounded" type="text" placeholder="Pretty">
<input class="input is-danger" type="text" placeholder="Error">
<input class="input" type="text" disabled placeholder="Disabled">
<input class="input is-static" type="text" value="Read-only static" readonly>

<textarea class="textarea" placeholder="Textarea"></textarea>
<textarea class="textarea has-fixed-size" rows="5"></textarea>
```

Colors (border + focus): `is-{primary|link|info|success|warning|danger}`.
Sizes: `is-small`, `is-normal`, `is-medium`, `is-large`.
States/styles: `is-hovered`, `is-focused`, `is-rounded` (input), `is-static` (input — render value-only, no border), `is-fullwidth`, `is-inline` (input/textarea).
Textarea: `has-fixed-size` disables resize. `[rows]` sets fixed height.

### Select

Wrap the native `<select>` in a `.select` div.

```html
<div class="select">
  <select>
    <option>One</option>
    <option>Two</option>
  </select>
</div>

<div class="select is-multiple">
  <select multiple size="4">
    <option>A</option><option>B</option>
    <option>C</option><option>D</option>
  </select>
</div>

<div class="select is-primary is-rounded is-loading">…</div>
```

Modifiers on `.select`: colors, sizes (`is-small`/`is-medium`/`is-large`), `is-rounded`, `is-loading`, `is-multiple`, `is-fullwidth`, `is-disabled`.

### Checkbox, Radio

Bulma keeps native browser controls — these classes only style the **label** wrapper.

```html
<label class="checkbox">
  <input type="checkbox"> I agree
</label>

<label class="radio">
  <input type="radio" name="answer"> Yes
</label>
<label class="radio">
  <input type="radio" name="answer"> No
</label>

<div class="checkboxes">
  <label class="checkbox"><input type="checkbox"> A</label>
  <label class="checkbox"><input type="checkbox"> B</label>
  <label class="checkbox"><input type="checkbox"> C</label>
</div>
```

`.checkboxes` and `.radios` are flex-row containers with gap.

### File

```html
<div class="file has-name is-info">
  <label class="file-label">
    <input class="file-input" type="file" name="resume">
    <span class="file-cta">
      <span class="file-icon"><i class="fas fa-upload"></i></span>
      <span class="file-label">Choose a file…</span>
    </span>
    <span class="file-name">no file chosen</span>
  </label>
</div>
```

Modifiers on `.file`:
- Colors per main color.
- Sizes: `is-small`, `is-normal`, `is-medium`, `is-large`.
- `has-name` show the file-name strip; pair with `is-empty` when no file picked.
- `is-boxed` — vertical layout (icon above label). 
- `is-centered`, `is-right`, `is-fullwidth`.

---

## Helpers (utility classes)

All helpers use `!important` so they win specificity battles.

### Color helpers

```
has-text-{color}, has-text-{color}-invert, has-text-{color}-on-scheme
has-text-{color}-light, has-text-{color}-dark
has-text-{color}-soft, has-text-{color}-bold
has-text-{color}-{00|05|10|…|95|100}
has-text-{color}-{NN}-invert
has-text-current, has-text-inherit
has-background-* — same matrix as has-text-*
has-background-current, has-background-inherit
```

Where `{color}` is one of the 6 main colors + `light`/`dark`/`white`/`black`/`text`, and shades like `grey`, `grey-light`, etc.

For interactive elements (`a`, `button`, or any `.is-hoverable`) the hover/active states automatically darken or lighten.

### Spacing helpers

Pattern: `<property><direction>-<size>`

- Property: `m` (margin), `p` (padding)
- Direction: `t` (top), `r` (right), `b` (bottom), `l` (left), `x` (left+right), `y` (top+bottom), or omit for all sides
- Size: `0`, `1` (.25rem), `2` (.5rem), `3` (.75rem), `4` (1rem), `5` (1.5rem), `6` (3rem), or `auto`

Examples: `mt-4` (margin-top 1rem), `px-6` (padding x-axis 3rem), `mb-0`, `mx-auto`, `p-3`.
Plus `marginless` and `paddingless` (zero everything).

### Typography helpers

Sizes (responsive): `is-size-1` … `is-size-7`, with `-mobile`, `-tablet`, `-tablet-only`, `-touch`, `-desktop`, `-desktop-only`, `-widescreen`, `-widescreen-only`, `-fullhd` suffixes.

Alignment: `has-text-{centered|justified|left|right}`, also responsive (e.g. `has-text-centered-mobile`).

Transforms: `is-capitalized`, `is-lowercase`, `is-uppercase`, `is-italic`, `is-underlined`.

Weight: `has-text-weight-{light|normal|medium|semibold|bold|extrabold}`.

Family: `is-family-{primary|secondary|sans-serif|monospace|code}`.

### Flexbox helpers

```
is-flex-direction-{row|row-reverse|column|column-reverse}
is-flex-wrap-{nowrap|wrap|wrap-reverse}
is-justify-content-{flex-start|flex-end|center|space-between|space-around|space-evenly|start|end|left|right}
is-align-content-{flex-start|flex-end|center|space-between|space-around|space-evenly|stretch|start|end|baseline}
is-align-items-{stretch|flex-start|flex-end|center|baseline|start|end|self-start|self-end}
is-align-self-{auto|flex-start|flex-end|center|baseline|stretch}
is-flex-grow-{0..5}, is-flex-shrink-{0..5}
```

### Visibility / Display helpers

```
is-block, is-flex, is-inline, is-inline-block, is-inline-flex, is-grid, is-hidden
is-display-{block|flex|inline|inline-block|inline-flex|grid|none}
is-invisible, is-visibility-hidden
is-sr-only         /* screen-reader-only (visually hidden, still announced) */
```

All of these accept a breakpoint suffix (`-mobile`, `-tablet`, `-tablet-only`, `-touch`, `-desktop`, `-desktop-only`, `-widescreen`, `-widescreen-only`, `-fullhd`):

```html
<div class="is-hidden-mobile is-block-tablet">…</div>
```

### Border / Aspect / Float / Gap / Position / Overflow / Other

| Helper | Effect |
|---|---|
| `has-radius-{small,normal,large,rounded}` | Apply named border-radius |
| `is-radiusless` | `border-radius: 0` |
| `is-shadowless` | `box-shadow: none` |
| `is-aspect-ratio-{w}by{h}` | Same set as image ratios |
| `is-pulled-left`, `is-pulled-right`, `is-clearfix` | Float / clearfix |
| `is-float-{left|right|none}`, `is-clear-{left|right|both|none}` | Float utilities |
| `is-gap-N`, `is-column-gap-N`, `is-row-gap-N` (N = 0…8, half steps `0.5` etc.) | CSS Grid / flex gap (multiples of 0.5rem) |
| `is-gapless` | gap: 0 |
| `is-relative`, `is-position-{absolute,fixed,relative,static,sticky}` | Position |
| `is-overlay` | Absolutely positioned to fill parent (set `relative` on parent) |
| `is-overflow-{auto,clip,hidden,scroll,visible}`, `is-overflow-x-*`, `is-overflow-y-*` | Overflow |
| `is-clipped` | `overflow: hidden` (use on `<html>` to lock body scroll for modals) |
| `is-clickable` | `cursor: pointer; pointer-events: all` |
| `is-unselectable` | Prevent text selection |

---

## Skeletons

Loading-state placeholders.

```html
<!-- Replace any element's text with a pulse -->
<p class="is-skeleton">Will be hidden, shown as a bar</p>

<!-- Wrap an empty container that fills space -->
<div class="skeleton-block"></div>

<!-- Multiple text lines -->
<div class="skeleton-lines">
  <div></div><div></div><div></div><div></div>
</div>

<!-- Sit a skeleton next to existing content -->
<span class="has-skeleton">Loading...</span>
```

`.is-skeleton` works on inputs, textareas, checkboxes, and `.delete` buttons too.

---

## Sass Customization

Two paths:

### A. Override CSS variables only (no Sass build)

Set them at `:root` or any container:

```css
:root {
  --bulma-primary-h: 280;
  --bulma-primary-s: 80%;
  --bulma-primary-l: 55%;

  --bulma-link-h: 280; --bulma-link-s: 80%; --bulma-link-l: 55%;

  --bulma-radius: 0.25rem;
  --bulma-control-height: 2.25em;
  --bulma-family-primary: "Plus Jakarta Sans", system-ui, sans-serif;
}
```

This is the recommended modern approach — no rebuild, instant runtime theming.

### B. Customize via Sass

Override variables **before** importing modules:

```scss
@use "bulma/sass/utilities/initial-variables" with (
  $family-primary: '"Plus Jakarta Sans", sans-serif',
  $primary: hsl(280, 80%, 55%),
  $tablet: 720px
);

@use "bulma/sass" with (
  $custom-colors: (
    "twitter": (#1da1f2, white),
    "discord": (#7289da, white),
  )
);
```

Each Sass module exposes its component-scoped variables (e.g. `$button-padding-vertical`, `$navbar-height`, `$card-radius`) — see the corresponding `.scss` file.

### Modular imports

Pick only what you need:

```scss
@use "bulma/sass/utilities";
@use "bulma/sass/themes";
@use "bulma/sass/base";
@use "bulma/sass/elements/button";
@use "bulma/sass/elements/title";
@use "bulma/sass/form/shared";
@use "bulma/sass/form/input-textarea";
@use "bulma/sass/components/navbar";
@use "bulma/sass/grid/columns";
@use "bulma/sass/helpers/spacing";
```

The full forward chain is in `sass/_index.scss`:

```
utilities → themes → base → elements → form → components → grid → layout → base/skeleton → helpers
```

### Class-prefixing all output

```scss
@use "bulma/sass" with ($class-prefix: "bma-");
```

`is-` and `has-` prefixes are also configurable via `$helpers-prefix` and `$helpers-has-prefix`.

### Built-in Sass mixins worth knowing

In `sass/utilities/mixins.scss`:

- Responsive: `@include from($w)`, `@include until($w)`, `@include between($a, $b)`, plus convenience mixins `@include mobile`, `tablet`, `tablet-only`, `touch`, `desktop`, `desktop-only`, `until-widescreen`, `widescreen`, `widescreen-only`, `until-fullhd`, `fullhd`. Generic: `@include breakpoint("mobile") { … }`.
- Container queries: `@include container-from($name, $w)`, `@include container-until($name, $w)`.
- Components: `@include arrow($color)`, `@include burger($size)`, `@include delete`, `@include loader`, `@include overlay($offset)`.
- Reset: `@include reset` (strips browser defaults), `@include unselectable`, `@include clearfix`, `@include placeholder { … }`, `@include selection { … }`.
- Layout helpers: `@include center($w, $h)`, `@include fa($size, $dim)`.

Plus the `%control` placeholder selector that powers buttons, inputs, selects, file CTAs, and pagination items.

---

## JavaScript Notes (Bulma ships none)

Bulma is purely CSS. The following components have a "show / hide" UX that **you** must wire up (a few lines of vanilla JS or your framework of choice):

- **Navbar burger** — toggle `.is-active` on both the `.navbar-burger` and the matching `.navbar-menu` (use the burger's `data-target` attribute to find it).
- **Modal** — toggle `.is-active` on `.modal`, and add `is-clipped` to `<html>` while open to lock scroll.
- **Dropdown** — toggle `.is-active` on `.dropdown` (or rely on `.is-hoverable` for pure-CSS hover).
- **Tabs / Panel tabs** — swap `.is-active` between siblings; show/hide your tab content yourself.
- **File input** — Bulma styles it but doesn't update the displayed `.file-name`; listen for `change` and set `.textContent`.
- **Notification / Message delete** — listen for clicks on the inner `.delete` button and remove the parent.

---

## Migration Notes (v0 → v1)

If you are upgrading from Bulma 0.9.x, the most important breaking changes to be aware of:

- **Sass `@use` instead of `@import`** — Bulma now uses Dart Sass modules; old `@import "bulma/bulma"` still works as compatibility but `@use "bulma/bulma"` is preferred.
- **CSS variables everywhere** — most theming is now via CSS custom properties (`--bulma-*`), so runtime theme switching no longer requires recompilation.
- **Built-in dark mode** — Bulma honors `prefers-color-scheme: dark` by default. Use `bulma-no-dark-mode.css` if you don't want it.
- **No separate RTL build** — logical properties (`margin-inline-start`, `inset-inline-end`, etc.) replace LTR-specific declarations. The single `bulma.css` works in both directions.
- **New `.grid` / `.fixed-grid` / `.cell`** — true CSS Grid system, separate from `.columns`.
- **New colors variants** — `is-{color}-soft`, `is-{color}-bold`, plus the 21-step `has-text-{color}-NN` shade scale.
- **Skeletons** — `.is-skeleton`, `.skeleton-block`, `.skeleton-lines`, `.has-skeleton` are new in v1.
- **Renamed helpers** — Most `.has-*` and `.is-*` helpers from v0 still work; new aliases were added (e.g. `is-display-block` next to `is-block`, `is-visibility-hidden` next to `is-invisible`).
- **`.tile`** — removed in favor of the new grid utilities.

For the authoritative changelog, see `CHANGELOG.md` in the package root.

---

## Quick "cheat-sheet" recap

| Need | Use |
|---|---|
| One row of items | `.columns` (flex) or `.grid` (CSS Grid) or `.level` (justified row) |
| Page wrapper | `.container` inside `.section` |
| Banner | `.hero is-{color} is-{small/medium/large}` |
| Card | `.card` + `.card-header`, `.card-image`, `.card-content`, `.card-footer` |
| Top nav | `.navbar` + `.navbar-brand`, `.navbar-menu` |
| Dialog | `.modal.is-active` + `.modal-card` |
| Inline CTA | `.button.is-primary` |
| Form row | `.field` > `.label` + `.control` > `.input` + `.help` |
| Pill | `.tag is-{color}` |
| Banner alert | `.notification is-{color}` or `.message is-{color}` |
| Hide on mobile | `.is-hidden-mobile` |
| Center text | `.has-text-centered` |
| Add margin top | `.mt-{0..6}` |
| Color text | `.has-text-{color}` |
| Color background | `.has-background-{color}` |
| Theme toggle | `<html data-theme="dark">` |

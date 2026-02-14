# Tailwind CSS v4 Reference

> Tailwind CSS v4.0 — utility-first CSS framework with zero-runtime, CSS-first configuration.
> Requires: Safari 16.4+, Chrome 111+, Firefox 128+.
> Docs: https://tailwindcss.com/docs

---

## 1. Installation

### Vite Plugin (Recommended)

```bash
npm install tailwindcss @tailwindcss/vite
```

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [tailwindcss()],
})
```

```css
/* app.css */
@import "tailwindcss";
```

### PostCSS

```bash
npm install tailwindcss @tailwindcss/postcss postcss
```

```javascript
// postcss.config.mjs
export default {
  plugins: {
    "@tailwindcss/postcss": {},
  },
}
```

```css
@import "tailwindcss";
```

### CLI

```bash
npm install tailwindcss @tailwindcss/cli
```

```bash
npx @tailwindcss/cli -i ./src/input.css -o ./src/output.css --watch
```

### CDN (Play / Prototyping Only)

```html
<script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
```

Custom theme with CDN:

```html
<style type="text/tailwindcss">
  @theme {
    --color-brand: #3f3cbb;
  }
</style>
```

### Framework Guides

Framework-specific guides available for: Laravel, SvelteKit, Next.js, Nuxt, React Router, SolidJS, Angular, and more at https://tailwindcss.com/docs/installation/framework-guides

---

## 2. Core Concepts

### Utility-First

Style elements by composing single-purpose classes directly in markup:

```html
<div class="mx-auto max-w-sm rounded-xl bg-white p-6 shadow-lg">
  <h3 class="text-xl font-medium text-black">Card Title</h3>
  <p class="text-gray-500">Card content goes here.</p>
</div>
```

### Responsive Design (Mobile-First)

Unprefixed utilities apply to all sizes. Prefixed utilities apply at that breakpoint **and above**:

```html
<div class="w-full md:w-1/2 lg:w-1/3">
  <!-- Full width on mobile, half on md+, third on lg+ -->
</div>
```

### State Variants

Prefix utilities with state modifiers:

```html
<button class="bg-blue-500 hover:bg-blue-700 focus:outline-2 active:bg-blue-800">
  Click me
</button>
```

### Dark Mode

Default uses `prefers-color-scheme`:

```html
<div class="bg-white dark:bg-gray-900 text-black dark:text-white">
  Adapts to system theme
</div>
```

### Stacking Variants

Multiple variants stack left-to-right (v4 order):

```html
<button class="dark:md:hover:bg-fuchsia-600">Save</button>
```

### Important Modifier (v4 syntax)

```html
<!-- v3: !flex    v4: flex! -->
<div class="flex! bg-red-500!">Important utilities</div>
```

### Arbitrary Values

```html
<div class="w-[200px] bg-[#1da1f2] text-[clamp(1rem,2vw,2rem)]">
  Custom one-off values
</div>
```

### CSS Variables in Arbitrary Values (v4 syntax)

```html
<!-- v3: bg-[--brand-color]    v4: bg-(--brand-color) -->
<div class="bg-(--brand-color)">Uses CSS variable</div>
```

---

## 3. Layout

### Display

| Class | CSS |
|-------|-----|
| `block` | `display: block` |
| `inline-block` | `display: inline-block` |
| `inline` | `display: inline` |
| `flex` | `display: flex` |
| `inline-flex` | `display: inline-flex` |
| `grid` | `display: grid` |
| `inline-grid` | `display: inline-grid` |
| `table` | `display: table` |
| `contents` | `display: contents` |
| `hidden` | `display: none` |
| `flow-root` | `display: flow-root` |
| `list-item` | `display: list-item` |
| `sr-only` | Visually hidden, accessible to screen readers |
| `not-sr-only` | Undo `sr-only` |

### Position

| Class | CSS |
|-------|-----|
| `static` | `position: static` |
| `relative` | `position: relative` |
| `absolute` | `position: absolute` |
| `fixed` | `position: fixed` |
| `sticky` | `position: sticky` |

**Inset utilities:** `top-*`, `right-*`, `bottom-*`, `left-*`, `inset-*`, `inset-x-*`, `inset-y-*`

```html
<div class="relative">
  <div class="absolute top-0 right-0">Positioned</div>
</div>
```

### Z-Index

| Class | CSS |
|-------|-----|
| `z-0` | `z-index: 0` |
| `z-10` | `z-index: 10` |
| `z-20` | `z-index: 20` |
| `z-30` | `z-index: 30` |
| `z-40` | `z-index: 40` |
| `z-50` | `z-index: 50` |
| `z-auto` | `z-index: auto` |

### Container

In v4, container is customized via `@utility`:

```css
@utility container {
  margin-inline: auto;
  padding-inline: 2rem;
}
```

### Box Sizing

| Class | CSS |
|-------|-----|
| `box-border` | `box-sizing: border-box` |
| `box-content` | `box-sizing: content-box` |

### Float & Clear

| Class | CSS |
|-------|-----|
| `float-left` | `float: left` |
| `float-right` | `float: right` |
| `float-none` | `float: none` |
| `clear-left` | `clear: left` |
| `clear-right` | `clear: right` |
| `clear-both` | `clear: both` |

### Object Fit

| Class | CSS |
|-------|-----|
| `object-contain` | `object-fit: contain` |
| `object-cover` | `object-fit: cover` |
| `object-fill` | `object-fit: fill` |
| `object-none` | `object-fit: none` |
| `object-scale-down` | `object-fit: scale-down` |

### Overflow

| Class | CSS |
|-------|-----|
| `overflow-auto` | `overflow: auto` |
| `overflow-hidden` | `overflow: hidden` |
| `overflow-visible` | `overflow: visible` |
| `overflow-scroll` | `overflow: scroll` |
| `overflow-clip` | `overflow: clip` |
| `overflow-x-auto` | `overflow-x: auto` |
| `overflow-y-auto` | `overflow-y: auto` |

### Columns

| Class | CSS |
|-------|-----|
| `columns-1` through `columns-12` | `columns: 1` through `columns: 12` |
| `columns-auto` | `columns: auto` |
| `columns-3xs` through `columns-7xl` | Named column widths |

### Flexbox

#### Direction & Wrap

| Class | CSS |
|-------|-----|
| `flex-row` | `flex-direction: row` |
| `flex-row-reverse` | `flex-direction: row-reverse` |
| `flex-col` | `flex-direction: column` |
| `flex-col-reverse` | `flex-direction: column-reverse` |
| `flex-wrap` | `flex-wrap: wrap` |
| `flex-wrap-reverse` | `flex-wrap: wrap-reverse` |
| `flex-nowrap` | `flex-wrap: nowrap` |

#### Flex Shorthand

| Class | CSS |
|-------|-----|
| `flex-1` | `flex: 1 1 0%` |
| `flex-auto` | `flex: auto` |
| `flex-initial` | `flex: 0 auto` |
| `flex-none` | `flex: none` |

#### Grow & Shrink

| Class | CSS |
|-------|-----|
| `grow` | `flex-grow: 1` |
| `grow-0` | `flex-grow: 0` |
| `shrink` | `flex-shrink: 1` |
| `shrink-0` | `flex-shrink: 0` |

#### Flex Basis

`basis-<number>`, `basis-auto`, `basis-full`, `basis-1/2`, `basis-1/3`, etc.

#### Justify Content

| Class | CSS |
|-------|-----|
| `justify-start` | `justify-content: flex-start` |
| `justify-end` | `justify-content: flex-end` |
| `justify-center` | `justify-content: center` |
| `justify-between` | `justify-content: space-between` |
| `justify-around` | `justify-content: space-around` |
| `justify-evenly` | `justify-content: space-evenly` |
| `justify-stretch` | `justify-content: stretch` |

#### Align Items

| Class | CSS |
|-------|-----|
| `items-start` | `align-items: flex-start` |
| `items-end` | `align-items: flex-end` |
| `items-center` | `align-items: center` |
| `items-baseline` | `align-items: baseline` |
| `items-stretch` | `align-items: stretch` |

#### Align Self

| Class | CSS |
|-------|-----|
| `self-auto` | `align-self: auto` |
| `self-start` | `align-self: flex-start` |
| `self-end` | `align-self: flex-end` |
| `self-center` | `align-self: center` |
| `self-stretch` | `align-self: stretch` |
| `self-baseline` | `align-self: baseline` |

#### Gap

`gap-<number>`, `gap-x-<number>`, `gap-y-<number>` (uses spacing scale)

#### Order

`order-<number>`, `order-first` (-9999), `order-last` (9999), `order-none` (0)

### Grid

#### Template Columns & Rows

| Class | CSS |
|-------|-----|
| `grid-cols-<n>` | `grid-template-columns: repeat(n, minmax(0, 1fr))` |
| `grid-cols-none` | `grid-template-columns: none` |
| `grid-cols-subgrid` | `grid-template-columns: subgrid` |
| `grid-rows-<n>` | `grid-template-rows: repeat(n, minmax(0, 1fr))` |
| `grid-rows-none` | `grid-template-rows: none` |
| `grid-rows-subgrid` | `grid-template-rows: subgrid` |

#### Spanning

| Class | CSS |
|-------|-----|
| `col-span-<n>` | `grid-column: span n / span n` |
| `col-span-full` | `grid-column: 1 / -1` |
| `col-start-<n>` | `grid-column-start: n` |
| `col-end-<n>` | `grid-column-end: n` |
| `row-span-<n>` | `grid-row: span n / span n` |
| `row-span-full` | `grid-row: 1 / -1` |
| `row-start-<n>` | `grid-row-start: n` |
| `row-end-<n>` | `grid-row-end: n` |

#### Auto Flow

| Class | CSS |
|-------|-----|
| `grid-flow-row` | `grid-auto-flow: row` |
| `grid-flow-col` | `grid-auto-flow: column` |
| `grid-flow-dense` | `grid-auto-flow: dense` |
| `grid-flow-row-dense` | `grid-auto-flow: row dense` |
| `grid-flow-col-dense` | `grid-auto-flow: column dense` |

#### Auto Sizing

| Class | CSS |
|-------|-----|
| `auto-cols-auto` | `grid-auto-columns: auto` |
| `auto-cols-min` | `grid-auto-columns: min-content` |
| `auto-cols-max` | `grid-auto-columns: max-content` |
| `auto-cols-fr` | `grid-auto-columns: minmax(0, 1fr)` |
| `auto-rows-auto` | `grid-auto-rows: auto` |
| `auto-rows-min` | `grid-auto-rows: min-content` |
| `auto-rows-max` | `grid-auto-rows: max-content` |
| `auto-rows-fr` | `grid-auto-rows: minmax(0, 1fr)` |

#### Arbitrary Grid Values (v4 syntax)

```html
<!-- v3: grid-cols-[max-content,auto]  v4: grid-cols-[max-content_auto] -->
<div class="grid grid-cols-[200px_minmax(900px,_1fr)_100px]">...</div>
```

#### Grid Example

```html
<div class="grid grid-cols-1 gap-4 md:grid-cols-3">
  <div class="col-span-2">Wide column</div>
  <div>Narrow column</div>
</div>
```

---

## 4. Spacing

### Spacing Scale

All spacing utilities use `calc(var(--spacing) * <number>)`. Default `--spacing: 0.25rem` (4px).

| Class | Value |
|-------|-------|
| `*-0` | 0 |
| `*-px` | 1px |
| `*-0.5` | 0.125rem (2px) |
| `*-1` | 0.25rem (4px) |
| `*-1.5` | 0.375rem (6px) |
| `*-2` | 0.5rem (8px) |
| `*-2.5` | 0.625rem (10px) |
| `*-3` | 0.75rem (12px) |
| `*-3.5` | 0.875rem (14px) |
| `*-4` | 1rem (16px) |
| `*-5` | 1.25rem (20px) |
| `*-6` | 1.5rem (24px) |
| `*-7` | 1.75rem (28px) |
| `*-8` | 2rem (32px) |
| `*-9` | 2.25rem (36px) |
| `*-10` | 2.5rem (40px) |
| `*-11` | 2.75rem (44px) |
| `*-12` | 3rem (48px) |
| `*-14` | 3.5rem (56px) |
| `*-16` | 4rem (64px) |
| `*-20` | 5rem (80px) |
| `*-24` | 6rem (96px) |
| `*-28` | 7rem (112px) |
| `*-32` | 8rem (128px) |
| `*-36` | 9rem (144px) |
| `*-40` | 10rem (160px) |
| `*-44` | 11rem (176px) |
| `*-48` | 12rem (192px) |
| `*-52` | 13rem (208px) |
| `*-56` | 14rem (224px) |
| `*-60` | 15rem (240px) |
| `*-64` | 16rem (256px) |
| `*-72` | 18rem (288px) |
| `*-80` | 20rem (320px) |
| `*-96` | 24rem (384px) |

In v4, any integer works with the spacing scale (e.g., `p-13` = `3.25rem`).

### Padding

| Prefix | Applies to |
|--------|-----------|
| `p-` | All sides |
| `px-` | Left + right (padding-inline) |
| `py-` | Top + bottom (padding-block) |
| `pt-` | Top |
| `pr-` | Right |
| `pb-` | Bottom |
| `pl-` | Left |
| `ps-` | Inline-start (LTR/RTL aware) |
| `pe-` | Inline-end (LTR/RTL aware) |

### Margin

Same prefixes as padding but with `m-`: `m-`, `mx-`, `my-`, `mt-`, `mr-`, `mb-`, `ml-`, `ms-`, `me-`

Margin also supports `auto`: `mx-auto`, `ml-auto`, `mr-auto`

Negative margins: `-m-4`, `-mt-2`, etc.

### Space Between

Adds margin between child elements:

| Class | Effect |
|-------|--------|
| `space-x-<n>` | Horizontal space between children |
| `space-y-<n>` | Vertical space between children |

```html
<div class="flex space-x-4">
  <div>1</div>
  <div>2</div>
  <div>3</div>
</div>
```

**v4 change:** `space-y-4` now applies `margin-bottom` to `:not(:last-child)` instead of `margin-top` to `~ :not([hidden])`.

---

## 5. Sizing

### Width

| Class | CSS |
|-------|-----|
| `w-<number>` | `width: calc(var(--spacing) * n)` |
| `w-auto` | `width: auto` |
| `w-px` | `width: 1px` |
| `w-full` | `width: 100%` |
| `w-screen` | `width: 100vw` |
| `w-dvw` | `width: 100dvw` |
| `w-svw` | `width: 100svw` |
| `w-lvw` | `width: 100lvw` |
| `w-min` | `width: min-content` |
| `w-max` | `width: max-content` |
| `w-fit` | `width: fit-content` |
| `w-1/2` | `width: 50%` |
| `w-1/3` | `width: 33.333%` |
| `w-2/3` | `width: 66.667%` |
| `w-1/4` | `width: 25%` |
| `w-3/4` | `width: 75%` |
| `w-1/5` | `width: 20%` |
| `w-1/6` | `width: 16.667%` |
| `w-5/6` | `width: 83.333%` |

Container-scale widths: `w-3xs` (16rem) through `w-7xl` (80rem).

### Height

Same patterns: `h-<number>`, `h-auto`, `h-full`, `h-screen`, `h-dvh`, `h-svh`, `h-lvh`, `h-min`, `h-max`, `h-fit`

### Size (Width + Height simultaneously)

`size-<number>`, `size-full`, `size-auto`, `size-min`, `size-max`, `size-fit`

```html
<img class="size-16" src="avatar.jpg" /> <!-- w-16 h-16 -->
```

### Min/Max Width & Height

| Class | CSS |
|-------|-----|
| `min-w-<number>` | `min-width: calc(var(--spacing) * n)` |
| `min-w-full` | `min-width: 100%` |
| `min-w-min` | `min-width: min-content` |
| `min-w-max` | `min-width: max-content` |
| `min-w-fit` | `min-width: fit-content` |
| `max-w-<number>` | `max-width: calc(var(--spacing) * n)` |
| `max-w-none` | `max-width: none` |
| `max-w-full` | `max-width: 100%` |
| `max-w-prose` | `max-width: 65ch` |
| `max-w-screen` | `max-width: 100vw` |
| `max-w-3xs` to `max-w-7xl` | Named container widths |
| `min-h-<number>` | `min-height: calc(var(--spacing) * n)` |
| `min-h-full` | `min-height: 100%` |
| `min-h-screen` | `min-height: 100vh` |
| `min-h-dvh` | `min-height: 100dvh` |
| `max-h-<number>` | `max-height: calc(var(--spacing) * n)` |
| `max-h-none` | `max-height: none` |
| `max-h-full` | `max-height: 100%` |
| `max-h-screen` | `max-height: 100vh` |

---

## 6. Typography

### Font Family

| Class | CSS |
|-------|-----|
| `font-sans` | `font-family: var(--font-sans)` |
| `font-serif` | `font-family: var(--font-serif)` |
| `font-mono` | `font-family: var(--font-mono)` |

### Font Size

| Class | Size | Line Height |
|-------|------|-------------|
| `text-xs` | 0.75rem (12px) | ~1.33 |
| `text-sm` | 0.875rem (14px) | ~1.43 |
| `text-base` | 1rem (16px) | 1.5 |
| `text-lg` | 1.125rem (18px) | ~1.56 |
| `text-xl` | 1.25rem (20px) | 1.4 |
| `text-2xl` | 1.5rem (24px) | ~1.33 |
| `text-3xl` | 1.875rem (30px) | 1.2 |
| `text-4xl` | 2.25rem (36px) | ~1.11 |
| `text-5xl` | 3rem (48px) | 1 |
| `text-6xl` | 3.75rem (60px) | 1 |
| `text-7xl` | 4.5rem (72px) | 1 |
| `text-8xl` | 6rem (96px) | 1 |
| `text-9xl` | 8rem (128px) | 1 |

**Line-height shorthand:** `text-sm/6` = text-sm with 1.5rem line-height.

### Font Weight

| Class | CSS |
|-------|-----|
| `font-thin` | `font-weight: 100` |
| `font-extralight` | `font-weight: 200` |
| `font-light` | `font-weight: 300` |
| `font-normal` | `font-weight: 400` |
| `font-medium` | `font-weight: 500` |
| `font-semibold` | `font-weight: 600` |
| `font-bold` | `font-weight: 700` |
| `font-extrabold` | `font-weight: 800` |
| `font-black` | `font-weight: 900` |

### Letter Spacing

| Class | CSS |
|-------|-----|
| `tracking-tighter` | `letter-spacing: -0.05em` |
| `tracking-tight` | `letter-spacing: -0.025em` |
| `tracking-normal` | `letter-spacing: 0em` |
| `tracking-wide` | `letter-spacing: 0.025em` |
| `tracking-wider` | `letter-spacing: 0.05em` |
| `tracking-widest` | `letter-spacing: 0.1em` |

### Line Height

| Class | CSS |
|-------|-----|
| `leading-none` | `line-height: 1` |
| `leading-tight` | `line-height: 1.25` |
| `leading-snug` | `line-height: 1.375` |
| `leading-normal` | `line-height: 1.5` |
| `leading-relaxed` | `line-height: 1.625` |
| `leading-loose` | `line-height: 2` |
| `leading-<number>` | `line-height: calc(var(--spacing) * n)` |

### Text Align

| Class | CSS |
|-------|-----|
| `text-left` | `text-align: left` |
| `text-center` | `text-align: center` |
| `text-right` | `text-align: right` |
| `text-justify` | `text-align: justify` |
| `text-start` | `text-align: start` |
| `text-end` | `text-align: end` |

### Text Color

```html
<p class="text-gray-700">Dark gray text</p>
<p class="text-blue-500/75">Blue at 75% opacity</p>
<p class="text-blue-600 dark:text-sky-400">Adaptive color</p>
```

### Text Decoration

| Class | CSS |
|-------|-----|
| `underline` | `text-decoration-line: underline` |
| `overline` | `text-decoration-line: overline` |
| `line-through` | `text-decoration-line: line-through` |
| `no-underline` | `text-decoration-line: none` |

Decoration style: `decoration-solid`, `decoration-dashed`, `decoration-dotted`, `decoration-double`, `decoration-wavy`

Decoration color: `decoration-red-500`, etc.

Decoration thickness: `decoration-auto`, `decoration-from-font`, `decoration-0` through `decoration-8`

Underline offset: `underline-offset-auto`, `underline-offset-0` through `underline-offset-8`

### Text Transform

| Class | CSS |
|-------|-----|
| `uppercase` | `text-transform: uppercase` |
| `lowercase` | `text-transform: lowercase` |
| `capitalize` | `text-transform: capitalize` |
| `normal-case` | `text-transform: none` |

### Text Overflow

| Class | CSS |
|-------|-----|
| `truncate` | `overflow: hidden; text-overflow: ellipsis; white-space: nowrap` |
| `text-ellipsis` | `text-overflow: ellipsis` |
| `text-clip` | `text-overflow: clip` |

### Whitespace

| Class | CSS |
|-------|-----|
| `whitespace-normal` | `white-space: normal` |
| `whitespace-nowrap` | `white-space: nowrap` |
| `whitespace-pre` | `white-space: pre` |
| `whitespace-pre-line` | `white-space: pre-line` |
| `whitespace-pre-wrap` | `white-space: pre-wrap` |
| `whitespace-break-spaces` | `white-space: break-spaces` |

### Word Break

| Class | CSS |
|-------|-----|
| `break-normal` | `overflow-wrap: normal; word-break: normal` |
| `break-words` | `overflow-wrap: break-word` |
| `break-all` | `word-break: break-all` |
| `break-keep` | `word-break: keep-all` |

### Line Clamp

`line-clamp-<n>` truncates text to n lines with ellipsis. `line-clamp-none` removes it.

```html
<p class="line-clamp-3">Long paragraph truncated to 3 lines...</p>
```

---

## 7. Backgrounds

### Background Color

```html
<div class="bg-white">White</div>
<div class="bg-blue-500">Blue</div>
<div class="bg-blue-500/50">Blue at 50% opacity</div>
<div class="bg-black dark:bg-white">Adaptive</div>
```

### Gradients

```html
<!-- Linear gradient (v4: bg-linear-to-r replaces bg-gradient-to-r) -->
<div class="bg-linear-to-r from-blue-500 to-purple-500">Linear</div>
<div class="bg-linear-to-br from-blue-500 via-green-500 to-yellow-500">With via</div>

<!-- Radial gradient (v4 new) -->
<div class="bg-radial from-blue-500 to-purple-500">Radial</div>

<!-- Conic gradient (v4 new) -->
<div class="bg-conic from-blue-500 via-green-500 to-yellow-500">Conic</div>
```

**v4 change:** `bg-gradient-to-*` renamed to `bg-linear-to-*`. Must explicitly reset `via-none` in dark mode overrides.

Gradient directions: `to-t`, `to-tr`, `to-r`, `to-br`, `to-b`, `to-bl`, `to-l`, `to-tl`

### Background Size

| Class | CSS |
|-------|-----|
| `bg-auto` | `background-size: auto` |
| `bg-cover` | `background-size: cover` |
| `bg-contain` | `background-size: contain` |

### Background Position

| Class | CSS |
|-------|-----|
| `bg-center` | `background-position: center` |
| `bg-top` | `background-position: top` |
| `bg-right` | `background-position: right` |
| `bg-bottom` | `background-position: bottom` |
| `bg-left` | `background-position: left` |
| `bg-left-top` | `background-position: left top` |
| `bg-right-bottom` | `background-position: right bottom` |

### Background Repeat

| Class | CSS |
|-------|-----|
| `bg-repeat` | `background-repeat: repeat` |
| `bg-no-repeat` | `background-repeat: no-repeat` |
| `bg-repeat-x` | `background-repeat: repeat-x` |
| `bg-repeat-y` | `background-repeat: repeat-y` |
| `bg-repeat-round` | `background-repeat: round` |
| `bg-repeat-space` | `background-repeat: space` |

---

## 8. Borders

### Border Width

| Class | CSS |
|-------|-----|
| `border` | `border-width: 1px` |
| `border-0` | `border-width: 0` |
| `border-2` | `border-width: 2px` |
| `border-4` | `border-width: 4px` |
| `border-8` | `border-width: 8px` |
| `border-t`, `border-r`, `border-b`, `border-l` | Side-specific |
| `border-x`, `border-y` | Axis-specific |

**v4 change:** Default border color is now `currentColor` (was `gray-200`). Add `border-gray-200` explicitly for v3 behavior.

### Border Color

```html
<div class="border border-gray-300">Gray border</div>
<div class="border border-red-500/50">Semi-transparent red</div>
```

### Border Style

| Class | CSS |
|-------|-----|
| `border-solid` | `border-style: solid` |
| `border-dashed` | `border-style: dashed` |
| `border-dotted` | `border-style: dotted` |
| `border-double` | `border-style: double` |
| `border-hidden` | `border-style: hidden` |
| `border-none` | `border-style: none` |

### Border Radius

| Class | CSS |
|-------|-----|
| `rounded-none` | `border-radius: 0` |
| `rounded-xs` | `0.125rem (2px)` |
| `rounded-sm` | `0.25rem (4px)` |
| `rounded-md` | `0.375rem (6px)` |
| `rounded-lg` | `0.5rem (8px)` |
| `rounded-xl` | `0.75rem (12px)` |
| `rounded-2xl` | `1rem (16px)` |
| `rounded-3xl` | `1.5rem (24px)` |
| `rounded-4xl` | `2rem (32px)` |
| `rounded-full` | `calc(infinity * 1px)` |

Side-specific: `rounded-t-*`, `rounded-r-*`, `rounded-b-*`, `rounded-l-*`
Corner-specific: `rounded-tl-*`, `rounded-tr-*`, `rounded-br-*`, `rounded-bl-*`
Logical: `rounded-s-*`, `rounded-e-*`, `rounded-ss-*`, `rounded-se-*`, `rounded-es-*`, `rounded-ee-*`

**v4 change:** `rounded` (no suffix) renamed to `rounded-sm`. Old `rounded-sm` is now `rounded-xs`.

### Ring

| Class | CSS |
|-------|-----|
| `ring` | `box-shadow: ... 1px ...` (was 3px in v3) |
| `ring-0` | No ring |
| `ring-1` | 1px ring |
| `ring-2` | 2px ring |
| `ring-3` | 3px ring (v3's `ring` default) |
| `ring-4` | 4px ring |
| `ring-8` | 8px ring |
| `ring-inset` | Inset ring |

Ring color: `ring-blue-500`, `ring-blue-500/50`

**v4 change:** `ring` is now 1px (was 3px) and uses `currentColor` (was `blue-500`).

### Outline

| Class | CSS |
|-------|-----|
| `outline` | `outline-style: solid; outline-width: 1px` |
| `outline-none` | `outline: 2px solid transparent` (v3 behavior) |
| `outline-hidden` | `outline-style: hidden` (new in v4) |
| `outline-dashed` | `outline-style: dashed` |
| `outline-dotted` | `outline-style: dotted` |
| `outline-double` | `outline-style: double` |
| `outline-offset-<n>` | `outline-offset: npx` |

### Divide

Adds borders between child elements:

```html
<div class="divide-y divide-gray-200">
  <div class="py-4">Item 1</div>
  <div class="py-4">Item 2</div>
  <div class="py-4">Item 3</div>
</div>
```

`divide-x-*`, `divide-y-*` for width. `divide-<color>` for color. `divide-dashed`, `divide-dotted` for style.

**v4 change:** `divide-y` now applies `border-bottom` to `:not(:last-child)` instead of `border-top` to `~ :not([hidden])`.

---

## 9. Effects

### Box Shadow

| Class | CSS |
|-------|-----|
| `shadow-2xs` | `0 1px rgb(0 0 0 / 0.05)` |
| `shadow-xs` | `0 1px 2px 0 rgb(0 0 0 / 0.05)` |
| `shadow-sm` | `0 1px 3px 0 rgb(0 0 0 / 0.1), ...` |
| `shadow-md` | `0 4px 6px -1px rgb(0 0 0 / 0.1), ...` |
| `shadow-lg` | `0 10px 15px -3px rgb(0 0 0 / 0.1), ...` |
| `shadow-xl` | `0 20px 25px -5px rgb(0 0 0 / 0.1), ...` |
| `shadow-2xl` | `0 25px 50px -12px rgb(0 0 0 / 0.25)` |
| `shadow-none` | `box-shadow: 0 0 #0000` |

Inset shadows: `inset-shadow-2xs`, `inset-shadow-xs`, `inset-shadow-sm`, `inset-shadow-none`

Shadow color: `shadow-red-500/50`

**v4 renames:** `shadow` -> `shadow-sm`, `shadow-sm` -> `shadow-xs`, `shadow-inner` -> `inset-shadow-sm`

### Opacity

| Class | CSS |
|-------|-----|
| `opacity-0` | `opacity: 0` |
| `opacity-5` | `opacity: 0.05` |
| `opacity-10` | `opacity: 0.1` |
| `opacity-25` | `opacity: 0.25` |
| `opacity-50` | `opacity: 0.5` |
| `opacity-75` | `opacity: 0.75` |
| `opacity-100` | `opacity: 1` |

Also supports arbitrary: `opacity-[0.67]`

### Mix Blend Mode

| Class | CSS |
|-------|-----|
| `mix-blend-normal` | `mix-blend-mode: normal` |
| `mix-blend-multiply` | `mix-blend-mode: multiply` |
| `mix-blend-screen` | `mix-blend-mode: screen` |
| `mix-blend-overlay` | `mix-blend-mode: overlay` |
| `mix-blend-darken` | `mix-blend-mode: darken` |
| `mix-blend-lighten` | `mix-blend-mode: lighten` |
| `mix-blend-color-dodge` | `mix-blend-mode: color-dodge` |
| `mix-blend-color-burn` | `mix-blend-mode: color-burn` |
| `mix-blend-hard-light` | `mix-blend-mode: hard-light` |
| `mix-blend-soft-light` | `mix-blend-mode: soft-light` |
| `mix-blend-difference` | `mix-blend-mode: difference` |
| `mix-blend-exclusion` | `mix-blend-mode: exclusion` |
| `mix-blend-hue` | `mix-blend-mode: hue` |
| `mix-blend-saturation` | `mix-blend-mode: saturation` |
| `mix-blend-color` | `mix-blend-mode: color` |
| `mix-blend-luminosity` | `mix-blend-mode: luminosity` |

---

## 10. Filters

### Blur

| Class | CSS |
|-------|-----|
| `blur-none` | `filter: blur(0)` |
| `blur-xs` | `filter: blur(2px)` |
| `blur-sm` | `filter: blur(4px)` |
| `blur-md` | `filter: blur(8px)` |
| `blur-lg` | `filter: blur(12px)` |
| `blur-xl` | `filter: blur(16px)` |
| `blur-2xl` | `filter: blur(24px)` |
| `blur-3xl` | `filter: blur(40px)` |

**v4 renames:** `blur` -> `blur-sm`, `blur-sm` -> `blur-xs`

### Brightness, Contrast, Saturate

`brightness-<n>`, `contrast-<n>`, `saturate-<n>` where n is 0, 50, 75, 100, 105, 110, 125, 150, 200

### Grayscale, Invert, Sepia

| Class | CSS |
|-------|-----|
| `grayscale` | `filter: grayscale(100%)` |
| `grayscale-0` | `filter: grayscale(0%)` |
| `invert` | `filter: invert(100%)` |
| `invert-0` | `filter: invert(0%)` |
| `sepia` | `filter: sepia(100%)` |
| `sepia-0` | `filter: sepia(0%)` |

### Drop Shadow

| Class | CSS |
|-------|-----|
| `drop-shadow-xs` | Small drop shadow |
| `drop-shadow-sm` | Medium drop shadow |
| `drop-shadow-md` | Large drop shadow |
| `drop-shadow-lg` | Larger drop shadow |
| `drop-shadow-xl` | Extra large drop shadow |
| `drop-shadow-2xl` | Huge drop shadow |
| `drop-shadow-none` | No drop shadow |

**v4 renames:** `drop-shadow` -> `drop-shadow-sm`, `drop-shadow-sm` -> `drop-shadow-xs`

### Backdrop Filter

All filter utilities have `backdrop-*` counterparts: `backdrop-blur-*`, `backdrop-brightness-*`, `backdrop-contrast-*`, `backdrop-grayscale`, `backdrop-invert`, `backdrop-saturate-*`, `backdrop-sepia`

**v4 renames:** `backdrop-blur` -> `backdrop-blur-sm`, `backdrop-blur-sm` -> `backdrop-blur-xs`

---

## 11. Transforms

### Scale

| Class | CSS |
|-------|-----|
| `scale-0` | `scale: 0` |
| `scale-50` | `scale: 50%` |
| `scale-75` | `scale: 75%` |
| `scale-90` | `scale: 90%` |
| `scale-95` | `scale: 95%` |
| `scale-100` | `scale: 100%` |
| `scale-105` | `scale: 105%` |
| `scale-110` | `scale: 110%` |
| `scale-125` | `scale: 125%` |
| `scale-150` | `scale: 150%` |
| `scale-none` | `scale: none` (v4 replaces `transform-none` for individual) |

Axis-specific: `scale-x-*`, `scale-y-*`
3D (v4 new): `scale-z-*`

### Rotate

| Class | CSS |
|-------|-----|
| `rotate-0` | `rotate: 0deg` |
| `rotate-1` | `rotate: 1deg` |
| `rotate-2` | `rotate: 2deg` |
| `rotate-3` | `rotate: 3deg` |
| `rotate-6` | `rotate: 6deg` |
| `rotate-12` | `rotate: 12deg` |
| `rotate-45` | `rotate: 45deg` |
| `rotate-90` | `rotate: 90deg` |
| `rotate-180` | `rotate: 180deg` |
| `-rotate-45` | `rotate: -45deg` |

3D (v4 new): `rotate-x-*`, `rotate-y-*`

### Translate

| Class | CSS |
|-------|-----|
| `translate-x-<n>` | `translate: calc(var(--spacing) * n) var(--tw-translate-y)` |
| `translate-y-<n>` | similar for Y axis |
| `translate-x-full` | `translate: 100%` |
| `translate-x-1/2` | `translate: 50%` |
| `-translate-x-<n>` | Negative values |

3D (v4 new): `translate-z-*`

### Skew

`skew-x-<n>`, `skew-y-<n>` where n is 0, 1, 2, 3, 6, 12

### Transform Origin

| Class | CSS |
|-------|-----|
| `origin-center` | `transform-origin: center` |
| `origin-top` | `transform-origin: top` |
| `origin-top-right` | `transform-origin: top right` |
| `origin-right` | `transform-origin: right` |
| `origin-bottom-right` | `transform-origin: bottom right` |
| `origin-bottom` | `transform-origin: bottom` |
| `origin-bottom-left` | `transform-origin: bottom left` |
| `origin-left` | `transform-origin: left` |
| `origin-top-left` | `transform-origin: top left` |

**v4 change:** Individual transform properties (`scale`, `rotate`, `translate`) are now native CSS properties, not composite `transform`. Use `scale-none`, `rotate-none`, `translate-none` instead of `transform-none`. For transitions, use `transition-[scale]` instead of `transition-transform`.

---

## 12. Transitions & Animation

### Transition Property

| Class | Properties |
|-------|-----------|
| `transition` | Colors, opacity, shadow, transform (with default 150ms ease) |
| `transition-all` | All properties |
| `transition-colors` | Color-related properties |
| `transition-opacity` | Opacity |
| `transition-shadow` | Box shadow |
| `transition-transform` | Transform, translate, scale, rotate |
| `transition-none` | No transition |

### Duration

| Class | CSS |
|-------|-----|
| `duration-0` | `transition-duration: 0ms` |
| `duration-75` | `transition-duration: 75ms` |
| `duration-100` | `transition-duration: 100ms` |
| `duration-150` | `transition-duration: 150ms` |
| `duration-200` | `transition-duration: 200ms` |
| `duration-300` | `transition-duration: 300ms` |
| `duration-500` | `transition-duration: 500ms` |
| `duration-700` | `transition-duration: 700ms` |
| `duration-1000` | `transition-duration: 1000ms` |

### Timing Function

| Class | CSS |
|-------|-----|
| `ease-linear` | `transition-timing-function: linear` |
| `ease-in` | `transition-timing-function: cubic-bezier(0.4, 0, 1, 1)` |
| `ease-out` | `transition-timing-function: cubic-bezier(0, 0, 0.2, 1)` |
| `ease-in-out` | `transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1)` |

### Delay

`delay-0`, `delay-75`, `delay-100`, `delay-150`, `delay-200`, `delay-300`, `delay-500`, `delay-700`, `delay-1000`

### Animation

| Class | Effect |
|-------|--------|
| `animate-spin` | Continuous rotation (360deg linear infinite) |
| `animate-ping` | Pulse scale + fade (for notifications) |
| `animate-pulse` | Gentle opacity pulse (for skeleton loaders) |
| `animate-bounce` | Bounce effect |
| `animate-none` | Remove animation |

Custom animations via `@theme`:

```css
@theme {
  --animate-fade-in: fade-in 0.3s ease-out;
  @keyframes fade-in {
    from { opacity: 0; }
    to { opacity: 1; }
  }
}
```

```html
<div class="animate-fade-in">Fades in</div>
```

### Motion Preference

```html
<button class="transition hover:scale-110 motion-reduce:transition-none motion-reduce:hover:transform-none">
  Respects reduced motion
</button>
```

### Transition Example

```html
<button class="bg-blue-500 transition duration-300 ease-in-out hover:-translate-y-1 hover:scale-110 hover:bg-indigo-500">
  Save Changes
</button>
```

---

## 13. Interactivity

### Cursor

| Class | CSS |
|-------|-----|
| `cursor-auto` | `cursor: auto` |
| `cursor-default` | `cursor: default` |
| `cursor-pointer` | `cursor: pointer` |
| `cursor-wait` | `cursor: wait` |
| `cursor-text` | `cursor: text` |
| `cursor-move` | `cursor: move` |
| `cursor-help` | `cursor: help` |
| `cursor-not-allowed` | `cursor: not-allowed` |
| `cursor-none` | `cursor: none` |
| `cursor-grab` | `cursor: grab` |
| `cursor-grabbing` | `cursor: grabbing` |

**v4 change:** Buttons default to `cursor: default` (browser default), not `cursor: pointer`. Add `cursor-pointer` explicitly.

### Pointer Events

| Class | CSS |
|-------|-----|
| `pointer-events-none` | `pointer-events: none` |
| `pointer-events-auto` | `pointer-events: auto` |

### Resize

| Class | CSS |
|-------|-----|
| `resize-none` | `resize: none` |
| `resize` | `resize: both` |
| `resize-x` | `resize: horizontal` |
| `resize-y` | `resize: vertical` |

### Scroll Behavior

| Class | CSS |
|-------|-----|
| `scroll-auto` | `scroll-behavior: auto` |
| `scroll-smooth` | `scroll-behavior: smooth` |

### User Select

| Class | CSS |
|-------|-----|
| `select-none` | `user-select: none` |
| `select-text` | `user-select: text` |
| `select-all` | `user-select: all` |
| `select-auto` | `user-select: auto` |

### Touch Action

`touch-auto`, `touch-none`, `touch-pan-x`, `touch-pan-y`, `touch-pan-left`, `touch-pan-right`, `touch-pan-up`, `touch-pan-down`, `touch-pinch-zoom`, `touch-manipulation`

### Scroll Snap

`snap-start`, `snap-end`, `snap-center`, `snap-align-none`, `snap-normal`, `snap-always`

Container: `snap-none`, `snap-x`, `snap-y`, `snap-both`, `snap-mandatory`, `snap-proximity`

### Appearance

`appearance-none` removes native styling from form elements.

### Accent Color

`accent-<color>` sets the accent color for checkboxes, radio buttons, range sliders.

### Caret Color

`caret-<color>` sets the text cursor color in inputs.

---

## 14. Responsive Breakpoints

### Default Breakpoints

| Prefix | Min Width | CSS Media Query |
|--------|-----------|-----------------|
| `sm` | 40rem (640px) | `@media (width >= 40rem)` |
| `md` | 48rem (768px) | `@media (width >= 48rem)` |
| `lg` | 64rem (1024px) | `@media (width >= 64rem)` |
| `xl` | 80rem (1280px) | `@media (width >= 80rem)` |
| `2xl` | 96rem (1536px) | `@media (width >= 96rem)` |

### Max-Width Variants

| Prefix | Max Width |
|--------|-----------|
| `max-sm` | `@media (width < 40rem)` |
| `max-md` | `@media (width < 48rem)` |
| `max-lg` | `@media (width < 64rem)` |
| `max-xl` | `@media (width < 80rem)` |
| `max-2xl` | `@media (width < 96rem)` |

### Range Targeting

```html
<div class="md:max-lg:flex">Only flex between md and lg</div>
```

### Custom Breakpoints

```css
@theme {
  --breakpoint-xs: 30rem;
  --breakpoint-3xl: 120rem;
}
```

Replace all breakpoints:

```css
@theme {
  --breakpoint-*: initial;
  --breakpoint-tablet: 40rem;
  --breakpoint-laptop: 64rem;
  --breakpoint-desktop: 80rem;
}
```

### Arbitrary Breakpoints

```html
<div class="min-[320px]:text-center max-[600px]:bg-sky-300">One-off breakpoint</div>
```

### Container Queries (v4)

```html
<div class="@container">
  <div class="flex flex-col @md:flex-row">
    Container-width responsive
  </div>
</div>
```

---

## 15. State Variants

### Interactive

| Variant | Selector |
|---------|----------|
| `hover:` | `:hover` (v4: only on devices with hover support via `@media (hover: hover)`) |
| `focus:` | `:focus` |
| `focus-within:` | `:focus-within` |
| `focus-visible:` | `:focus-visible` |
| `active:` | `:active` |
| `visited:` | `:visited` |
| `target:` | `:target` |

### Structural

| Variant | Selector |
|---------|----------|
| `first:` | `:first-child` |
| `last:` | `:last-child` |
| `only:` | `:only-child` |
| `odd:` | `:nth-child(odd)` |
| `even:` | `:nth-child(even)` |
| `first-of-type:` | `:first-of-type` |
| `last-of-type:` | `:last-of-type` |
| `empty:` | `:empty` |
| `nth-[3]:` | `:nth-child(3)` |
| `nth-[3n+1]:` | `:nth-child(3n+1)` |

### Form States

| Variant | Selector |
|---------|----------|
| `disabled:` | `:disabled` |
| `enabled:` | `:enabled` |
| `checked:` | `:checked` |
| `indeterminate:` | `:indeterminate` |
| `required:` | `:required` |
| `optional:` | `:optional` |
| `valid:` | `:valid` |
| `invalid:` | `:invalid` |
| `user-valid:` | `:user-valid` |
| `user-invalid:` | `:user-invalid` |
| `in-range:` | `:in-range` |
| `out-of-range:` | `:out-of-range` |
| `placeholder-shown:` | `:placeholder-shown` |
| `autofill:` | `:autofill` |
| `read-only:` | `:read-only` |

### Group & Peer

```html
<!-- Group: style children based on parent state -->
<a href="#" class="group rounded-lg p-5 hover:bg-sky-500">
  <h3 class="text-gray-900 group-hover:text-white">Title</h3>
  <p class="text-gray-500 group-hover:text-white">Description</p>
</a>

<!-- Named groups for nesting -->
<li class="group/item">
  <a class="group/edit invisible group-hover/item:visible">Edit</a>
</li>

<!-- Peer: style element based on sibling state -->
<input type="email" class="peer" />
<p class="invisible peer-invalid:visible text-red-500">Invalid email</p>

<!-- Named peers -->
<input id="draft" class="peer/draft" type="radio" />
<label class="peer-checked/draft:text-sky-500">Draft</label>
```

### has-* and not-*

```html
<!-- has: parent has matching descendant -->
<label class="has-checked:bg-indigo-50 has-checked:ring-indigo-500">
  <input type="radio" />
</label>

<!-- not: negation -->
<button class="hover:not-focus:bg-indigo-700">Button</button>
```

### Pseudo-Elements

| Variant | Pseudo-element |
|---------|---------------|
| `before:` | `::before` (auto `content: ''`) |
| `after:` | `::after` (auto `content: ''`) |
| `placeholder:` | `::placeholder` |
| `file:` | `::file-selector-button` |
| `marker:` | `::marker` |
| `selection:` | `::selection` |
| `first-line:` | `::first-line` |
| `first-letter:` | `::first-letter` |
| `backdrop:` | `::backdrop` |

### Child Selection

```html
<!-- v4: left-to-right order -->
<ul class="*:first:pt-0 *:last:pb-0">
  <li>Styled children</li>
</ul>
```

### Media/Feature Variants

| Variant | Media Query |
|---------|-------------|
| `dark:` | `prefers-color-scheme: dark` |
| `motion-reduce:` | `prefers-reduced-motion: reduce` |
| `motion-safe:` | `prefers-reduced-motion: no-preference` |
| `contrast-more:` | `prefers-contrast: more` |
| `contrast-less:` | `prefers-contrast: less` |
| `portrait:` | `orientation: portrait` |
| `landscape:` | `orientation: landscape` |
| `print:` | `@media print` |

### Open State

```html
<details class="open:bg-white open:shadow-lg open:ring-1">
  <summary>Details</summary>
  <div>Content</div>
</details>
```

---

## 16. Dark Mode

### Default: System Preference (Media Strategy)

Works automatically with `prefers-color-scheme`:

```html
<div class="bg-white dark:bg-slate-900">
  <h1 class="text-slate-900 dark:text-white">Title</h1>
</div>
```

### Manual Toggle: Class Strategy

```css
/* app.css */
@import "tailwindcss";
@custom-variant dark (&:where(.dark, .dark *));
```

```html
<html class="dark">
  <body>
    <div class="bg-white dark:bg-black">Adapts to class</div>
  </body>
</html>
```

### Data Attribute Strategy

```css
@import "tailwindcss";
@custom-variant dark (&:where([data-theme=dark], [data-theme=dark] *));
```

```html
<html data-theme="dark">...</html>
```

### Toggle Script (Light/Dark/System)

```javascript
document.documentElement.classList.toggle(
  "dark",
  localStorage.theme === "dark" ||
    (!("theme" in localStorage) &&
      window.matchMedia("(prefers-color-scheme: dark)").matches)
);

// Set preference
localStorage.theme = "light";   // Force light
localStorage.theme = "dark";    // Force dark
localStorage.removeItem("theme"); // Follow system
```

---

## 17. Customization

### v4: CSS-First Configuration with @theme

```css
@import "tailwindcss";

@theme {
  /* Custom colors */
  --color-brand: #3f3cbb;
  --color-brand-light: #6366f1;

  /* Custom fonts */
  --font-display: "Satoshi", sans-serif;

  /* Custom breakpoints */
  --breakpoint-3xl: 120rem;

  /* Custom spacing base */
  --spacing: 0.25rem;

  /* Custom border radius */
  --radius-card: 1rem;

  /* Custom shadows */
  --shadow-card: 0 2px 8px rgb(0 0 0 / 0.08);

  /* Custom animations */
  --animate-fade-in: fade-in 0.3s ease-out;
  @keyframes fade-in {
    from { opacity: 0; transform: scale(0.95); }
    to { opacity: 1; transform: scale(1); }
  }
}
```

### Theme Namespaces

| Namespace | Generated Utilities |
|-----------|-------------------|
| `--color-*` | `bg-*`, `text-*`, `border-*`, `fill-*`, `stroke-*` |
| `--font-*` | `font-*` |
| `--text-*` | `text-*` (sizes) |
| `--font-weight-*` | `font-*` (weights) |
| `--tracking-*` | `tracking-*` |
| `--leading-*` | `leading-*` |
| `--breakpoint-*` | `sm:`, `md:`, `lg:`, etc. |
| `--spacing` | All spacing/sizing utilities |
| `--radius-*` | `rounded-*` |
| `--shadow-*` | `shadow-*` |
| `--inset-shadow-*` | `inset-shadow-*` |
| `--animate-*` | `animate-*` |
| `--ease-*` | `ease-*` |

### Replacing Defaults

```css
@theme {
  --color-*: initial;        /* Remove all default colors */
  --color-white: #fff;       /* Add only what you need */
  --color-primary: #3f3cbb;
}
```

```css
@theme {
  --*: initial;  /* Remove ALL defaults for complete custom theme */
}
```

### Custom Utilities (v4)

```css
/* v3: @layer utilities { .tab-4 { tab-size: 4; } } */
/* v4: */
@utility tab-4 {
  tab-size: 4;
}
```

### Custom Variants (v4)

```css
@custom-variant theme-midnight (&:where([data-theme=midnight], [data-theme=midnight] *));
```

```html
<div class="theme-midnight:bg-slate-900">Custom variant</div>
```

### Legacy Config Support

v4 still supports `tailwind.config.js` but it is **not auto-detected**:

```css
@import "tailwindcss";
@config "../../tailwind.config.js";
```

### Source Control

```css
/* Equivalent of v3's safelist */
@source inline("underline", "bg-red-500", "lg:flex");

/* Scan additional paths */
@source "../node_modules/my-ui-lib/src/**/*.js";
```

### @apply and @reference

```css
/* In component CSS */
.btn {
  @apply rounded-lg px-4 py-2 font-semibold;
}
```

For scoped styles (Vue/Svelte):

```vue
<style>
  @reference "../../app.css";
  h1 { @apply text-2xl font-bold text-red-500; }
</style>
```

### Using CSS Variables Directly

Theme variables are available as standard CSS custom properties:

```css
.custom-component {
  font-size: var(--text-base);
  color: var(--color-gray-700);
  border-radius: var(--radius-lg);
}
```

```javascript
const styles = getComputedStyle(document.documentElement);
const color = styles.getPropertyValue("--color-blue-500");
```

### Prefix

```css
@import "tailwindcss" prefix(tw);
```

```html
<div class="tw:flex tw:bg-red-500 tw:hover:bg-red-600">Prefixed</div>
```

---

## 18. V3 to V4 Migration

### Key Differences

| Feature | v3 | v4 |
|---------|----|----|
| Config | `tailwind.config.js` | `@theme` in CSS |
| Import | `@tailwind base/components/utilities` | `@import "tailwindcss"` |
| Content detection | Manual `content: [...]` | Automatic |
| Purge/safelist | Config-based | `@source` directive |
| Custom utilities | `@layer utilities { }` | `@utility name { }` |
| Custom variants | Plugin API | `@custom-variant` |
| PostCSS plugin | `tailwindcss` | `@tailwindcss/postcss` |
| CLI | `npx tailwindcss` | `npx @tailwindcss/cli` |
| Important | `!flex` | `flex!` |
| Arbitrary CSS vars | `bg-[--var]` | `bg-(--var)` |
| Variant order | Right to left | Left to right |
| Preprocessors | Sass/Less supported | Not supported |

### Renamed Utilities

| v3 | v4 |
|----|----|
| `shadow-sm` | `shadow-xs` |
| `shadow` | `shadow-sm` |
| `shadow-inner` | `inset-shadow-sm` |
| `drop-shadow-sm` | `drop-shadow-xs` |
| `drop-shadow` | `drop-shadow-sm` |
| `blur-sm` | `blur-xs` |
| `blur` | `blur-sm` |
| `rounded-sm` | `rounded-xs` |
| `rounded` | `rounded-sm` |
| `outline-none` | `outline-hidden` |
| `ring` | `ring-3` (default changed to 1px) |
| `bg-gradient-to-*` | `bg-linear-to-*` |

### Removed Utilities (Use Opacity Modifiers)

| Removed | Replacement |
|---------|-------------|
| `bg-opacity-50` | `bg-black/50` |
| `text-opacity-50` | `text-black/50` |
| `border-opacity-50` | `border-black/50` |
| `flex-shrink-*` | `shrink-*` |
| `flex-grow-*` | `grow-*` |
| `overflow-ellipsis` | `text-ellipsis` |
| `decoration-slice` | `box-decoration-slice` |
| `decoration-clone` | `box-decoration-clone` |

### Default Value Changes

| Property | v3 | v4 |
|----------|----|----|
| Border color | `gray-200` | `currentColor` |
| Ring width | 3px | 1px |
| Ring color | `blue-500` | `currentColor` |
| Button cursor | `pointer` | `default` |
| Placeholder color | `gray-400` | Current text at 50% opacity |

### Removed Config Options

- `corePlugins` - no longer available
- `safelist` - use `@source inline()` instead
- `separator` - no longer configurable
- `resolveConfig()` - use CSS variables instead

### Upgrade Tool

```bash
npx @tailwindcss/upgrade
```

Requires Node.js 20+. Review the diff before committing.

---

## 19. Common Component Patterns

### Button

```html
<!-- Primary -->
<button class="rounded-lg bg-blue-600 px-4 py-2 text-sm font-semibold text-white shadow-sm hover:bg-blue-700 focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-blue-600 active:bg-blue-800 disabled:cursor-not-allowed disabled:opacity-50">
  Primary Button
</button>

<!-- Secondary / Outline -->
<button class="rounded-lg border border-gray-300 bg-white px-4 py-2 text-sm font-semibold text-gray-700 shadow-sm hover:bg-gray-50 dark:border-gray-600 dark:bg-gray-800 dark:text-gray-200 dark:hover:bg-gray-700">
  Secondary Button
</button>

<!-- Ghost -->
<button class="rounded-lg px-4 py-2 text-sm font-semibold text-gray-700 hover:bg-gray-100 dark:text-gray-200 dark:hover:bg-gray-800">
  Ghost Button
</button>
```

### Card

```html
<div class="overflow-hidden rounded-xl bg-white shadow-md dark:bg-gray-800">
  <img class="h-48 w-full object-cover" src="image.jpg" alt="" />
  <div class="p-6">
    <p class="text-sm font-semibold uppercase tracking-wide text-indigo-500">Category</p>
    <h3 class="mt-1 text-lg font-medium text-gray-900 dark:text-white">Card Title</h3>
    <p class="mt-2 text-gray-500 dark:text-gray-400">Card description goes here.</p>
  </div>
</div>
```

### Badge

```html
<span class="inline-flex items-center rounded-full bg-green-100 px-2.5 py-0.5 text-xs font-medium text-green-800">
  Active
</span>
<span class="inline-flex items-center rounded-full bg-red-100 px-2.5 py-0.5 text-xs font-medium text-red-800">
  Error
</span>
<span class="inline-flex items-center rounded-full bg-yellow-100 px-2.5 py-0.5 text-xs font-medium text-yellow-800">
  Warning
</span>
```

### Alert

```html
<!-- Info -->
<div class="rounded-lg border border-blue-200 bg-blue-50 p-4 dark:border-blue-800 dark:bg-blue-900/50">
  <div class="flex items-center gap-3">
    <svg class="size-5 shrink-0 text-blue-600" fill="currentColor" viewBox="0 0 20 20">...</svg>
    <p class="text-sm text-blue-700 dark:text-blue-300">This is an informational message.</p>
  </div>
</div>

<!-- Error -->
<div class="rounded-lg border border-red-200 bg-red-50 p-4">
  <p class="text-sm text-red-700">Something went wrong. Please try again.</p>
</div>
```

### Form Input

```html
<div>
  <label class="mb-1 block text-sm font-medium text-gray-700 dark:text-gray-300" for="email">
    Email
  </label>
  <input
    type="email"
    id="email"
    class="block w-full rounded-lg border border-gray-300 bg-white px-3 py-2 text-sm text-gray-900 shadow-sm placeholder:text-gray-400 focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500 disabled:cursor-not-allowed disabled:bg-gray-50 disabled:text-gray-500 dark:border-gray-600 dark:bg-gray-800 dark:text-white dark:placeholder:text-gray-500"
    placeholder="you@example.com"
  />
</div>
```

### Select

```html
<select class="block w-full rounded-lg border border-gray-300 bg-white px-3 py-2 text-sm text-gray-900 shadow-sm focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-800 dark:text-white">
  <option>Option 1</option>
  <option>Option 2</option>
</select>
```

### Checkbox

```html
<label class="flex items-center gap-2">
  <input type="checkbox" class="size-4 rounded border-gray-300 text-blue-600 focus:ring-blue-500" />
  <span class="text-sm text-gray-700 dark:text-gray-300">Remember me</span>
</label>
```

### Navbar

```html
<nav class="border-b border-gray-200 bg-white dark:border-gray-700 dark:bg-gray-900">
  <div class="mx-auto flex max-w-7xl items-center justify-between px-4 py-3">
    <a href="/" class="text-xl font-bold text-gray-900 dark:text-white">Logo</a>
    <div class="hidden items-center gap-6 md:flex">
      <a href="#" class="text-sm font-medium text-gray-700 hover:text-gray-900 dark:text-gray-300 dark:hover:text-white">Home</a>
      <a href="#" class="text-sm font-medium text-gray-500 hover:text-gray-900 dark:text-gray-400 dark:hover:text-white">About</a>
      <a href="#" class="text-sm font-medium text-gray-500 hover:text-gray-900 dark:text-gray-400 dark:hover:text-white">Contact</a>
      <button class="rounded-lg bg-blue-600 px-4 py-2 text-sm font-semibold text-white hover:bg-blue-700">Sign Up</button>
    </div>
  </div>
</nav>
```

### Modal / Dialog

```html
<!-- Backdrop -->
<div class="fixed inset-0 z-50 flex items-center justify-center bg-black/50 backdrop-blur-sm">
  <!-- Modal -->
  <div class="mx-4 w-full max-w-md rounded-xl bg-white p-6 shadow-xl dark:bg-gray-800">
    <h2 class="text-lg font-semibold text-gray-900 dark:text-white">Modal Title</h2>
    <p class="mt-2 text-sm text-gray-500 dark:text-gray-400">Modal content goes here.</p>
    <div class="mt-6 flex justify-end gap-3">
      <button class="rounded-lg border border-gray-300 px-4 py-2 text-sm font-medium text-gray-700 hover:bg-gray-50 dark:border-gray-600 dark:text-gray-300 dark:hover:bg-gray-700">Cancel</button>
      <button class="rounded-lg bg-blue-600 px-4 py-2 text-sm font-semibold text-white hover:bg-blue-700">Confirm</button>
    </div>
  </div>
</div>
```

### Avatar Group

```html
<div class="flex -space-x-2">
  <img class="inline-block size-8 rounded-full ring-2 ring-white" src="avatar1.jpg" alt="" />
  <img class="inline-block size-8 rounded-full ring-2 ring-white" src="avatar2.jpg" alt="" />
  <img class="inline-block size-8 rounded-full ring-2 ring-white" src="avatar3.jpg" alt="" />
  <span class="inline-flex size-8 items-center justify-center rounded-full bg-gray-200 text-xs font-medium text-gray-600 ring-2 ring-white">+5</span>
</div>
```

### Responsive Grid Layout

```html
<div class="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
  <div class="rounded-lg bg-white p-4 shadow">Card 1</div>
  <div class="rounded-lg bg-white p-4 shadow">Card 2</div>
  <div class="rounded-lg bg-white p-4 shadow">Card 3</div>
  <div class="rounded-lg bg-white p-4 shadow">Card 4</div>
</div>
```

### Centered Page Layout

```html
<div class="mx-auto min-h-screen max-w-7xl px-4 sm:px-6 lg:px-8">
  <main class="py-8">
    <!-- Content -->
  </main>
</div>
```

### Sticky Footer Layout

```html
<div class="flex min-h-screen flex-col">
  <header class="border-b px-4 py-3">Header</header>
  <main class="flex-1 px-4 py-8">Content</main>
  <footer class="border-t px-4 py-3">Footer</footer>
</div>
```

### Skeleton Loader

```html
<div class="animate-pulse space-y-4">
  <div class="h-4 w-3/4 rounded bg-gray-200 dark:bg-gray-700"></div>
  <div class="h-4 rounded bg-gray-200 dark:bg-gray-700"></div>
  <div class="h-4 w-5/6 rounded bg-gray-200 dark:bg-gray-700"></div>
</div>
```

### Toggle / Switch

```html
<label class="relative inline-flex cursor-pointer items-center">
  <input type="checkbox" class="peer sr-only" />
  <div class="h-6 w-11 rounded-full bg-gray-200 after:absolute after:left-[2px] after:top-[2px] after:size-5 after:rounded-full after:bg-white after:shadow after:transition-all peer-checked:bg-blue-600 peer-checked:after:translate-x-full peer-focus:ring-2 peer-focus:ring-blue-300 dark:bg-gray-700"></div>
</label>
```

---

## Quick Reference: Color Opacity Syntax

```html
bg-blue-500/75      /* 75% opacity */
text-white/50       /* 50% opacity */
border-red-500/25   /* 25% opacity */
ring-indigo-500/80  /* 80% opacity */
```

## Quick Reference: Arbitrary Values

```html
w-[200px]           /* Fixed width */
h-[calc(100vh-4rem)]/* Calculated height */
bg-[#1da1f2]        /* Hex color */
grid-cols-[1fr_2fr] /* Custom grid */
text-[clamp(1rem,2vw,2rem)] /* Clamp */
top-[var(--offset)] /* CSS variable */
```

## Quick Reference: Size Scale

| Class | Size |
|-------|------|
| `*-3xs` | 16rem (256px) |
| `*-2xs` | 18rem (288px) |
| `*-xs` | 20rem (320px) |
| `*-sm` | 24rem (384px) |
| `*-md` | 28rem (448px) |
| `*-lg` | 32rem (512px) |
| `*-xl` | 36rem (576px) |
| `*-2xl` | 42rem (672px) |
| `*-3xl` | 48rem (768px) |
| `*-4xl` | 56rem (896px) |
| `*-5xl` | 64rem (1024px) |
| `*-6xl` | 72rem (1152px) |
| `*-7xl` | 80rem (1280px) |

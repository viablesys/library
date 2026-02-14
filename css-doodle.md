# css-doodle Reference

Web component for drawing patterns and art with CSS. Uses Shadow DOM v1 + Custom Elements v1.

## Setup

```html
<!-- CDN -->
<script src="https://esm.sh/css-doodle/css-doodle.min.js?raw"></script>

<!-- ES Module -->
<script type="module">
  import 'https://esm.sh/css-doodle'
</script>
```

```bash
npm install css-doodle
```

## Basic Usage

```html
<css-doodle>
  :doodle {
    @grid: 8 / 8em;
    gap: 1px;
  }
  background: hsl(@r(360), 70%, 70%);
</css-doodle>
```

The component generates a grid of divs. Rules inside are plain CSS extended with `@`-prefixed selectors, properties, and functions.

---

## Grid

| Syntax | Meaning |
|--------|---------|
| `@grid: 5` | 5x5 grid |
| `@grid: 5x7` | 5 cols, 7 rows |
| `@grid: 8 / 8em` | 8x8, each cell 8em |
| `grid="5"` attribute | Same as @grid: 5 |

Range: 1-64 per dimension (2D), up to 4,096 cells (1D).

---

## Selectors

| Selector | Target |
|----------|--------|
| `:doodle` | The component itself (set @grid, width, etc.) |
| `:container` | Grid Layout container holding all cells |
| `@nth(n, ...)` | nth cell(s), like :nth-child() |
| `@even` / `@odd` | Even/odd indexed cells |
| `@at(col, row)` | Cell at specific grid position |
| `@random([ratio])` | Random cells (ratio 0-1, default 0.5) |
| `@row(n, ...)` | Cells in row(s) |
| `@col(n, ...)` | Cells in column(s) |
| `@match(expr)` | Math-conditional selection |
| `@cell(random)` | Random cell targeting |

### @match variables

| Var | Meaning | Range |
|-----|---------|-------|
| `i` | Cell index (1-based) | 1..I |
| `x` | Column number | 1..X |
| `y` | Row number | 1..Y |
| `I` | Total cells | - |
| `X` | Total columns | - |
| `Y` | Total rows | - |

```css
@match(x == y) { /* diagonal */ }
@match(i <= 8 || i >= 18) { /* first 8 + last from 18 */ }
```

---

## Properties

| Property | Description |
|----------|-------------|
| `@size: 10em` | width + height (square) |
| `@size: 8em 6em` | width height (rect) |
| `@place: center` | Position relative to grid |
| `@place: 75% 80%` | x% y% positioning |
| `@shape: circle` | Clip-path shape |
| `@shape: hypocycloid 4` | Parametric shape with arg |
| `@content: "text"` | Insert text node |

---

## Index & Position Functions

| Function | Alias | Returns |
|----------|-------|---------|
| `@index()` | `@i` | Current cell index (1-based) |
| `@row()` | `@y` | Current row number |
| `@col()` | `@x` | Current column number |
| `@size-row()` | `@Y` | Max rows |
| `@size-col()` | `@X` | Max columns |
| `@size()` | `@I` | Total cell count |

---

## Random Functions

| Function | Alias | Description |
|----------|-------|-------------|
| `@rand(start, end)` | `@r` | Random number in range |
| `@pick(v1, v2, ...)` | `@p` | Random pick from list |
| `@pick-n(...)` | `@pn` | Sequential picking |
| `@pick-d(...)` | `@pd` | Distinct random order |
| `@Pick(...)` | `@P` | Pick, guaranteed different from last |
| `@last-pick()` | `@lp` | Last picked value |
| `@last-rand()` | `@lr` | Last random value |
| `@rn()` | - | Perlin noise-based random |

### Examples

```css
background: hsl(@r(360), 70%, 70%);
color: @p(red, blue, green, gold);
border-radius: @r(10, 50)%;
```

---

## Repetition Functions

| Function | Alias | Join | Description |
|----------|-------|------|-------------|
| `@repeat(n, val)` | `@rep` | none | Repeat value n times |
| `@multiple(n, val)` | `@m` | `,` | Repeat comma-separated |
| `@Multiple(n, val)` | `@M` | ` ` | Repeat space-separated |

### Iteration variables inside @m/@rep

| Var | Meaning |
|-----|---------|
| `@n` | Current iteration (1-based) |
| `@nx` | Current column iteration |
| `@ny` | Current row iteration |
| `@N` | Total iterations |

### Example: multiple box-shadows

```css
box-shadow: @m(10,
  @r(-bg, bg) @r(-bg, bg) 0 @r(1, 5)px
  @p(#f44, #4af, #fa4, #4f4)
);
```

---

## Math Functions

All JS Math functions available with `@` prefix:

`@sin` `@cos` `@tan` `@abs` `@sqrt` `@pow` `@floor` `@ceil` `@round`
`@log` `@max` `@min` `@PI` `@random`

```css
transform: rotate(@calc(@i * 360 / @I)deg);
opacity: @calc(@sin(@i / @I * @PI));
```

---

## Utility Functions

| Function | Description |
|----------|-------------|
| `@calc(expr)` | Evaluate math expressions |
| `@var(expr)` | CSS variable wrapper (prevents evaluation) |
| `@hex(num)` | Number to hex string |
| `@stripe(c1 [sz], c2 [sz], ...)` | Generate gradient stripes |
| `@google-font("Name")` | Load Google Font |

---

## SVG Functions

### @svg()

Embed SVG as background-image. Supports CSS-like syntax for SVG elements.

```css
background: @svg(
  viewBox: 0 0 100 100;
  circle {
    cx, cy, r: 50;
    fill: none;
    stroke: currentColor;
    stroke-width: 2;
  }
);
```

### @svg-filter()

Apply SVG filter effects:

```css
filter: @svg-filter(
  feTurbulence {
    type: fractalNoise;
    baseFrequency: .01;
    numOctaves: 5;
  }
  feDisplacementMap {
    in: SourceGraphic;
    scale: 50;
  }
);
```

---

## @doodle() — Nested Doodles

Generate a background-image from nested css-doodle rules:

```css
background: @doodle(
  @grid: 4 / 100%;
  background: @p(#f44, #4af, #fa4);
  border-radius: @p(0, 50%);
);
```

---

## @shaders() — GLSL

Use GLSL fragment shaders as background:

```css
background: @shaders(
  void main() {
    vec2 uv = gl_FragCoord.xy / u_resolution.xy;
    gl_FragColor = vec4(uv, 0.5, 1.0);
  }
);
```

Built-in uniforms: `u_resolution`, `u_time`. Textures: `texture_0`, `texture_1`, etc.

---

## @shape() — Custom Shapes

Generate clip-path polygons with math:

```css
clip-path: @shape(
  points: 200;
  r: cos(4t);       /* polar equation — rose curve */
  rotate: 45;
  scale: .8;
);
```

### Commands

| Command | Description |
|---------|-------------|
| `points: N` | Number of polygon vertices |
| `r: expr` | Polar radius as function of `t` (0..2PI) |
| `rotate: deg` | Rotation in degrees |
| `scale: n` | Scale factor |
| `move: x y` | Translation |
| `frame` | Add bounding frame |

### Built-in shapes (for @shape property)

`circle` `triangle` `rhombus` `pentagon` `hexagon` `heptagon` `octagon`
`star` `diamond` `cross` `clover` `hypocycloid` `astroid` `infinity`
`heart` `fish` `drop` `pear` `bicorn` `bean` `bud` `wave`

---

## @plot() — Positioning via Equations

Place cells using parametric equations:

```css
@place: @plot(
  x: cos(t) * 40% + 50%;
  y: sin(t) * 40% + 50%;
);
```

---

## Attributes

| Attribute | Description |
|-----------|-------------|
| `grid="5x7"` | Set grid dimensions |
| `seed="2020"` | Reproducible randomization |
| `use="var(--rule)"` | Import rules from CSS custom property |
| `click:update` | Re-render on click |
| `auto:update` | Auto re-render periodically |

---

## JS API

```js
const d = document.querySelector('css-doodle');

d.grid = "5";                    // set grid
d.seed = "abc";                  // set seed
d.update(`@grid: 6 / 8em;`);    // re-render with new rules
d.update();                      // refresh (re-rolls random)

// Export as image
await d.export({ scale: 4, download: true, name: 'art' });
```

---

## @use — Production Pattern

Store rules in CSS custom properties for stability:

```html
<style>
  :root {
    --pattern: (
      @grid: 10 / 100%;
      background: @p(coral, teal, gold);
    );
  }
</style>
<css-doodle use="var(--pattern)"></css-doodle>
```

---

## Common Patterns

### Gradient grid

```html
<css-doodle>
  :doodle { @grid: 12 / 80vmin; }
  background: hsl(
    calc(180 + @x * 10 + @y * 10),
    70%, 65%
  );
  transition: .2s;
</css-doodle>
```

### Random circles

```html
<css-doodle>
  :doodle { @grid: 8 / 60vmin; gap: 2px; }
  @shape: circle;
  background: @p(#f44336, #2196f3, #ff9800, #4caf50);
  opacity: @r(.3, 1);
  transform: scale(@r(.5, 1));
</css-doodle>
```

### Animated pattern

```html
<css-doodle click:update>
  :doodle { @grid: 10 / 80vmin; }
  @size: @r(40%, 100%);
  background: @p(
    #264653, #2a9d8f, #e9c46a, #f4a261, #e76f51
  );
  border-radius: @pick(0, 0, 0, 50%);
  transform: rotate(@r(360)deg) scale(@r(.8, 1.2));
  transition: .4s ease;
</css-doodle>
```

### Shadow art with @m

```html
<css-doodle>
  :doodle { @grid: 1 / 80vmin; }
  @size: 4px;
  border-radius: 50%;
  box-shadow: @m(200,
    @r(-40vmin, 40vmin) @r(-40vmin, 40vmin)
    @r(1, 8)px
    @p(#f44, #4af, #fa4, #4f4, #a4f)
  );
</css-doodle>
```

### Nested doodle background

```html
<css-doodle>
  :doodle { @grid: 6 / 90vmin; gap: 1px; }
  background: @doodle(
    @grid: 3 / 100%;
    background: @p(#000, #fff, @lp);
  );
</css-doodle>
```

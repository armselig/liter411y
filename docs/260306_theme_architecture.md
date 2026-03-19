# CSS Custom Properties Architecture

## Context

The library currently has global design tokens in `assets/styles/_theme.css` but components
expose no styling API. Users can't restyle components because Lit's Shadow DOM encapsulates all
styles. CSS custom properties are the standard solution: they **pierce Shadow DOM** and inherit
from `:root` into shadow roots, making them the ideal styling API for Web Components.

The goal is a layered approach that:
1. "Just works" — sensible defaults at the component level, no CSS import needed
2. Optional baseline — an importable stylesheet that wires component properties to global tokens
3. Self-documenting — the CSS file itself is the reference for available properties
4. Easily overridable — standard CSS cascade, no specificity tricks required

---

## CSS Architecture

### 3-level cascade (outermost wins):

```
User CSS (unlayered, always wins)
    ↓ overrides ↓
assets/styles/_components.css  (@layer components — sets component defaults via :root)
    ↓ overrides ↓
assets/styles/_theme.css        (@layer theme — global color/spacing tokens)
    ↓ overrides ↓
Component's static styles       (Shadow DOM internal defaults — fallback chain)
```

### Naming conventions:
- **Public API** (user-facing): `--asl-[component]-[property]`  e.g. `--asl-example-color`
- **Internal/private** (inside shadow DOM only): `--_[property]`  e.g. `--_color`

The internal variables map external API properties → internal usage with fallback chains:

```css
/* inside component static styles */
:host {
    --_color: var(--asl-example-color, var(--asl-c-primary, oklch(75% 0.087 247)));
}
span {
    color: var(--_color);
}
```

This creates a clean single place (`:host {}`) to see all configurable properties per component.

---

## Files to Create / Modify

### 1. `components/example/example.js` — Demonstrate the pattern

Add `css` to the Lit import and add `static styles`:

```js
import { html, css } from "lit";
import { AslBase } from "#components/base.js";

/**
 * Example component.
 *
 * @cssproperty --asl-example-color - Text color. Default: `var(--asl-c-primary)`.
 */
export class AslExample extends AslBase {
    static styles = css`
        :host {
            display: inline;
            --_color: var(--asl-example-color, var(--asl-c-primary, oklch(75% 0.087 247)));
        }
        span {
            color: var(--_color);
        }
    `;

    render() {
        return html`<span>Hello world!</span>`;
    }
}
```

Key points:
- `@cssproperty` JSDoc tag gives IDE/tooling visibility
- `:host {}` is the public API surface — one look to see what's configurable
- The fallback chain ensures it looks fine with zero extra CSS

### 2. `assets/styles/_components.css` — NEW file (defaults + documentation)

This file serves double duty: it provides default values for component properties AND documents
the full API. Users who want the library's baseline look import `index.css`, which pulls this in.

```css
/**
 * Liter411y — Component CSS Custom Properties
 *
 * This file sets default values for all component-level custom properties.
 * Every property listed here can be overridden in your own CSS:
 *
 *   :root { --asl-example-color: hotpink; }               /* all instances */
 *   asl-example.special { --asl-example-color: hotpink; } /* specific instance */
 *
 * Import this file (or the full index.css) to get the library's baseline styles.
 */

/* ─── asl-example ─────────────────────────────────────────────────────────── */
/**
 * --asl-example-color
 *   Text color of the example component.
 *   Default: var(--asl-c-primary)
 */
:root {
    --asl-example-color: var(--asl-c-primary);
}
```

The structure per component:
- A section header comment with the element tag name
- One JSDoc block per property (name, description, default)
- A `:root {}` block with the actual default value

### 3. `package.json` — Add CSS to the exports map

The current `./*` glob only maps to JS. Extend exports so bundlers (Vite, webpack, Rollup)
can resolve CSS imports using the package name:

```json
"exports": {
    "./*": "./components/*.js",
    "./styles": "./assets/styles/index.css",
    "./styles/*": "./assets/styles/*"
}
```

Users then write:
```css
@import "liter411y/styles";           /* full baseline */
@import "liter411y/styles/_theme.css"; /* tokens only, if needed */
```

The wildcard also covers future per-component CSS files without touching `package.json` again.

### 4. `assets/styles/index.css` — Add `components` layer + import

```css
@layer base, theme, components;

@import "_reset.css" layer(base);
@import "_theme.css" layer(theme);
@import "_components.css" layer(components);
```

The `components` layer wins over `theme`, ensuring component defaults cleanly override raw
tokens when both are present. User CSS (unlayered) always wins over all layers.

---

## Side note: existing bugs in `_theme.css`

Lines 17–18 reference `var(--space-md)` (missing the `--asl-` prefix), so `--asl-space-sm`
and `--asl-space-lg` currently resolve to `initial`. These are pre-existing bugs unrelated to
this feature — should be fixed separately.

---

## How users consume this

```css
/* Option A: Full baseline — import the whole stylesheet */
@import "liter411y/assets/styles/index.css";

/* Option B: Override specific properties after importing */
@import "liter411y/assets/styles/index.css";
:root {
    --asl-example-color: hotpink;
}

/* Option C: No import at all — components still render with hardcoded fallbacks */
```

```js
// JS side is unchanged — users just import the component as before
import "liter411y/example/example.register.js";
```

---

## Verification

1. Register `asl-example` in an HTML file with no extra CSS → renders with `oklch(75% 0.087 247)` text color (hardcoded fallback)
2. Import `assets/styles/index.css` → color resolves through `var(--asl-c-primary)` via `_components.css`
3. Add `:root { --asl-example-color: hotpink }` after the import → all instances change
4. Add `asl-example.special { --asl-example-color: blue }` → only that instance changes
5. Inspect Shadow DOM in DevTools → `--_color` should resolve correctly at each stage

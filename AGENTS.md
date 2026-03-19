# AGENTS.md — Liter411y

Guidance for agentic coding tools working in this repository.

---

## Project Overview

**Liter411y** is a Frontend UI Design System Library (FUDSL) built with
[Lit](https://lit.dev/) Web Components. It is a private, early-stage project.

- Runtime: ES Modules only (`"type": "module"` in package.json)
- Only production dependency: `lit@3.3.2`
- All source lives under `components/`
- No build step — components are published as-is (plain JS)

---

## Environment Requirements

```
node >= 24.14.0   (see .nvmrc: lts/krypton)
npm  >= 11.9.0    (pinned via packageManager field)
```

`engine-strict=true` in `.npmrc` — npm will refuse to run with wrong versions.
`save-exact=true` in `.npmrc` — always pin exact dependency versions, no ranges.

---

## Commands

### Install

```bash
npm install
```

### Test

No test framework is configured yet (tracked in TODO.md). The current script
is a placeholder that exits with an error:

```bash
npm test   # ← will fail intentionally
```

**Do not add, run, or modify tests** until a testing strategy is decided.

### Lint / Format

No ESLint or Prettier config exists yet (planned in TODO.md). Do **not** add
linting or formatting tooling unless explicitly asked to.

### Build

No build step. Components are plain ES modules exported directly.

---

## Directory Structure

```
components/
  index.js              ← public re-export barrel
  base.js               ← AslBase (extends LitElement)
  {name}/
    {name}.js           ← component class
    {name}.register.js  ← customElements.define() call
assets/
  styles/               ← shared CSS/style assets
docs/                   ← architecture notes
```

---

## Component Architecture

All components follow this two-file pattern:

### `components/{name}/{name}.js` — the class

```js
import { html } from "lit";
import { AslBase } from "#components/base.js";

/**
 * TSDoc/JSDoc comment explaining WHY this component exists.
 */
export class AslName extends AslBase {
    render() {
        return html`<span>content</span>`;
    }
}
```

### `components/{name}/{name}.register.js` — the registration

```js
import { AslName } from "#components/{name}/{name}.js";

customElements.define("asl-name", AslName);
```

Registration files are **side-effect-only** and listed under `sideEffects` in
`package.json`. Never import the registration file from the class file.

### `components/index.js` — the barrel

Add a named re-export for every new component class:

```js
export { AslName } from "#components/{name}/{name}.js";
```

---

## Imports & Exports

- **Named exports only.** Never use `export default`.
- **Path aliases always** — use `#components/...` not relative `../...` paths.
  The alias is configured in both `package.json` (`imports`) and `jsconfig.json`.
- Import Lit primitives directly from `"lit"`: `html`, `css`, `LitElement`.
- Import component classes from their alias path, not from the barrel:

```js
// ✅ Correct
import { AslBase } from "#components/base.js";

// ❌ Wrong — avoid relative paths
import { AslBase } from "../base.js";

// ❌ Wrong — avoid default exports
export default class AslFoo extends AslBase { }
```

---

## Naming Conventions

| Thing | Convention | Example |
|---|---|---|
| Class names | PascalCase with `Asl` prefix | `AslButton`, `AslBase` |
| Custom element tags | kebab-case with `asl-` prefix | `asl-button`, `asl-card` |
| File names | kebab-case | `my-button.js` |
| Directory names | kebab-case | `components/my-button/` |
| Methods / properties | camelCase | `renderIcon()`, `isDisabled` |

---

## Code Style

Enforced by `.editorconfig` — ensure your editor respects it:

- **Indent**: 4 spaces (no tabs)
- **Line endings**: LF
- **Max line length**: 80 characters
- **Charset**: UTF-8
- **Trailing whitespace**: trimmed (except in Markdown)
- **Final newline**: always present

---

## Documentation (JSDoc / TSDoc)

Add a JSDoc block to every class and non-trivial method explaining **why** it
exists. The **how** should be clear from the code itself.

```js
/**
 * Base component that all/most components extend from.
 * Provides shared lifecycle hooks and theming primitives.
 */
export class AslBase extends LitElement { }
```

- Use `/** ... */` block style, not `//` line comments for class-level docs.
- Inline `//` comments are fine for complex implementation details.

---

## Key Rules for Agents

1. **No default exports.** Always use named exports.
2. **Always use `#components/...` path aliases** in imports, never `../`.
3. **Every new component** needs two files: `.js` (class) and `.register.js`.
4. **Add every new component** to `components/index.js` as a named re-export.
5. **Pin dependency versions** exactly (no `^` or `~`) — `.npmrc` enforces this.
6. **Do not add tooling** (ESLint, Prettier, test framework, git hooks) unless
   explicitly asked — these are tracked in TODO.md.
7. **No TypeScript.** This is a plain JavaScript project with `jsconfig.json`
   for editor support only.
8. **Extend `AslBase`**, not `LitElement` directly, for all new components.
9. **No build step.** Do not introduce bundlers or transpilers.
10. **JSDoc every class** with a description of its purpose.

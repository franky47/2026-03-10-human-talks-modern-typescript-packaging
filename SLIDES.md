---
theme: geist
title: Modern TypeScript Packaging
author: François Best
transition: none
highlighter: shiki
lineNumbers: false
colorSchema: light
fonts:
  sans: Geist
  mono: Geist Mono
  local: [Geist, Geist Mono]
  provider: none
---

# Modern TypeScript Packaging

## 2026 Edition

Human Talks Grenoble — 2026-03-10

<div class="author">
  <img src="/avatar.jpg" />
  <div>
    <strong>François Best</strong>
    <span>@francoisbest.com</span>
  </div>
</div>

---
layout: center
---

<div class="grid grid-cols-2 gap-8 w-full text-center">
<div>

# <span class="opacity-30">Application code</span>

</div>
<v-click>
<div>

# Library code

</div>
</v-click>
</div>

---

## The Landscape in 2026

<v-clicks>

- **ESM-only** is the new default
- TypeScript 6.0 Beta
- Fast build tools: **tsdown** (**oxc** + **Rolldown**), **tsgo**

</v-clicks>

---

## ESM-Only Publishing

```jsonc {all|3|4-9|all}
// package.json
{
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js",
    },
  },
}
```

`require(esm)` — CJS consumers can still `require()` your ESM package

---

## The Exports Map

```jsonc {all|3-7|8-11|all}
{
  "exports": {
    ".": { // → import {…} from 'foo'
      "types": "./dist/index.d.ts", // ← always first
      "browser": "./dist/index.browser.js", // ← other targets
      "default": "./dist/index.js", // ← always last
    },
    "./utils": { // → import {…} from 'foo/utils'
      "types": "./dist/utils.d.ts",
      "default": "./dist/utils.js",
    },
  },
}
```

**Condition order matters** — `types` first, `default` last

---

## tsconfig.json for Libraries

```jsonc {all|3-5|6-8|9-11|all}
{
  "compilerOptions": {
    "target": "ES2022",             // Runtime support, modern syntax
    "module": "NodeNext",           // Node's ESM/CJS resolution rules
    "moduleResolution": "NodeNext", // Resolve imports like Node does
    "declaration": true,            // Emit .d.ts files
    "declarationMap": true,         // Go-to-definition lands on .ts source
    "sourceMap": true,              // Debuggable output
    "erasableSyntaxOnly": true,     // Native TS execution (Node 23.6+)
    "verbatimModuleSyntax": true,   // Explicit import/export type
    "isolatedDeclarations": true,   // Fast parallel .d.ts emit
  },
}
```

Shortcut: `@total-typescript/tsconfig`

---
class: px-8
---

## Transpilation & Bundling

<br/>

<div className="font-medium">

```mermaid {theme: 'neutral'}
%%{init: {'themeVariables': {'fontSize': '24px', 'fontFamily': 'Geist, sans-serif' }}}%%
graph LR
  A["Who consumes? &nbsp;&nbsp;"] --> B["NPM / public &nbsp;"]
  B --> E["<strong>Transpile</strong> TS → JS&nbsp;<br/>+ generate .d.ts"]
  E --> F["Bundle? &nbsp;"]
  E --> G["No bundle? &nbsp;"]
  F --> H["<strong>tsdown</strong><br/>(single file, tree-shake)"]
  G --> I["tsc / <strong>tsgo</strong><br/>(preserve file structure)"]
  A ~~~ _s1[ ] ~~~ _s2[ ]
  A --> C["monorepo / private"]
  C --> D["<strong>No build</strong><br/>raw .ts source&nbsp;"]
  style _s1 fill:none,stroke:none
  style _s2 fill:none,stroke:none
```

</div>

---

## `erasableSyntaxOnly`

<br/>

### Blocked (needs runtime transform)

| Construct                   | Use instead         |
| --------------------------- | ------------------- |
| `enum`                      | `as const` object   |
| `namespace`                 | ES modules          |
| param properties in classes | explicit assignment |

<v-click>

### Why?

Node.js **strips types natively** — zero-transform TypeScript

</v-click>

---

## TypeScript 6.0 → 7.0

<div class="grid grid-cols-2 gap-8">
<div>

### 6.0 (March 2026)

<v-clicks>

- New defaults: `strict`, `isolatedDeclarations`
- Last JS-written compiler

</v-clicks>

</div>
<div>

### 7.0 — `tsgo`

<v-clicks>

- Full rewrite in **Go**
- **10x faster** type-checking
- Native CLI: `tsgo --build`

</v-clicks>

</div>
</div>

---

## Build Tools — 2026

Criteria: transpilation + .d.ts emit, bundling, tree-shaking, type-checking

<br/>

| Tool           | Engine         | Bundle     | .d.ts           | Tree-shake   |
| -------------- | -------------- | ---------- | --------------- | ------------ |
| **tsdown**     | Rolldown + oxc | yes        | ⚡ isolated decl | yes          |
| tsup           | esbuild        | yes        | via tsc         | yes          |
| tsc / **tsgo** | JS / Golang    | no         | yes             | no           |

<v-click>

</v-click>

---
layout: two-cols
---

## Monorepo

```
packages/
  shared/
    src/index.ts   // raw .ts, no build  →
    package.json
  app/
    src/main.ts    // imports @repo/shared
    package.json
```

```jsonc
// packages/app/package.json
{
  "name": "@repo/app",
  "dependencies": {
    "@repo/shared": "workspace:*",
  },
}
```

::right::

## Internal Packages

<div>

```jsonc
// packages/shared/package.json
{
  "name": "@repo/shared",
  "exports": {
    ".": "./src/index.ts",
  },
}
```

<v-clicks>

- Declare raw `.ts` files in exports map
- Use local `workspace:*` resolution
- Only build **for outside the monorepo**

</v-clicks>

</div>

---

## Before You Publish



```bash
# Lint your package.json
pnpx publint

# Check TypeScript resolution for all consumers
pnpx @arethetypeswrong/cli --pack .
```

🌐 Online version:

- https://publint.dev
- https://arethetypeswrong.github.io/


---

## Recap — 2026 Defaults

|          | Public library                    | monorepo pkg |
| -------- | --------------------------------- | ------------ |
| Format   | ESM-only, emit `.js` + `.d.ts`    | raw `.ts`    |
| Build    | **tsdown**                        | none         |
| tsconfig | `NodeNext` + `declaration`        | bundler mode |
| Types    | `isolatedDeclarations`            | source       |
| Validate | `publint` + `attw`                | —            |
| Registry | **npm** (or JSR)                  | workspace    |


---
layout: center
---

# Thank you!

<br/>

Slides, coding agent skill & resources:

<img src="/qr.png" alt="QR Code" class="mx-auto my-8" />

<div class="mt-2 text-sm opacity-50">

https://github.com/franky47/2026-03-10-human-talks-modern-typescript-packaging

</div>

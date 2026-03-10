# Modern TypeScript Library Packaging — Research Notes (March 2026)

> Compiled research for a 10min Human Talks presentation.
> Covers best practices for public npm libraries and internal monorepo packages.

---

## Table of Contents

1. [TL;DR — The 2026 Defaults](#1-tldr--the-2026-defaults)
2. [ESM vs CJS](#2-esm-vs-cjs)
3. [Bundling: Yes vs No](#3-bundling-yes-vs-no)
4. [Transpilation: Ship JS or Raw TS?](#4-transpilation-ship-js-or-raw-ts)
5. [Module Resolution & Import Extensions](#5-module-resolution--import-extensions)
6. [File Extensions (.ts, .mts, .cts)](#6-file-extensions-ts-mts-cts)
7. [package.json Fields](#7-packagejson-fields)
8. [The `exports` Map — Deep Dive](#8-the-exports-map--deep-dive)
9. [tsconfig.json — Best Practices](#9-tsconfigjson--best-practices)
10. [TypeScript 6.0 Changes](#10-typescript-60-changes)
11. [Modern Build Tooling](#11-modern-build-tooling)
12. [Registries: npm, JSR, vlt](#12-registries-npm-jsr-vlt)
13. [Monorepo Setup](#13-monorepo-setup)
14. [Source Maps & Declaration Maps](#14-source-maps--declaration-maps)
15. [Validation Tools](#15-validation-tools)

---

## 1. TL;DR — The 2026 Defaults

### For a public npm library (starting from scratch)

```
ESM-only · type: "module" · exports map with "types" first
tsc or tsdown for build · .d.ts + .d.ts.map for types
moduleResolution: "nodenext" · target: "es2022"
Validate with publint + arethetypeswrong
```

### For an internal monorepo package

```
No build step · exports point to raw .ts source
private: true · workspace:* protocol (pnpm)
App's bundler handles transpilation
```

---

## 2. ESM vs CJS

### The landscape in 2026

- **Node.js 18** (last LTS without good ESM support) reached **EOL April 2025**
- **Node.js 22+** supports `require()` of ESM modules (no top-level `await`)
- **Node.js v25.4.0** made `require(esm)` **stable** (no longer experimental)
- **Bun** and **Deno** are ESM-native
- All major bundlers handle ESM natively

### Recommendation

**Ship ESM-only.** Set `"type": "module"` in package.json.

Dual CJS+ESM publishing is now unnecessary overhead for most libraries:

- `require(esm)` eliminates the primary reason dual publishing existed
- Dual publishing risks the **"dual package hazard"** (same package loaded twice — ESM + CJS instances with separate state)
- Requires separate `.d.ts` and `.d.cts` type declarations to avoid FalseESM/FalseCJS problems
- Sindre Sorhus and many prolific authors have been ESM-only since 2021

**Only dual-publish if** you have known CJS consumers on Node.js < 22.

---

## 3. Bundling: Yes vs No

| Factor             | Bundle (tsup, tsdown)            | No-Bundle (tsc, file-to-file)                  |
| ------------------ | -------------------------------- | ---------------------------------------------- |
| Tree-shaking       | Consumer relies on their bundler | Natural — consumers import only what they need |
| Internal structure | Hidden from consumers            | May leak internal paths                        |
| Package size       | Smaller (single file)            | Larger (many files)                            |
| Debugging          | Harder without source maps       | Easier — 1:1 file mapping                      |
| Circular deps      | Bundler resolves them            | Can cause runtime issues                       |

### Recommendation

- **Single entry-point library** → bundle with tsdown or tsup
- **Multi-export library** (many subpaths) → file-to-file with tsc or unbuild's mkdist
- **Internal monorepo package** → no build at all (raw .ts)
- **Never minify library code** — let the application bundler handle that

---

## 4. Transpilation: Ship JS or Raw TS?

### Can you ship raw .ts to npm?

**No.** Node.js **explicitly blocks `.ts` files from `node_modules/`**. This is a deliberate policy, not a bug.

- Node.js v24+ has **stable built-in type stripping** (enabled by default since v22.18.0)
- But it only works for your **own** `.ts` files, not dependencies in `node_modules/`
- Only supports "erasable" syntax (no `enum`, no runtime `namespace`, no parameter properties)
- Bun is the only runtime that runs `.ts` from `node_modules/`, but you can't target Bun-only
- The TC39 Type Annotations proposal is still **Stage 1** — years away

### Recommendation

- **Public npm library** → transpile to JS + ship `.d.ts` declarations
- **Internal monorepo package** → ship raw `.ts` (see [Monorepo Setup](#13-monorepo-setup))
- **JSR registry** → publish raw `.ts` directly (JSR handles transpilation)

---

## 5. Module Resolution & Import Extensions

### How to write imports

The answer depends on whether you're **compiling to JS** (tsc/bundler) or **running .ts directly** (Node.js type stripping):

```typescript
// Scenario A: Compiling with tsc (moduleResolution: "nodenext")
// Use .js extension — TypeScript resolves .js → .ts at compile time
import { foo } from './utils/helper.js' // ✅ Works after compilation

// Scenario B: Running .ts directly with Node.js v24+ type stripping
// Use .ts extension — the actual file IS .ts
import { foo } from './utils/helper.ts' // ✅ Works in Node.js v24+

// These don't work:
import { bar } from './utils/helper' // ❌ Fails in Node.js ESM (no extension)
import { baz } from './utils/helper.js' // ❌ Fails if running .ts directly (no .js file exists)
```

**Key insight**: The `.js` extension convention exists because `tsc` emits `.js` files and keeps the import specifier as-is. When running `.ts` natively (Node.js type stripping, Bun, Deno), you use `.ts` extensions because that's the actual file.

### tsconfig flags for `.ts` extensions in imports

Two TypeScript options control whether you can write `.ts` extensions in imports:

**`allowImportingTsExtensions`** (TS 5.0) — Allows `import { foo } from './foo.ts'` in source code. **Requires `noEmit` or `emitDeclarationOnly`** because tsc wouldn't know how to rewrite the extension in emitted JS. Use this when a bundler or runtime handles execution (Node.js type stripping, Bun, Deno).

**`rewriteRelativeImportExtensions`** (TS 5.7) — More powerful: allows `.ts` imports AND rewrites them to `.js` in emitted output. **Does NOT require `noEmit`** — you can compile with tsc and get correct `.js` imports in output. When enabled, `allowImportingTsExtensions` defaults to `true` automatically.

```jsonc
// Option A: No emit (bundler/runtime handles it)
{
  "compilerOptions": {
    "allowImportingTsExtensions": true,
    "noEmit": true  // or emitDeclarationOnly
  }
}

// Option B: tsc emits JS with rewritten extensions (TS 5.7+)
{
  "compilerOptions": {
    "rewriteRelativeImportExtensions": true
    // allowImportingTsExtensions is implied
    // .ts imports are rewritten to .js in output
  }
}
```

This is significant: with `rewriteRelativeImportExtensions`, you can write natural `.ts` imports that work both when running directly (Node.js v24+, Bun, Deno) AND when compiled to JS.

### Cross-runtime compatibility

| Import style     | Node.js ESM (compiled JS) | Node.js ESM (running .ts directly) | Bun | Deno |
| ---------------- | ------------------------- | ---------------------------------- | --- | ---- |
| `./foo` (no ext) | ❌                        | ❌                                 | ✅  | ❌   |
| `./foo.js`       | ✅                        | ❌ (no .js file)                   | ✅  | ✅   |
| `./foo.ts`       | ❌ (no .ts file in dist)  | ✅ (v24+)                          | ✅  | ✅   |

### TypeScript `moduleResolution` settings

| Setting    | `exports` support | Requires extensions | Best for                                 |
| ---------- | ----------------- | ------------------- | ---------------------------------------- |
| `node10`   | ❌                | ❌                  | **Never use** (deprecated in TS 6.0)     |
| `node16`   | ✅                | ✅ (ESM)            | Libraries targeting old Node.js versions |
| `nodenext` | ✅                | ✅ (ESM)            | **Libraries** (recommended)              |
| `bundler`  | ✅                | ❌                  | **Apps** processed by a bundler          |

**Rule of thumb**: `nodenext` for libraries, `bundler` for apps.

---

## 6. File Extensions (.ts, .mts, .cts)

### Source extensions

| Extension | Module format                              | When to use                |
| --------- | ------------------------------------------ | -------------------------- |
| `.ts`     | Determined by `type` field in package.json | Default for most files     |
| `.mts`    | Always ESM                                 | ESM file in a CJS package  |
| `.cts`    | Always CJS                                 | CJS file in an ESM package |

### Output mapping

| Source              | JS output | Declaration output |
| ------------------- | --------- | ------------------ |
| `.ts` (ESM context) | `.js`     | `.d.ts`            |
| `.mts`              | `.mjs`    | `.d.mts`           |
| `.cts`              | `.cjs`    | `.d.cts`           |

### Interaction with `type: "module"`

| `type` field           | `.js` / `.ts` | `.mjs` / `.mts` | `.cjs` / `.cts` |
| ---------------------- | ------------- | --------------- | --------------- |
| `"module"`             | ESM           | ESM             | CJS             |
| `"commonjs"` or absent | CJS           | ESM             | CJS             |

`.mjs`/`.mts` and `.cjs`/`.cts` **always override** the `type` field.

---

## 7. package.json Fields

### Essential fields (2026)

| Field            | Status        | Include?                     |
| ---------------- | ------------- | ---------------------------- |
| `exports`        | **Essential** | Always                       |
| `type: "module"` | **Essential** | Always                       |
| `files`          | **Essential** | Always (allowlist approach)  |
| `imports`        | Useful        | If you have internal aliases |

### Legacy/outdated fields

| Field               | Status              | Include?                     |
| ------------------- | ------------------- | ---------------------------- |
| `main`              | Legacy fallback     | Optional, for old tools      |
| `types` (top-level) | Legacy fallback     | Optional, for `node10` users |
| `module`            | Non-standard        | **No**                       |
| `typesVersions`     | Obsolete workaround | **No**                       |

### Minimal package.json for a public library

```json
{
  "name": "my-library",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    }
  },
  "files": ["dist"],
  "sideEffects": false
}
```

---

## 8. The `exports` Map — Deep Dive

### Why `exports` over `main`

- **Multiple entry points** (subpath exports)
- **Encapsulation** (only listed paths accessible — unlisted paths throw `ERR_PACKAGE_PATH_NOT_EXPORTED`)
- **Conditional resolution** (ESM/CJS/types/browser/node)
- **Wildcard patterns**

### Condition ordering (CRITICAL)

```json
{
  "exports": {
    ".": {
      "types": "...", // 1. ALWAYS FIRST (TypeScript)
      "browser": "...", // 2. Environment-specific
      "node": "...", // 3. Environment-specific
      "import": "...", // 4. Module format
      "require": "...", // 5. Module format
      "default": "..." // 6. ALWAYS LAST (fallback)
    }
  }
}
```

**`"types"` MUST be first** — this is the #1 most common mistake in TypeScript library packaging.

### Common patterns

**ESM-only (simplest, recommended)**:

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    }
  }
}
```

**Dual CJS/ESM (if needed)**:

```json
{
  "exports": {
    ".": {
      "import": {
        "types": "./dist/index.d.mts",
        "default": "./dist/index.mjs"
      },
      "require": {
        "types": "./dist/index.d.cts",
        "default": "./dist/index.cjs"
      }
    }
  }
}
```

**Multiple subpaths**:

```json
{
  "exports": {
    ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" },
    "./react": { "types": "./dist/react.d.ts", "default": "./dist/react.js" },
    "./utils": { "types": "./dist/utils.d.ts", "default": "./dist/utils.js" },
    "./package.json": "./package.json"
  }
}
```

**Wildcard exports**:

```json
{
  "exports": {
    ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" },
    "./*": { "types": "./dist/*.d.ts", "default": "./dist/*.js" }
  }
}
```

**Internal monorepo package (no build)**:

```json
{
  "name": "@myorg/shared",
  "private": true,
  "type": "module",
  "exports": {
    ".": "./src/index.ts",
    "./*": "./src/*.ts"
  }
}
```

### The `imports` field (private internal mappings)

```json
{
  "imports": {
    "#utils": "./src/utils.js",
    "#db": {
      "node": "better-sqlite3",
      "default": "./src/db-polyfill.js"
    }
  }
}
```

Entries must start with `#`. Only accessible within the package.

### Common pitfalls

1. **`"types"` not first** → TypeScript won't find your types
2. **Mismatched `.d.ts` extensions** → `.mjs` output needs `.d.mts` types, `.cjs` needs `.d.cts`
3. **Missing `"default"` fallback** → reduced compatibility
4. **Forgetting encapsulation** → once you add `exports`, unlisted deep imports break
5. **Not exporting `./package.json`** → some tools need it

---

## 9. tsconfig.json — Best Practices

### Public library (tsc, Node.js)

```jsonc
{
  "compilerOptions": {
    // Strictness
    "strict": true, // Default in TS 6.0
    "noUncheckedIndexedAccess": true, // NOT included in strict
    "noImplicitOverride": true, // NOT included in strict

    // Module system
    "module": "NodeNext",
    "moduleDetection": "force",
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,

    // Output
    "target": "es2022",
    "lib": ["es2022"],
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,

    // Performance
    "skipLibCheck": true,
  },
  "include": ["src"],
}
```

### Internal monorepo package

Same as above, plus:

```jsonc
{
  "compilerOptions": {
    "composite": true, // Enables project references + incremental builds
  },
}
```

### Application (not a library)

Same base, but remove `declaration`, `declarationMap`, `composite`.

### Bundler-based library (Vite/esbuild handles emit)

```jsonc
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "noEmit": true, // tsc only type-checks; bundler emits JS
    // For declarations: replace noEmit with emitDeclarationOnly + declaration
  },
}
```

### Key options explained

| Option                         | Why                                                                    |
| ------------------------------ | ---------------------------------------------------------------------- |
| `module: "NodeNext"`           | Correct Node.js ESM/CJS semantics, supports `exports` field            |
| `verbatimModuleSyntax`         | Forces `import type` for type-only imports; replaces legacy flags      |
| `isolatedModules`              | Ensures each file can be independently transpiled (esbuild/swc compat) |
| `moduleDetection: "force"`     | Treats every file as a module                                          |
| `declaration + declarationMap` | Generates `.d.ts` + enables "Go to Definition" into source             |
| `skipLibCheck`                 | Skip type-checking `node_modules` `.d.ts` files (performance)          |

### Matt Pocock's `@total-typescript/tsconfig`

Pre-built configs organized by transpiler × environment × project type:

```json
{ "extends": "@total-typescript/tsconfig/tsc/no-dom/library" }
```

12 variants available: `tsc`/`bundler` × `dom`/`no-dom` × `app`/`library`/`library-monorepo`.

---

## 10. TypeScript 6.0 Changes

**Status**: RC released March 6, 2026. Final release: March 17, 2026.
**Note**: TS 6.0 is the **last major version written in TypeScript**. TS 7.0 will be the native Go port (tsgo).

### New defaults (no config needed)

| Option                         | Old Default                 | TS 6.0 Default                          |
| ------------------------------ | --------------------------- | --------------------------------------- |
| `strict`                       | `false`                     | **`true`**                              |
| `target`                       | `es3`/`es5`                 | **Latest ES**                           |
| `noUncheckedSideEffectImports` | `false`                     | **`true`**                              |
| `types` array                  | Auto-include all `@types/*` | **Empty** (explicit inclusion required) |

### Deprecated/removed

- **ES5 target** → minimum is ES2015
- **`moduleResolution: node10`** and **`classic`** → deprecated
- **`module: amd/system/umd`** → deprecated
- **`baseUrl`** for path resolution → deprecated (use `paths` or `#/` subpath imports)
- **Import assertions (`assert {}`)** → removed (use `with {}` import attributes)

### Notable new features from TS 5.5–6.0

| Feature                                              | Version | Impact on libraries                                                                   |
| ---------------------------------------------------- | ------- | ------------------------------------------------------------------------------------- |
| `isolatedDeclarations`                               | TS 5.5  | Fast `.d.ts` generation without full type-checking; enables parallel declaration emit |
| `erasableSyntaxOnly`                                 | TS 5.8  | Only allows erasable syntax (see below); aligns with Node.js type stripping           |
| `#/` subpath imports                                 | TS 6.0  | Package-internal imports via `#/` prefix                                              |
| `module: "commonjs"` + `moduleResolution: "bundler"` | TS 6.0  | Previously disallowed combo, now permitted                                            |

### `erasableSyntaxOnly` (TS 5.8) — recommended for new projects

Blocks any TypeScript syntax that requires **code generation** (not just type erasure). Created in collaboration with the Node.js core team to ensure compatibility with Node's native type stripping.

**Blocked syntax:**

- `enum` (all forms, including `const enum`)
- `namespace` with runtime code
- Parameter properties (`constructor(private x: number)`)
- `import =` / `export =`
- Angle-bracket type assertions (`<Type>value` — use `value as Type` instead)

**Allowed** (erasable): type annotations, interfaces, `type` aliases, `import type`, generics, `declare` blocks, `abstract`, overloads — anything that can be deleted to produce valid JS.

**Who recommends it:** Node.js official docs, `@tsconfig/bases`, Matt Pocock (Total TypeScript), Rob Palmer (TC39 delegate). Angular is already replacing enums.

**Recommendation:** Enable for new projects. Library authors should strongly consider it — avoids enum API compatibility issues and future-proofs for the TC39 Type Annotations proposal. For existing projects, migrate gradually using `eslint-plugin-erasable-syntax-only`.

See: https://www.totaltypescript.com/erasable-syntax-only

### ⚠️ Action required for TS 6.0

- **Add `"types": ["node"]`** to tsconfig (no longer auto-included)
- **Migrate from `node10`/`classic` resolution** to `nodenext` or `bundler`
- **Replace `assert {}` with `with {}`** in import attributes

---

## 11. Modern Build Tooling

### Comparison matrix

| Tool         | Engine           | Speed     | Config      | .d.ts         | Tree-shaking | Best for                            |
| ------------ | ---------------- | --------- | ----------- | ------------- | ------------ | ----------------------------------- |
| **tsdown**   | Rolldown (Rust)  | Very fast | Simple      | Yes (via tsc) | Excellent    | New libraries (forward-looking)     |
| **tsup**     | esbuild (Go)     | Very fast | Simple      | Yes (via tsc) | Good         | Proven choice, large community      |
| **unbuild**  | Rollup + mkdist  | Moderate  | Zero-config | Yes           | Excellent    | Multi-export libs, stub mode DX     |
| **tsc/tsgo** | Native TS        | Slow/Fast | Minimal     | Native        | N/A          | No-bundle approach, max correctness |
| **pkgroll**  | Rollup + esbuild | Moderate  | Zero-config | Yes           | Excellent    | package.json-as-config approach     |
| **esbuild**  | Go               | Fastest   | Manual      | No (need tsc) | Good         | Custom pipelines, embedded engine   |
| **swc**      | Rust             | Very fast | Manual      | No (need tsc) | N/A          | Embedded engine (Next.js, etc.)     |
| **oxc**      | Rust             | Fastest   | Manual      | Yes (isolated decl) | N/A     | Embedded engine (Rolldown, Vite 8)  |

### The big stories

**tsdown** (v0.21.1) — Spiritual successor to tsup, built on **Rolldown** (Rust-based Rollup replacement by VoidZero/Evan You, currently **v1 RC**). Better tree-shaking, scope hoisting, code splitting than esbuild. Rollup plugin compatible. Still pre-1.0 but rapidly maturing. Forward-looking default for new projects.

**oxc** — The Rust compiler infrastructure powering Rolldown, Vite 8, and tsdown. Its **oxc-transform** package (v0.112.0, ~268k weekly npm downloads) handles TypeScript stripping, JSX transformation, and syntax lowering (to ES2015). Uniquely offers **isolated declarations** `.d.ts` emit — 40x faster than tsc. 3-5x faster than SWC with ~2 MB package size (vs SWC's 37 MB). Library authors interact with oxc indirectly through tsdown/Vite rather than directly. Pre-1.0 as standalone npm package, but production-proven through its integration in Rolldown/Vite 8. Limitations: no TC39 Stage 3 decorators, no ES5 target.

**tsgo** (beta) — Native **Go port** of the TypeScript compiler by Microsoft's TS team (led by Anders Hejlsberg). Announced March 2025. **~10x performance improvement**. TypeScript 6.0 is the **last major version written in TypeScript** — TypeScript 7.0 will be the Go port. Biggest impact: `.d.ts` generation (the main bottleneck) becomes near-instant.

**Isolated declarations** (TS 5.5) — Enables non-tsc tools to generate `.d.ts` without full type-checking. Requires explicit return type annotations on exported functions. **Significantly speeds up tsdown/tsup** for `.d.ts` bundling since the bundler can generate declarations without invoking the full type-checker. Combine with `isolatedDeclarations: true` in tsconfig. oxc's implementation (`oxc_isolated_declarations`) is 40x faster than tsc for typical files — Vue-macros reported `.d.ts` generation dropping from 76s to 16s. tsdown uses oxc's isolated declarations when the flag is enabled, falling back to tsc otherwise.

### Decision framework

- **New library, simple** → `tsdown` (or `tsup` if tsdown isn't stable enough)
- **Multi-export library** → `unbuild` (file-to-file via mkdist, zero-config, stub mode)
- **No bundling needed** → `tsc` alone (consumers will bundle)
- **Already using Vite** → Vite library mode
- **Zero config** → `pkgroll` or `unbuild`

### Tools that are NOT build tools (but power them)

- **Biome** — linting + formatting only (no transpilation/bundling)
- **oxc** — compiler infrastructure (parser, transformer, resolver, minifier) — powers Rolldown/Vite 8/tsdown; standalone npm package `oxc-transform` available but pre-1.0
- **SWC** — embedded engine (powers Next.js/Turbopack), rarely used standalone for library builds
- **esbuild** — increasingly embedded (powers tsup, legacy Vite dev), being replaced by oxc in Vite 8

---

## 12. Registries: npm, JSR, vlt

### npm (default)

- Ship compiled JS + `.d.ts`
- Use `"files"` allowlist (not `.npmignore`)
- Use `npm pack --dry-run` to preview published files
- Use `--provenance` flag for supply chain security (from CI)

### JSR (jsr.io)

Created by the Deno team. **Publish raw `.ts` directly** — no build step needed.

| Feature                  | JSR                           | npm                   |
| ------------------------ | ----------------------------- | --------------------- |
| TypeScript               | First-class (raw `.ts`)       | Compiled JS + `.d.ts` |
| Module format            | ESM only                      | ESM and/or CJS        |
| Auto-generated docs      | Yes (from JSDoc/TSDoc)        | No                    |
| Type checking on publish | Yes                           | No                    |
| Quality scoring          | Yes                           | No                    |
| npm compatibility        | Yes (via compatibility layer) | Native                |
| Scoping                  | Required (`@scope/name`)      | Optional              |

```json
// jsr.json
{
  "name": "@myorg/my-package",
  "version": "1.0.0",
  "exports": "./mod.ts"
}
```

```bash
npx jsr publish   # or: deno publish
```

Consumers install via: `npx jsr add @myorg/my-package`

### vlt (vlt.sh)

New package manager + registry by former npm engineers. Not covered in depth — still early stage.

---

## 13. Monorepo Setup

### The "Internal Packages" pattern (recommended)

Internal packages (not published to npm) skip the build step entirely. The app's bundler handles transpilation.

```
packages/utils/
  src/index.ts         ← raw TypeScript, no build step
  package.json         ← exports point to src/*.ts
```

```json
// packages/utils/package.json
{
  "name": "@myorg/utils",
  "private": true,
  "type": "module",
  "exports": {
    ".": "./src/index.ts",
    "./math": "./src/math.ts"
  }
}
```

```json
// apps/web/package.json
{
  "dependencies": {
    "@myorg/utils": "workspace:*"
  }
}
```

**Why this works**: Next.js, Vite, Turbopack, esbuild all handle `.ts` imports natively. No separate build step, changes reflected instantly.

### Workspace protocols

| Protocol      | Syntax                            | Behavior                                                                     |
| ------------- | --------------------------------- | ---------------------------------------------------------------------------- |
| `workspace:*` | `"@myorg/utils": "workspace:*"`   | Local symlink; on publish, replaced with actual version. **pnpm/yarn only.** |
| `workspace:^` | `"@myorg/utils": "workspace:^"`   | On publish → `^1.0.0`                                                        |
| `file:`       | `"@myorg/utils": "file:../utils"` | Creates a copy. All package managers.                                        |
| `link:`       | `"@myorg/utils": "link:../utils"` | Creates a symlink.                                                           |

**Recommendation**: `workspace:*` with pnpm.

### TypeScript project references (for large monorepos)

```jsonc
// packages/utils/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true
  }
}

// apps/web/tsconfig.json
{
  "references": [{ "path": "../../packages/utils" }]
}
```

Helps the TS language server with performance in large monorepos. Not strictly required if editor performance is acceptable.

### Recommended monorepo stack

- **pnpm** for package management (disk-efficient, strict, `workspace:` protocol)
- **Turborepo** for build orchestration + caching (if you have buildable packages)
- **Internal packages pattern** for everything that isn't published

### Example structure

```
my-monorepo/
  pnpm-workspace.yaml
  turbo.json (optional)
  tsconfig.base.json

  packages/
    utils/              ← internal (no build)
      src/index.ts
      package.json      ← exports: ./src/index.ts
    shared-lib/         ← published (has build)
      src/index.ts
      dist/
      package.json      ← exports: ./dist/index.js

  apps/
    web/
      package.json      ← depends on @myorg/utils, @myorg/shared-lib
```

---

## 14. Source Maps & Declaration Maps

### Three levels

| Type                    | File              | Purpose                                                               |
| ----------------------- | ----------------- | --------------------------------------------------------------------- |
| **Runtime source maps** | `.js.map`         | Map transpiled JS → original TS for stack traces & debugging          |
| **Declaration maps**    | `.d.ts.map`       | Map `.d.ts` → original `.ts` for "Go to Definition" in editors        |
| **Inline source maps**  | embedded in `.js` | No separate file, but bloats size. **Not recommended for libraries.** |

### Recommendation

- **Always ship `.d.ts.map`** (declaration maps) — huge DX improvement for consumers (cmd+click → actual `.ts` source)
- **Include `.ts` source** in the package if you ship declaration maps
- **Runtime `.js.map`** — optional if output is readable unminified ESM. Include if output is significantly transformed.
- **Never minify library code** — leave that to the app bundler

Node.js type stripping replaces type annotations with **whitespace** (not removal), so line numbers are preserved without source maps when running `.ts` directly.

---

## 15. Validation Tools

### Before publishing, always run:

**publint** (https://publint.dev) — Lints your package.json for common issues:

```bash
npx publint
```

**Are The Types Wrong?** (https://arethetypeswrong.github.io) — Validates TypeScript type resolution across all `moduleResolution` modes:

```bash
npx @arethetypeswrong/cli my-package
```

---

## Quick Reference: Putting It All Together

### Public npm library — minimal setup

```bash
# Build
tsdown src/index.ts --format esm --dts
```

```json
// package.json
{
  "name": "my-lib",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    }
  },
  "files": ["dist"]
}
```

```json
// tsconfig.json
{
  "extends": "@total-typescript/tsconfig/tsc/no-dom/library",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

### Internal monorepo package — zero setup

```json
// package.json
{
  "name": "@myorg/utils",
  "private": true,
  "type": "module",
  "exports": {
    ".": "./src/index.ts"
  }
}
```

No build step. No tsconfig (inherits from root). The app's bundler handles everything.

---

## Sources

### Node.js

- [Node.js Packages (exports, type, conditions)](https://nodejs.org/api/packages.html)
- [Node.js ESM docs](https://nodejs.org/api/esm.html)
- [Node.js CJS docs (require(esm))](https://nodejs.org/api/modules.html)
- [Node.js TypeScript support (type stripping)](https://nodejs.org/docs/latest-v24.x/api/typescript.html)
- [Dual Package Hazard](https://nodejs.org/api/packages.html#dual-package-hazard)

### TypeScript

- [TypeScript 6.0 Iteration Plan](https://github.com/microsoft/TypeScript/issues/63085)
- [tsconfig/bases (node22, node-lts)](https://github.com/tsconfig/bases)
- [@total-typescript/tsconfig](https://github.com/total-typescript/tsconfig)
- [Total TypeScript TSConfig Cheat Sheet](https://www.totaltypescript.com/tsconfig-cheat-sheet)
- [Total TypeScript: erasableSyntaxOnly](https://www.totaltypescript.com/erasable-syntax-only)
- [Total TypeScript: TypeScript Go Rewrite](https://www.totaltypescript.com/typescript-announces-go-rewrite)
- [Total TypeScript: How To Create An NPM Package](https://www.totaltypescript.com/how-to-create-an-npm-package)
- [Total TypeScript: Relative Import Extensions](https://www.totaltypescript.com/relative-import-paths-need-explicit-file-extensions-in-ecmascript-imports)

### Build Tools & Compiler Infrastructure

- [tsdown](https://tsdown.dev/) (based on oxc & rolldown)
- [tsup](https://tsup.egoist.dev) (based on esbuild & rollup)
- [unbuild](https://github.com/unjs/unbuild)
- [pkgroll](https://github.com/privatenumber/pkgroll)
- [Rolldown](https://rolldown.rs/)
- [esbuild](https://esbuild.github.io/)
- [SWC](https://swc.rs/)
- [oxc](https://oxc.rs/) (Rust compiler infrastructure — parser, transformer, isolated declarations)
- [oxc-transform on npm](https://www.npmjs.com/package/oxc-transform)
- [VoidZero](https://voidzero.dev/) (company behind Vite, Rolldown, oxc)

### Registries & Ecosystem

- [JSR](https://jsr.io/docs/introduction)
- [publint](https://publint.dev)
- [Are The Types Wrong?](https://arethetypeswrong.github.io)
- [Sindre Sorhus ESM-only discussion](https://github.com/sindresorhus/meta/discussions/15)

### Runtimes

- [Bun modules](https://bun.sh/docs/runtime/modules)
- [Bun TypeScript](https://bun.sh/docs/runtime/typescript)
- [Deno TypeScript](https://docs.deno.com/runtime/fundamentals/typescript/)

---

## Items Verified / Still to Verify

- [x] **tsgo**: Still in beta. TS 6.0 is last TS-written major; TS 7.0 = Go port.
- [x] **tsdown**: v0.21.1 (pre-1.0). Actively developed.
- [x] **Rolldown**: v1 RC.
- [x] **Isolated declarations**: Yes, significantly speeds up tsdown `.d.ts` bundling. oxc's implementation is 40x faster than tsc.
- [x] **`@total-typescript/tsconfig`**: v1.0.4 not yet updated for TS 6.0, but defaults already align. No TS 6.0 blog post yet.
- [x] **`erasableSyntaxOnly`**: Yes, mainstream recommendation. Recommended by Node.js docs, tsconfig/bases, Matt Pocock, Rob Palmer. See expanded section in doc.
- [x] **Node.js `module-sync` condition**: Researched (see `research/module-sync-condition.md`), too advanced for this talk

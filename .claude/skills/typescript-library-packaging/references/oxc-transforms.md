# oxc Transformation Tools & the TypeScript Packaging Ecosystem

## Executive Summary

[oxc](https://oxc.rs/) (the "Oxidation Compiler") is a collection of high-performance JavaScript/TypeScript tools written in Rust, created by Boshen (VP Engineering at VoidZero). While oxc encompasses a parser, linter (Oxlint), formatter (Oxfmt), resolver, minifier, and transformer, this document focuses specifically on its **transformation capabilities** and how they fit into the modern TypeScript library packaging story.

The key transformation tools are: **oxc-transform** (TypeScript/JSX compilation + isolated declarations emit), **oxc_isolated_declarations** (standalone .d.ts generation without tsc), and **oxc-resolver** (Node.js-compatible module resolution). These tools power Rolldown (Vite's Rust-based bundler) and, by extension, tsdown (the library bundler built on Rolldown). As of March 2026, the transformer is actively used in production through Rolldown/Vite 8, though the standalone npm package (`oxc-transform` v0.112.0) has not yet reached a 1.0 release.

oxc is 3-5x faster than SWC, 20-50x faster than Babel, and its isolated declarations emit is 40x faster than tsc. The package size is approximately 2 MB vs SWC's 37 MB.

## oxc-transform: The Core Transformer

### What it does

The [oxc-transform npm package](https://www.npmjs.com/package/oxc-transform) provides TypeScript-to-JavaScript compilation, JSX transformation, syntax lowering, and isolated declarations (.d.ts) emit -- all in a single Rust-native package with Node.js bindings.

**Current version**: 0.112.0 (as of March 2026)
**Weekly npm downloads**: ~268,000
**Package size**: ~2 MB (vs SWC's 37 MB, Babel's 170+ MB ecosystem)
**Dependents**: 66 packages in the npm registry

### API

```typescript
import {
  transform,
  transformSync,
  isolatedDeclaration,
  isolatedDeclarationSync,
} from 'oxc-transform'

// TypeScript -> JavaScript transformation
const result = await transform('lib.ts', sourceCode, {
  typescript: {
    onlyRemoveTypeImports: true, // for verbatimModuleSyntax
    rewriteImportExtensions: 'rewrite', // .ts -> .js in imports
    allowNamespaces: false,
    removeClassFieldsWithoutInitializer: false,
  },
  jsx: {
    runtime: 'automatic', // or "classic"
    development: false,
    refresh: false, // React Fast Refresh
    importSource: 'react',
  },
  target: ['es2022'], // syntax lowering target
  sourcemap: true,
  declaration: {
    // combined JS + .d.ts emit
    stripInternal: true,
  },
})
// result: { code, map, declaration, declarationMap, errors }

// Standalone .d.ts generation
const dts = isolatedDeclarationSync('lib.ts', sourceCode, {
  sourcemap: false,
  stripInternal: false,
})
// dts: { code, map, errors }
```

### TypeScript Transformation

oxc's TypeScript transform operates as a **type stripper** -- it removes type annotations, interfaces, type aliases, generics, `as` assertions, `satisfies`, `import type`, and other erasable syntax. It works on a **per-file basis** (like `isolatedModules: true` in tsc).

**Key options:**

| Option                                | Purpose                                                                                |
| ------------------------------------- | -------------------------------------------------------------------------------------- |
| `onlyRemoveTypeImports: true`         | Required when using `verbatimModuleSyntax` -- only removes `import type` statements    |
| `rewriteImportExtensions: "rewrite"`  | Rewrites `.ts` -> `.js` in import paths (like tsc's `rewriteRelativeImportExtensions`) |
| `rewriteImportExtensions: "remove"`   | Removes extensions from import paths                                                   |
| `allowNamespaces`                     | Enable legacy namespace support (defaults to false)                                    |
| `removeClassFieldsWithoutInitializer` | For pre-ES2022 class field semantics                                                   |

**What it handles:**

- Type annotation stripping (all erasable syntax)
- `import type` / `export type` elision
- Enum transformation (generates runtime objects)
- Namespace transformation (limited -- `const` only, no `var`/`let` exports)
- Legacy (experimental) decorators + decorator metadata emission
- Import extension rewriting

**Limitations:**

- Each file transformed independently (`isolatedModules` required)
- Namespace variable exports only support `const` (not `var`/`let`)
- Decorator metadata: types requiring inference fall back to `Object`
- TC39 Stage 3 decorators: [not yet implemented](https://github.com/oxc-project/oxc/issues/9170) (legacy/experimental only)

### Relationship to `erasableSyntaxOnly`

oxc's transformer **does** support transforming enums and namespaces (non-erasable syntax), unlike simple type strippers such as Node.js's built-in type stripping (amaro/SWC). However, if your library uses `erasableSyntaxOnly: true` in tsconfig (the recommended 2026 approach), then oxc's transform becomes a pure type-erasure pass -- the fastest path.

For libraries that need enum/namespace support, oxc handles the code generation, but the recommendation remains: use `as const` objects instead of enums.

### JSX Transformation

Full JSX support with both runtime modes:

- **Automatic** (React 17+): auto-injects `jsx`/`jsxs` imports from configurable source
- **Classic**: uses `React.createElement` (configurable pragma)
- **React Refresh / Fast Refresh**: built-in support for HMR during development
- **Pure annotations**: marks JSX expressions as side-effect-free for tree-shaking
- Custom `importSource` (Preact, Solid, etc.)

### Syntax Lowering (Target Downleveling)

oxc can downlevel ESNext syntax to **ES2015** (but NOT ES5). Target syntax supported:

| ES Version | Transforms                                              |
| ---------- | ------------------------------------------------------- |
| ES2026     | Explicit Resource Management (`using`)                  |
| ES2024     | RegExp v flag                                           |
| ES2022     | Class static blocks, class fields (public/private)      |
| ES2021     | Logical assignment, numeric separators                  |
| ES2020     | Nullish coalescing, optional chaining, export namespace |
| ES2019     | Optional catch binding                                  |
| ES2018     | Object rest/spread, async iteration, RegExp features    |
| ES2017     | Async functions                                         |
| ES2016     | Exponentiation operator                                 |
| ES2015     | Arrow functions, RegExp sticky/unicode                  |

**Not supported**: ES5 target, TC39 Stage 3 decorators transform, RegExp modifiers.

Target accepts esbuild-compatible environment strings: `["chrome87", "es2022", "node18"]`.

### CommonJS Module Transform

oxc includes a `@babel/plugin-transform-modules-commonjs` equivalent, enabling ESM-to-CJS conversion. This is primarily used internally by Rolldown for CJS output rather than as a standalone feature.

## oxc_isolated_declarations: Fast .d.ts Without tsc

### What it is

`oxc_isolated_declarations` generates `.d.ts` declaration files from TypeScript source **without invoking the TypeScript compiler or its type checker**. It works purely syntactically, relying on the `isolatedDeclarations` constraint (TypeScript 5.5+) that requires explicit type annotations on all exports.

### How it works

Instead of running the full TypeScript type checker to infer types, it:

1. Parses the TypeScript source with oxc's parser
2. Extracts the public API surface (exported declarations)
3. Emits `.d.ts` output based on the explicit type annotations

This is possible because `isolatedDeclarations` requires that every exported function, variable, and class member has an explicit type annotation -- removing the need for type inference.

### Performance

| Metric        | oxc            | tsc      |
| ------------- | -------------- | -------- |
| Typical files | **40x faster** | baseline |
| Larger files  | **20x faster** | baseline |

**Real-world example**: Vue-macros reported `.d.ts` generation time reduced from **76 seconds to 16 seconds**. Airtable integrated oxc's isolated declarations emit into their Bazel CI pipeline.

### Example

**Input:**

```typescript
export function foo(a: number, b: string): number {
  return a + Number(b)
}

export enum Bar {
  a,
  b,
}
```

**Output (.d.ts):**

```typescript
export declare function foo(a: number, b: string): number
export declare enum Bar {
  a = 0,
  b = 1,
}
```

### Requirements

- `isolatedDeclarations: true` must be enabled in tsconfig.json
- All exported symbols must have explicit type annotations (functions need return types, variables need type annotations)
- This is a design constraint of the TypeScript feature itself, not specific to oxc

### Options

| Option          | Purpose                                               |
| --------------- | ----------------------------------------------------- |
| `sourcemap`     | Generate source map for the .d.ts                     |
| `stripInternal` | Remove declarations marked with `@internal` JSDoc tag |

### Available via

- **Rust crate**: `oxc_isolated_declarations` on crates.io
- **npm**: via `isolatedDeclaration()` / `isolatedDeclarationSync()` from `oxc-transform`
- **Rolldown plugin**: `rolldown-plugin-dts`
- **Universal plugin**: `unplugin-isolated-decl` (supports Vite, Rollup, esbuild, Webpack)

## oxc-resolver: Module Resolution

[oxc-resolver](https://github.com/oxc-project/oxc-resolver) is a Rust port of webpack's `enhanced-resolve`, `tsconfig-paths-webpack-plugin`, and `tsconfck`. It implements full Node.js ESM and CJS module resolution.

**Performance**: 28x faster than `enhanced-resolve`

**Key features:**

- Node.js CJS and ESM resolution algorithms
- TypeScript tsconfig.json support: path mapping, baseUrl, extends chain, project references
- Automatic tsconfig.json discovery by traversing parent directories
- In-memory file system support (via Rust `FileSystem` trait)
- 22 platform-specific native bindings

**npm package**: `oxc-resolver`
**Rust crate**: `oxc_resolver`

This is primarily used internally by Rolldown and Oxlint rather than directly by library authors. It ensures that the same resolution logic powers both the bundler and linter.

## Build Tool Integrations

### How Rolldown Uses oxc

Rolldown (the Rust-based Rollup replacement by VoidZero) uses oxc as its **foundational layer**:

- **Parser**: oxc parses all JavaScript/TypeScript source files
- **Transformer**: oxc handles TypeScript stripping, JSX compilation, and syntax lowering
- **Resolver**: oxc resolves module imports
- **Minifier**: oxc minifies output (replacing Terser)

With `rolldown-vite` (Vite 8), esbuild is no longer required. All transformation and minification is handled by oxc through Rolldown. This resulted in 3x to 16x build time improvements across projects.

### tsdown (Library Bundler)

[tsdown](https://tsdown.dev/) is the spiritual successor to tsup, built on Rolldown. Its `.d.ts` strategy:

1. If `isolatedDeclarations: true` in tsconfig -> uses **oxc-transform** (extremely fast)
2. If not -> falls back to **tsc** (slower but more compatible)

tsdown uses `rolldown-plugin-dts` internally for declaration file generation and bundling.

### Vite 8 / rolldown-vite

Vite 8 replaced its dual esbuild+Rollup architecture with Rolldown (which uses oxc). Every Vite 8 user is now indirectly using oxc for TypeScript/JSX transformation.

### unplugin-isolated-decl

A universal plugin for generating `.d.ts` via isolated declarations across build tools:

- Supports **Vite, Rollup, esbuild, Webpack**
- Configurable transformer backend: `"oxc"` (default), `"swc"`, or `"typescript"`
- Zero configuration out of the box

### unplugin-oxc

Universal plugin providing oxc transformation for any build tool.

### oxc-webpack-loader

Drop-in replacement for `swc-loader` and `babel-loader` using oxc.

## Comparison: oxc vs SWC vs esbuild

| Feature                | oxc                         | SWC                      | esbuild                   |
| ---------------------- | --------------------------- | ------------------------ | ------------------------- |
| **Language**           | Rust                        | Rust                     | Go                        |
| **TS transform speed** | 3-5x faster than SWC        | baseline                 | Similar to SWC            |
| **Memory usage**       | 20% less than SWC           | baseline                 | Similar                   |
| **Package size**       | ~2 MB                       | ~37 MB                   | ~9 MB                     |
| **.d.ts generation**   | Yes (isolated declarations) | No                       | No                        |
| **JSX transform**      | Yes (+ React Refresh)       | Yes (+ React Refresh)    | Yes                       |
| **Decorator support**  | Legacy/experimental only    | Legacy + TC39 Stage 3    | TC39 Stage 3              |
| **Syntax lowering**    | ES2015+                     | ES5+                     | ES2015+                   |
| **CommonJS output**    | Yes                         | Yes                      | Yes                       |
| **Bundling**           | Via Rolldown                | Via swcpack (limited)    | Built-in                  |
| **Standalone use**     | npm package + Rust crate    | npm package + Rust crate | npm package + Go binary   |
| **Ecosystem role**     | Powers Rolldown/Vite        | Powers Next.js/Turbopack | Powers tsup/Vite (legacy) |

### Key differentiators for oxc

1. **Isolated declarations .d.ts emit** -- neither SWC nor esbuild can generate .d.ts files
2. **Integrated into Rolldown/Vite** -- one unified pipeline instead of multiple tools
3. **Smallest package size** -- 2 MB vs 37 MB for SWC
4. **Import extension rewriting** -- native `.ts` -> `.js` rewriting

### Where SWC/esbuild still win

1. **SWC**: TC39 Stage 3 decorator support, ES5 target support, more mature standalone API
2. **esbuild**: Built-in bundling, ES5 target (limited), more battle-tested in production
3. **Both**: Longer production track record, larger standalone user base

## The VoidZero / oxc Ecosystem

```
VoidZero (company)
  |
  +-- Vite (build tool / dev server)
  |     |
  |     +-- Vite 8: uses Rolldown as bundler
  |     +-- Vite+: upcoming unified toolchain (early 2026 preview)
  |           includes: vite lib, vite run, vite ui
  |
  +-- Vitest (test runner, uses Vite)
  |
  +-- Rolldown (Rust bundler, Rollup replacement, RC status)
  |     |
  |     +-- Uses oxc parser, transformer, resolver, minifier
  |     +-- Rollup plugin compatible
  |
  +-- oxc (Rust compiler infrastructure)
  |     |
  |     +-- oxc-parser (AST parser, ESTree-compatible)
  |     +-- oxc-transform (TS/JSX transformer + isolated declarations)
  |     +-- oxc-resolver (module resolution)
  |     +-- oxc-minifier
  |     +-- Oxlint 1.0 (linter, production-ready)
  |     +-- Oxfmt (formatter, beta)
  |
  +-- tsdown (library bundler, built on Rolldown)
```

All projects are open source under MIT. Boshen (oxc creator) is VP Engineering at VoidZero. Evan You (Vite/Vue creator) founded VoidZero.

## Current Status (March 2026)

| Component                  | Status                                                                     |
| -------------------------- | -------------------------------------------------------------------------- |
| oxc-parser                 | Stable, used in production by Rolldown/Vite 8                              |
| oxc-transform (TypeScript) | Mature, used in production via Rolldown; npm package at v0.112.0 (pre-1.0) |
| oxc-transform (JSX)        | Mature, includes React Refresh                                             |
| oxc-transform (lowering)   | Functional ES2015+, no ES5, no TC39 decorators                             |
| oxc_isolated_declarations  | Mature, used by tsdown, Vue-macros, Airtable                               |
| oxc-resolver               | Stable, 28x faster than enhanced-resolve                                   |
| Oxlint                     | 1.0 stable release                                                         |
| Oxfmt                      | Beta (February 2026)                                                       |
| Rolldown                   | Release Candidate                                                          |
| tsdown                     | v0.21.x, pre-1.0 but actively used                                         |

**Bottom line**: The transformer is **production-ready when used through Rolldown/Vite 8/tsdown**. As a standalone npm package, it is pre-1.0 with rapid iteration (~weekly releases). The API is not yet fully stabilized, but the core transformation logic is battle-tested through its integration into the VoidZero toolchain.

## Practical Guidance for Library Authors

### When to use oxc transforms directly

- **Through tsdown**: The recommended path. tsdown uses oxc's isolated declarations for fast .d.ts generation when `isolatedDeclarations: true` is set.
- **Through unplugin-isolated-decl**: For custom build pipelines that need fast .d.ts generation.
- **Through Vite 8 library mode**: oxc handles all transforms automatically.

### When to use oxc-transform directly

- Custom build pipelines where you need programmatic TypeScript/JSX transformation
- CI/CD pipelines that need fast .d.ts generation (e.g., Airtable's Bazel integration)
- Replacing swc-loader or babel-loader in Webpack (via oxc-webpack-loader)

### When NOT to use oxc

- If you need TC39 Stage 3 decorators (use SWC or esbuild)
- If you need ES5 output (use SWC or esbuild)
- If you need full tsc-compatible .d.ts without `isolatedDeclarations` (use tsc or tsgo)

## Key Takeaways

- oxc is the **compiler infrastructure** powering the VoidZero toolchain (Rolldown, Vite 8, tsdown)
- Its isolated declarations feature enables **40x faster .d.ts generation** compared to tsc
- The transformer is 3-5x faster than SWC with 80% smaller package size
- For library authors, the primary touchpoint is through **tsdown** (which uses oxc via Rolldown)
- The `oxc-transform` npm package provides direct access for custom build pipelines
- With `erasableSyntaxOnly: true`, oxc's transform becomes pure type erasure -- the fastest possible path
- The ecosystem is converging: Vite 8 users are already using oxc without knowing it
- Pre-1.0 as a standalone package, but production-proven through Rolldown/Vite integration

## Open Questions & Limitations

- **TC39 Stage 3 decorators**: Not yet supported ([tracking issue](https://github.com/oxc-project/oxc/issues/9170)); legacy decorators work
- **ES5 target**: Not supported and unlikely to be added (ES2015 minimum)
- **Standalone CLI**: No dedicated CLI tool for oxc-transform; use programmatic API or through build tools
- **1.0 timeline**: No announced date for oxc-transform 1.0; the npm package ships weekly pre-release versions
- **tsconfig integration**: Some tsconfig options not yet fully supported in isolated declarations ([issue #3958](https://github.com/oxc-project/oxc/issues/3958))

## Sources

1. [oxc official site](https://oxc.rs/) -- project overview and documentation
2. [oxc Transformer docs](https://oxc.rs/docs/guide/usage/transformer.html) -- transformer usage guide
3. [oxc TypeScript transform docs](https://oxc.rs/docs/guide/usage/transformer/typescript) -- TS-specific options
4. [oxc Isolated Declarations docs](https://oxc.rs/docs/guide/usage/transformer/isolated-declarations) -- .d.ts emit
5. [oxc Syntax Lowering docs](https://oxc.rs/docs/guide/usage/transformer/lowering) -- target downleveling
6. [oxc JSX docs](https://oxc.rs/docs/guide/usage/transformer/jsx) -- JSX transformation
7. [oxc Transformer Alpha blog post](https://oxc.rs/blog/2024-09-29-transformer-alpha.html) -- original announcement with benchmarks
8. [oxc-transform on npm](https://www.npmjs.com/package/oxc-transform) -- npm package
9. [oxc-resolver on GitHub](https://github.com/oxc-project/oxc-resolver) -- resolver docs
10. [oxc GitHub repository](https://github.com/oxc-project/oxc) -- 18.3k stars
11. [Announcing Rolldown-Vite](https://voidzero.dev/posts/announcing-rolldown-vite) -- how oxc powers Vite 8
12. [VoidZero January 2026 recap](https://voidzero.dev/posts/whats-new-jan-2026) -- latest ecosystem updates
13. [tsdown .d.ts documentation](https://tsdown.dev/options/dts) -- how tsdown uses oxc-transform
14. [rolldown-plugin-dts](https://github.com/sxzz/rolldown-plugin-dts) -- Rolldown plugin for .d.ts bundling
15. [unplugin-isolated-decl](https://github.com/unplugin/unplugin-isolated-decl) -- universal isolated declarations plugin
16. [Announcing VoidZero](https://voidzero.dev/posts/announcing-voidzero-inc) -- company and ecosystem structure
17. [Announcing Vite+](https://voidzero.dev/posts/announcing-vite-plus) -- unified toolchain vision
18. [TC39 decorators tracking issue](https://github.com/oxc-project/oxc/issues/9170) -- decorator support status
19. [Oxlint 1.0 announcement](https://voidzero.dev/posts/announcing-oxlint-1-stable) -- production-ready linter
20. [Oxfmt Beta](https://oxc.rs/blog/2026-02-24-oxfmt-beta.html) -- formatter status

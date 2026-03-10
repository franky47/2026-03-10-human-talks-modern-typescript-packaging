# The `module-sync` Condition in Node.js Package Exports

## Executive Summary

The `module-sync` condition is a package.json exports map condition introduced in **Node.js 22.10.0** (September 2024). It allows packages to point both `require()` and `import` consumers to the **same ES module file**, provided that module does not use top-level `await`. This eliminates the need to ship separate CJS and ESM builds and avoids the infamous **dual-package hazard**. It was designed alongside Node.js's `require(esm)` feature, which is now stable across all supported LTS lines (v20.19.0+, v22.12.0+).

## What Is the `module-sync` Condition?

`module-sync` is a [conditional export](https://nodejs.org/api/packages.html) that matches **regardless** of whether a package is loaded via `import`, `import()`, or `require()`. The target file must be an ES module that contains **no top-level `await`** anywhere in its module graph. If it does, `require()` will throw `ERR_REQUIRE_ASYNC_MODULE`.

It was implemented by [Joyee Cheung](https://joyeecheung.github.io/blog/) in [PR #54648](https://github.com/nodejs/node/pull/54648) and shipped in [Node.js 22.10.0](https://nodejs.org/en/blog/release/v22.10.0).

## How It Differs from `import` and `require`

| Condition | `import` | `import()` | `require()` | Expected format | Top-level `await` |
|---------------|----------|------------|-------------|-----------------|-------------------|
| `import` | Yes | Yes | No | Any | Allowed |
| `require` | No | No | Yes | Any | Allowed |
| `module-sync` | Yes | Yes | Yes | ESM only | **Not allowed** |

Key distinctions:

- **`import`** and **`require`** are mutually exclusive -- a consumer uses one or the other.
- **`module-sync`** is universal: it matches for all three loading mechanisms. Both `import` and `require()` resolve to the same ESM file, which means only one copy of the module is ever loaded.

## What Problem Does It Solve?

### The Dual-Package Hazard

When a package ships both CJS (`require` condition) and ESM (`import` condition) builds, the same package can be loaded **twice** in the same process -- once as CJS and once as ESM. This causes:

- **Duplicated state** (singletons, registries, caches are not shared)
- **`instanceof` checks failing** across the two copies
- **Larger `node_modules`** since two builds must be shipped

### Why Not Reuse the `module` Condition?

Bundlers (webpack, Rollup, esbuild) already use a de-facto `"module"` condition. Node.js intentionally created a **new** condition name because existing packages using `"module"` sometimes rely on **CJS-style resolution** for their ESM files (e.g., `import "./noext"` or `import "./dir"`), which bundlers allow but Node.js's native ESM resolver does not. Using a new name avoids breaking those packages. ([Source: Joyee Cheung on X](https://x.com/JoyeeCheung/status/1838839922855211164))

## How to Use It in Practice

### Recommended: ESM-first with CJS fallback for older Node.js

```jsonc
{
  "name": "my-library",
  "type": "module",
  "exports": {
    ".": {
      "module-sync": "./dist/index.js",  // ESM for Node.js >= 22.10 (or 20.19)
      "default": "./dist/index.cjs"       // CJS fallback for older Node.js
    }
  }
}
```

On Node.js versions that support `require(esm)`, the `module-sync` condition is active. Both `require('my-library')` and `import 'my-library'` resolve to `./dist/index.js` (ESM). On older Node.js, `module-sync` is not recognized, so `default` kicks in and serves CJS.

### Supporting Both Bundlers and Node.js

During the transition period, use **both** `module-sync` and `module` pointing to the same file:

```jsonc
{
  "name": "my-library",
  "type": "module",
  "exports": {
    ".": {
      "module-sync": "./dist/index.js",  // Node.js with require(esm)
      "module": "./dist/index.js",        // Bundlers (webpack, esbuild, etc.)
      "default": "./dist/index.cjs"       // Older Node.js
    }
  }
}
```

Node.js [recommends](https://nodejs.org/api/packages.html) that bundlers should **not** implement `module-sync` and instead continue using `module`. The two conditions can coexist.

### Condition priority order

Conditions are matched top-to-bottom. The recommended order is:

1. `module-sync` (most specific, new Node.js)
2. `module` (bundlers)
3. `import` (ESM fallback)
4. `require` (CJS fallback)
5. `default` (catch-all)

## Node.js Version Support

| Version | `require(esm)` Status | `module-sync` Available |
|-------------|-------------------------------|-------------------------|
| **v20.17+** | Behind `--experimental-require-module` flag | No |
| **v20.19.0**| Unflagged, enabled by default | **Yes** |
| **v22.0-22.9** | Behind `--experimental-require-module` flag | No |
| **v22.10.0**| First version with `module-sync` (still flagged) | **Yes** |
| **v22.12.0**| Unflagged, enabled by default | **Yes** |
| **v23.0.0+**| Enabled by default | **Yes** |
| **v25.x** | Stable | **Yes** |

The `module-sync` condition is automatically active when `require(esm)` is enabled (i.e., when `--no-require-module` is **not** set). You can detect support at runtime with `process.features.require_module`.

Backport to v20.19.0 was notable -- an exception was made because of the feature's ecosystem importance, even though v20 was approaching maintenance mode. ([Node.js 20.19.0 release notes](https://nodejs.org/en/blog/release/v20.19.0))

## Adoption Status

Adoption is **early but growing** (as of early 2026):

- The Node.js documentation officially recommends `module-sync` as the preferred approach for dual packages.
- **Bun** does not yet support the `module-sync` condition natively (there is an [open feature request](https://github.com/oven-sh/bun/issues/20768) from July 2025), though Bun has supported synchronous `require()` of ESM for some time.
- **Vite** has discussed adding `module-sync` to its default SSR external conditions ([Issue #19201](https://github.com/vitejs/vite/issues/19201)).
- Package authors are beginning to adopt it, particularly those who previously shipped dual CJS+ESM builds.

## Relationship to `require(esm)`

`module-sync` exists **because of** `require(esm)`. The timeline:

1. **ES modules were designed to be only conditionally async** -- they are synchronous unless the graph contains top-level `await`.
2. **Node.js v22.0** added experimental `require(esm)` behind a flag, allowing `require()` to load synchronous ESM.
3. **Node.js v22.10** added the `module-sync` condition so packages could signal "this ESM file is safe to `require()`."
4. **Node.js v22.12 / v23.0** unflagged `require(esm)`, making `module-sync` active by default.
5. **Node.js v20.19** backported the feature to the v20 LTS line.
6. **Late 2025**: `require(esm)` was marked **stable**.

The `module-sync` condition is the **package-level opt-in mechanism** that makes `require(esm)` practical for the ecosystem. Without it, there was no way for a package to say "serve ESM to both `require()` and `import` consumers on modern Node.js, but fall back to CJS on older versions."

## Key Takeaways

- **`module-sync` unifies ESM delivery** for both `require()` and `import` on modern Node.js, eliminating the dual-package hazard.
- **Introduced in Node.js 22.10.0**, now available on all supported LTS lines (v20.19.0+, v22.12.0+).
- The target ESM file **must not use top-level `await`** anywhere in its module graph.
- It is intentionally **different from the bundler `module` condition** due to resolution semantics differences. Use both during transition.
- As `require(esm)` is now stable, `module-sync` is the recommended way to publish packages that work everywhere.
- Bun does not yet support it natively; Deno support status is unclear.

## Sources

1. [Node.js Packages Documentation (v25.8.0)](https://nodejs.org/api/packages.html) -- Official documentation on conditional exports including `module-sync`
2. [PR #54648: Implement the "module-sync" exports condition](https://github.com/nodejs/node/pull/54648) -- Original implementation by Joyee Cheung
3. [Node.js 22.10.0 Release Notes](https://nodejs.org/en/blog/release/v22.10.0) -- First release containing `module-sync`
4. [Node.js 20.19.0 Release Notes](https://nodejs.org/en/blog/release/v20.19.0) -- Backport of `require(esm)` and `module-sync` to v20
5. [require(esm) in Node.js: from experiment to stability -- Joyee Cheung](https://joyeecheung.github.io/blog/2025/12/30/require-esm-in-node-js-from-experiment-to-stability/) -- Comprehensive history and rationale
6. [require(esm) in Node.js -- Joyee Cheung](https://joyeecheung.github.io/blog/2024/03/18/require-esm-in-node-js/) -- Original blog post on the feature
7. [Joyee Cheung on X re: why "module-sync" instead of "module"](https://x.com/JoyeeCheung/status/1838839922855211164) -- Explains naming decision
8. [Bun: module-sync condition support? (Issue #20768)](https://github.com/oven-sh/bun/issues/20768) -- Bun feature request
9. [Vite: Add module-sync to SSR conditions (Issue #19201)](https://github.com/vitejs/vite/issues/19201) -- Vite integration discussion
10. [require(esm) Backported to Node.js 20 -- Socket.dev](https://socket.dev/blog/require-esm-backported-to-node-js-20) -- Coverage of the v20 backport
11. [Publishing dual module ESM libraries -- satya164](https://satya164.page/posts/publishing-dual-module-esm-libraries) -- Practical guidance on dual-package publishing
12. [Tracking issue: require(esm) (#52697)](https://github.com/nodejs/node/issues/52697) -- Master tracking issue

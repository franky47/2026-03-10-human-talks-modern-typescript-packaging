# ESM vs CJS — Decisions

## The landscape in 2026

- **Node.js 18** (last LTS without good ESM support) reached **EOL April 2025**
- **Node.js 22+** supports `require()` of ESM modules (no top-level `await`)
- **Node.js v25.4.0** made `require(esm)` **stable** (no longer experimental)
- **Bun** and **Deno** are ESM-native
- All major bundlers handle ESM natively

## Recommendation: ship ESM-only

Set `"type": "module"` in package.json. Dual CJS+ESM is unnecessary overhead:

- `require(esm)` eliminates the primary reason dual publishing existed
- Dual publishing risks the **"dual package hazard"** (same package loaded twice with separate state)
- Requires separate `.d.ts` and `.d.cts` type declarations
- Sindre Sorhus and many prolific authors have been ESM-only since 2021

**Only dual-publish if** you have known CJS consumers on Node.js < 22.

## The `module-sync` condition (advanced)

For packages that still need to support both `require()` and `import` consumers during the transition:

```jsonc
{
  "exports": {
    ".": {
      "module-sync": "./dist/index.js",  // ESM for Node.js >= 22.10
      "default": "./dist/index.cjs"       // CJS fallback for older Node.js
    }
  }
}
```

`module-sync` matches for both `require()` and `import`, pointing to a single ESM file. Requires: no top-level `await` in the module graph. Introduced in Node.js 22.10.0, backported to 20.19.0.

**When to use**: libraries that want a single ESM source but need CJS fallback for old Node.js. For most new libraries, ESM-only with `"default"` is sufficient.

## Dual package hazard

When a package ships both CJS and ESM builds, it can be loaded twice in the same process:
- **Duplicated state** (singletons, registries, caches not shared)
- **`instanceof` checks fail** across the two copies
- **Larger `node_modules`**

This is why ESM-only is preferred. If you must dual-publish, ensure both entry points share state or use the `module-sync` condition.

## File extensions

| `type` field | `.js` / `.ts` | `.mjs` / `.mts` | `.cjs` / `.cts` |
|---|---|---|---|
| `"module"` | ESM | ESM | CJS |
| `"commonjs"` or absent | CJS | ESM | CJS |

`.mjs`/`.mts` and `.cjs`/`.cts` **always override** the `type` field.

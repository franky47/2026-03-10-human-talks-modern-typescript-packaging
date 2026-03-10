---
name: typescript-library-packaging
description: >
  Guide for packaging TypeScript libraries for npm, JSR, and monorepos.
  Covers ESM, exports maps, tsconfig, build tools, and TypeScript 6.0.
  Use when setting up a new TypeScript library, configuring package.json exports,
  choosing build tools (tsc, tsdown, tsup, unbuild), writing tsconfig.json for a library,
  migrating to TypeScript 6.0, enabling erasableSyntaxOnly, publishing to npm or JSR,
  or validating a package before publishing.
---

# TypeScript Library Packaging (2026)

## The 2026 defaults

### Public npm library (new project)

```
ESM-only  ·  "type": "module"  ·  exports map with "types" first
tsdown or tsc for build  ·  .d.ts + .d.ts.map for types
moduleResolution: "nodenext"  ·  target: "es2022"
erasableSyntaxOnly: true  ·  verbatimModuleSyntax: true
Validate with publint + arethetypeswrong
```

### Internal monorepo package

```
No build step  ·  exports point to raw .ts source
private: true  ·  workspace:* protocol (pnpm)
App's bundler handles transpilation
```

## Quick decision tree

### Module format

- **Ship ESM-only.** `require(esm)` is stable in Node.js 22+. Dual CJS+ESM is unnecessary overhead.
- Only dual-publish if you have known CJS consumers on Node.js < 22.

### Build strategy

- **Single entry-point** -> bundle with tsdown (or tsup)
- **Multi-export library** -> file-to-file with tsc or unbuild mkdist
- **Internal monorepo package** -> no build at all (raw .ts)
- **Never minify library code** (let the app bundler do it)

### Raw .ts on npm?

- **No.** Node.js blocks `.ts` from `node_modules/`. Transpile to JS + `.d.ts`.
- Internal monorepo packages can use raw `.ts` (bundler handles it).
- JSR accepts raw `.ts` directly.

### moduleResolution

- **Libraries**: `nodenext`
- **Apps**: `bundler`

## Minimal package.json

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

## Minimal tsconfig.json

```jsonc
{
  "compilerOptions": {
    "strict": true,
    "module": "NodeNext",
    "moduleDetection": "force",
    "verbatimModuleSyntax": true,
    "erasableSyntaxOnly": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "target": "es2022",
    "lib": ["es2022"],
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "skipLibCheck": true,
  },
  "include": ["src"],
}
```

## Exports map rules

1. `"types"` condition **MUST be first** (most common mistake)
2. `"default"` condition should be last (fallback)
3. Once you add `exports`, unlisted deep imports throw `ERR_PACKAGE_PATH_NOT_EXPORTED`
4. `.mjs` output needs `.d.mts` types; `.cjs` needs `.d.cts`

See [exports-map.md](references/exports-map.md) for patterns (subpaths, wildcards, dual CJS/ESM, monorepo).

## Validation (always run before publishing)

```bash
npx publint                        # lint package.json
npx @arethetypeswrong/cli my-pkg   # validate type resolution
npm pack --dry-run                 # preview published files
```

## References

| Topic                                       | File                                                    |
| ------------------------------------------- | ------------------------------------------------------- |
| Exports map patterns                        | [exports-map.md](references/exports-map.md)             |
| tsconfig deep dive                          | [tsconfig.md](references/tsconfig.md)                   |
| ESM vs CJS decisions                        | [esm-cjs.md](references/esm-cjs.md)                     |
| Build tools comparison                      | [build-tools.md](references/build-tools.md)             |
| erasableSyntaxOnly & Node.js type stripping | [erasable-syntax.md](references/erasable-syntax.md)     |
| Monorepo setup                              | [monorepo.md](references/monorepo.md)                   |
| TypeScript 6.0 migration                    | [typescript-6.md](references/typescript-6.md)           |
| Registries (npm, JSR, vlt)                  | [registries.md](references/registries.md)               |
| Import extensions & module resolution       | [import-extensions.md](references/import-extensions.md) |
| oxc transformation tools                    | [oxc-transforms.md](references/oxc-transforms.md)       |

## Key principles

- **ESM-only is the default.** CJS compatibility via `require(esm)` is stable.
- **`"types"` first in exports.** Always.
- **Ship `.d.ts.map` declaration maps.** Enables "Go to Definition" into source.
- **Enable `erasableSyntaxOnly`.** Aligns with Node.js type stripping and TC39 direction.
- **Use `as const` objects instead of enums.** Better compatibility, same type safety.
- **Validate before publishing.** publint + arethetypeswrong catch most issues.

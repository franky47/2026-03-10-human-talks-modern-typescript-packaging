# TypeScript 6.0 Migration

**Status**: RC released March 6, 2026. Final release: March 17, 2026.
**Note**: TS 6.0 is the **last major version written in TypeScript**. TS 7.0 will be the native Go port (tsgo).

## New defaults (no config needed)

| Option | Old Default | TS 6.0 Default |
|---|---|---|
| `strict` | `false` | **`true`** |
| `target` | `es3`/`es5` | **Latest ES** |
| `module` | `commonjs` | **`esnext`** |
| `esModuleInterop` | `false` | **always `true`** (cannot be `false`) |
| `noUncheckedSideEffectImports` | `false` | **`true`** |
| `types` array | Auto-include all `@types/*` | **Empty** (explicit inclusion required) |

## Deprecated/removed

- **ES5 target** -> minimum is ES2015
- **`moduleResolution: node10`** and **`classic`** -> deprecated
- **`module: amd/system/umd`** -> deprecated
- **`baseUrl`** for path resolution -> deprecated (use `paths` or `#/` subpath imports)
- **Import assertions (`assert {}`)** -> removed (use `with {}` import attributes)
- **`allowSyntheticDefaultImports`** cannot be set to `false`
- **`outFile`** deprecated
- All code assumed to be in JS strict mode

## Action required

1. **Add `"types": ["node"]`** to tsconfig (no longer auto-included)
2. **Migrate from `node10`/`classic` resolution** to `nodenext` or `bundler`
3. **Replace `assert {}` with `with {}`** in import attributes
4. Use `"ignoreDeprecations": "6.0"` as an escape hatch during migration

## Notable features from TS 5.5–6.0

| Feature | Version | Impact on libraries |
|---|---|---|
| `isolatedDeclarations` | TS 5.5 | Fast `.d.ts` generation without full type-checking; enables parallel declaration emit |
| `rewriteRelativeImportExtensions` | TS 5.7 | Allows `.ts` imports that get rewritten to `.js` in output |
| `erasableSyntaxOnly` | TS 5.8 | Blocks non-erasable syntax; aligns with Node.js type stripping |
| `#/` subpath imports | TS 6.0 | Package-internal imports via `#/` prefix |
| `module: "commonjs"` + `moduleResolution: "bundler"` | TS 6.0 | Previously disallowed combo, now permitted |

## Migration guide

https://gist.github.com/privatenumber/3d2e80da28f84ee30b77d53e1693378f

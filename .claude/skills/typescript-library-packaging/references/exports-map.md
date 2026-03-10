# Exports Map — Deep Dive

## Why `exports` over `main`

- **Multiple entry points** (subpath exports)
- **Encapsulation** — only listed paths accessible; unlisted paths throw `ERR_PACKAGE_PATH_NOT_EXPORTED`
- **Conditional resolution** — ESM/CJS/types/browser/node
- **Wildcard patterns**

## Condition ordering (CRITICAL)

```json
{
  "exports": {
    ".": {
      "types": "...",    // 1. ALWAYS FIRST (TypeScript)
      "browser": "...",  // 2. Environment-specific
      "node": "...",     // 3. Environment-specific
      "import": "...",   // 4. Module format
      "require": "...",  // 5. Module format
      "default": "..."   // 6. ALWAYS LAST (fallback)
    }
  }
}
```

**`"types"` MUST be first** — the #1 most common mistake in TypeScript library packaging.

## Common patterns

### ESM-only (simplest, recommended)

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

### Dual CJS/ESM (only if needed)

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

### Multiple subpaths

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

### Wildcard exports

```json
{
  "exports": {
    ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" },
    "./*": { "types": "./dist/*.d.ts", "default": "./dist/*.js" }
  }
}
```

### Internal monorepo package (no build)

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

## The `imports` field (private internal mappings)

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

## Common pitfalls

1. **`"types"` not first** — TypeScript won't find your types
2. **Mismatched `.d.ts` extensions** — `.mjs` output needs `.d.mts` types, `.cjs` needs `.d.cts`
3. **Missing `"default"` fallback** — reduced compatibility
4. **Forgetting encapsulation** — once you add `exports`, unlisted deep imports break
5. **Not exporting `./package.json`** — some tools need it

## Legacy/outdated fields

| Field | Status | Include? |
|---|---|---|
| `main` | Legacy fallback | Optional, for old tools |
| `types` (top-level) | Legacy fallback | Optional, for `node10` users |
| `module` | Non-standard | **No** |
| `typesVersions` | Obsolete workaround | **No** |

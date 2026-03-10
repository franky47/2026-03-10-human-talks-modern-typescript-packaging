# Import Extensions & Module Resolution

## How to write imports

The answer depends on whether you're **compiling to JS** or **running .ts directly**:

```typescript
// Scenario A: Compiling with tsc (moduleResolution: "nodenext")
// Use .js extension — TypeScript resolves .js -> .ts at compile time
import { foo } from './utils/helper.js';  // Works after compilation

// Scenario B: Running .ts directly with Node.js v24+ type stripping
// Use .ts extension — the actual file IS .ts
import { foo } from './utils/helper.ts';  // Works in Node.js v24+

// These don't work:
import { bar } from './utils/helper';     // Fails in Node.js ESM (no extension)
import { baz } from './utils/helper.js';  // Fails if running .ts directly (no .js file)
```

**Key insight**: The `.js` extension convention exists because `tsc` emits `.js` files and keeps the import specifier as-is. When running `.ts` natively, you use `.ts` extensions because that's the actual file.

## tsconfig flags for `.ts` extensions in imports

### `allowImportingTsExtensions` (TS 5.0)

Allows `import { foo } from './foo.ts'` in source code. **Requires `noEmit` or `emitDeclarationOnly`** because tsc wouldn't know how to rewrite the extension. Use when a bundler or runtime handles execution.

### `rewriteRelativeImportExtensions` (TS 5.7)

More powerful: allows `.ts` imports AND rewrites them to `.js` in emitted output. **Does NOT require `noEmit`**. When enabled, `allowImportingTsExtensions` defaults to `true` automatically.

```jsonc
// Option A: No emit (bundler/runtime handles it)
{
  "compilerOptions": {
    "allowImportingTsExtensions": true,
    "noEmit": true
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

With `rewriteRelativeImportExtensions`, you can write natural `.ts` imports that work both when running directly (Node.js v24+, Bun, Deno) AND when compiled to JS.

## Cross-runtime compatibility

| Import style | Node.js ESM (compiled JS) | Node.js ESM (running .ts) | Bun | Deno |
|---|---|---|---|---|
| `./foo` (no ext) | No | No | Yes | No |
| `./foo.js` | Yes | No (no .js file) | Yes | Yes |
| `./foo.ts` | No (no .ts in dist) | Yes (v24+) | Yes | Yes |

## File extensions

### Source extensions

| Extension | Module format | When to use |
|---|---|---|
| `.ts` | Determined by `type` field | Default for most files |
| `.mts` | Always ESM | ESM file in a CJS package |
| `.cts` | Always CJS | CJS file in an ESM package |

### Output mapping

| Source | JS output | Declaration output |
|---|---|---|
| `.ts` (ESM context) | `.js` | `.d.ts` |
| `.mts` | `.mjs` | `.d.mts` |
| `.cts` | `.cjs` | `.d.cts` |

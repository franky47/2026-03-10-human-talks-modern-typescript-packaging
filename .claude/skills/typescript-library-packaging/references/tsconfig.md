# tsconfig.json — Deep Dive

## Public library (tsc, Node.js)

```jsonc
{
  "compilerOptions": {
    // Strictness
    "strict": true,                    // Default in TS 6.0
    "noUncheckedIndexedAccess": true,  // NOT included in strict
    "noImplicitOverride": true,        // NOT included in strict

    // Module system
    "module": "NodeNext",
    "moduleDetection": "force",
    "verbatimModuleSyntax": true,
    "erasableSyntaxOnly": true,
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
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

## Internal monorepo package

Same as above, plus:

```jsonc
{
  "compilerOptions": {
    "composite": true  // Enables project references + incremental builds
  }
}
```

## Application (not a library)

Same base, but remove `declaration`, `declarationMap`, `composite`.

## Bundler-based library (Vite/esbuild handles emit)

```jsonc
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "noEmit": true  // tsc only type-checks; bundler emits JS
    // For declarations: replace noEmit with emitDeclarationOnly + declaration
  }
}
```

## Key options explained

| Option | Why |
|---|---|
| `module: "NodeNext"` | Correct Node.js ESM/CJS semantics, supports `exports` field |
| `verbatimModuleSyntax` | Forces `import type` for type-only imports; replaces legacy flags |
| `erasableSyntaxOnly` | Blocks enums/namespaces/parameter properties; aligns with Node.js type stripping |
| `isolatedModules` | Ensures each file can be independently transpiled (esbuild/swc compat) |
| `moduleDetection: "force"` | Treats every file as a module |
| `declaration + declarationMap` | Generates `.d.ts` + enables "Go to Definition" into source |
| `skipLibCheck` | Skip type-checking `node_modules` `.d.ts` files (performance) |

## moduleResolution settings

| Setting | `exports` support | Requires extensions | Best for |
|---|---|---|---|
| `node10` | No | No | **Never use** (deprecated in TS 6.0) |
| `node16` | Yes | Yes (ESM) | Libraries targeting old Node.js |
| `nodenext` | Yes | Yes (ESM) | **Libraries** (recommended) |
| `bundler` | Yes | No | **Apps** processed by a bundler |

## `@total-typescript/tsconfig`

Pre-built configs organized by transpiler x environment x project type:

```json
{ "extends": "@total-typescript/tsconfig/tsc/no-dom/library" }
```

12 variants: `tsc`/`bundler` x `dom`/`no-dom` x `app`/`library`/`library-monorepo`.

## Source maps & declaration maps

| Type | File | Purpose |
|---|---|---|
| Runtime source maps | `.js.map` | Map transpiled JS to original TS for stack traces |
| Declaration maps | `.d.ts.map` | Map `.d.ts` to original `.ts` for "Go to Definition" |
| Inline source maps | embedded in `.js` | **Not recommended for libraries** |

- **Always ship `.d.ts.map`** — huge DX improvement for consumers
- **Include `.ts` source** in the package if you ship declaration maps
- **Never minify library code**

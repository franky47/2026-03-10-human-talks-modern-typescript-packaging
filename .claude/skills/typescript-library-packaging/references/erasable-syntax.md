# erasableSyntaxOnly & Node.js Type Stripping

## What is `erasableSyntaxOnly`?

A TypeScript 5.8 compiler option that errors on any syntax that cannot be removed by simply deleting it — syntax that requires code generation rather than plain erasure. Created in collaboration between the TypeScript team and the Node.js core team.

## Blocked constructs

| Construct | Example | Alternative |
|---|---|---|
| `enum` (all forms, including `const enum`) | `enum Dir { Up, Down }` | `as const` object (see below) |
| `namespace` with runtime code | `namespace Utils { export function add() {} }` | ES modules |
| Parameter properties | `constructor(public x: number)` | Explicit property + assignment |
| `import =` / `export =` | `import fs = require("fs")` | Standard ES imports/exports |
| Angle-bracket assertions | `<Type>value` | `value as Type` |

## What remains allowed

All erasable constructs: type annotations, interfaces, type aliases, generics, `as` assertions, `satisfies`, `import type`, `export type`, `declare` (except `declare enum`), `abstract`, overload signatures, decorators.

## The `as const` pattern (enum replacement)

```typescript
// Instead of enum:
// enum Direction { Up = "UP", Down = "DOWN" }

// Use as const:
const Direction = {
  Up: "UP",
  Down: "DOWN",
} as const;

type Direction = (typeof Direction)[keyof typeof Direction];
// Type: "UP" | "DOWN"
```

Same type safety, pure JavaScript with an erasable type assertion.

## Why `const enum` is also blocked

Despite being inlined at compile time, `const enum` is blocked because:
1. SWC and Node's type stripper treat the `enum` keyword as universally illegal
2. Inlining still requires the tool to understand and process enum values (not simple erasure)
3. Cross-file `const enum` requires resolving values from `.d.ts` files

## Recommended companion flag: `verbatimModuleSyntax`

```jsonc
{
  "compilerOptions": {
    "erasableSyntaxOnly": true,
    "verbatimModuleSyntax": true
  }
}
```

Together, these ensure your TypeScript source is a near 1:1 representation of the JavaScript that will run.

## Node.js native type stripping

### How it works

Node.js uses [amaro](https://github.com/nodejs/amaro) (wrapping SWC) to replace TypeScript syntax with **whitespace** — preserving line numbers without source maps.

### Timeline

| Version | Milestone |
|---|---|
| v22.6.0 | `--experimental-strip-types` flag added |
| v23.6.0 | Type stripping enabled by default (no flag needed) |
| v22.18.0 | Type stripping backported to LTS, marked stable |
| v24.3.0 | Type stripping stable in Current branch |

### Key limitation

Node.js **blocks `.ts` files from `node_modules/`**. Native type stripping only works for your own `.ts` files, not dependencies. This is deliberate policy.

### `--experimental-transform-types`

For projects that need enums or other non-erasable syntax, Node.js provides this flag for actual code transformation. Remains experimental — not the recommended path.

## Who recommends `erasableSyntaxOnly`?

- **Node.js official docs** (recommended for projects using native type stripping)
- **`@tsconfig/bases`** (included in recommended Node.js configurations)
- **Matt Pocock / Total TypeScript** (TSConfig Cheat Sheet, dedicated article)
- **Rob Palmer** (TC39 delegate, submitted the Node.js docs PR)
- **Angular** (replacing enums with constant objects)

## Gradual migration for existing projects

Use [eslint-plugin-erasable-syntax-only](https://github.com/JoshuaKGoldberg/eslint-plugin-erasable-syntax-only) — provides granular per-rule enforcement. Set rules to "warn" before enabling the all-or-nothing tsconfig flag.

## Connection to TC39 Type Annotations proposal

The [TC39 Type Annotations proposal](https://github.com/tc39/proposal-type-annotations) (Stage 1) would make JavaScript engines treat type annotations as comments. `erasableSyntaxOnly` enforces exactly the subset this proposal covers. If the proposal advances, code written with this flag would run natively in browsers without any tooling.

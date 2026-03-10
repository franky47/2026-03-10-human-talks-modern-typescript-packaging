# TypeScript `erasableSyntaxOnly` -- Deep Research

## Executive Summary

The `erasableSyntaxOnly` compiler option was introduced in **TypeScript 5.8** (released February 2025). When enabled, it causes TypeScript to error on any syntax that cannot be removed by simply deleting it from the source -- that is, syntax that requires code generation or transformation rather than plain erasure. The blocked constructs are: **enums** (including `const enum`), **namespaces/modules with runtime code**, **class parameter properties**, **`import =` aliases**, **`export =` assignments**, and **legacy angle-bracket type assertions** in `.tsx`-adjacent contexts.

This flag was created in direct collaboration between the TypeScript team and the Node.js core team. Its primary motivation is compatibility with **Node.js native type stripping**, which landed as stable in Node.js v22.18.0 and v24.3.0. Node's type stripping works by replacing TypeScript syntax with whitespace -- it does not perform any code generation. Therefore, only "erasable" TypeScript constructs are supported. The flag also aligns with the **TC39 Type Annotations proposal** (Stage 1), which envisions a future where JavaScript engines treat type annotations as comments.

The flag is rapidly becoming a mainstream recommendation. It is included in the **official Node.js TypeScript documentation**, the **`@tsconfig/bases` recommended configurations**, and **Matt Pocock's TSConfig Cheat Sheet** on Total TypeScript. For new projects -- both libraries and applications -- enabling `erasableSyntaxOnly` is now considered a best practice by multiple authoritative voices in the ecosystem.

## What `erasableSyntaxOnly` Does

### The Core Concept: Erasable vs Non-Erasable Syntax

"Erasable" syntax is TypeScript-specific syntax that can be deleted from a source file and the result is still valid JavaScript with identical runtime behavior. Standard type annotations are the quintessential example:

```typescript
// ALLOWED: Type annotations are erasable
// Remove `: string` and `: number` and you have valid JS
function greet(name: string): string {
  return `Hello, ${name}`;
}

// ALLOWED: Interfaces are erasable (they produce no JS output)
interface User {
  name: string;
  age: number;
}

// ALLOWED: Type aliases are erasable
type ID = string | number;

// ALLOWED: Generics are erasable
function identity<T>(value: T): T {
  return value;
}

// ALLOWED: 'as' type assertions are erasable
const x = someValue as string;

// ALLOWED: Standalone 'type' imports/exports are erasable
import type { Foo } from './foo';
export type { Bar };
```

"Non-erasable" syntax requires the TypeScript compiler to **generate JavaScript code** that does not exist in the original source. You cannot simply delete it; you must transform it.

### Complete List of Blocked Constructs

When `erasableSyntaxOnly` is enabled, the following produce compiler errors:

#### 1. Enum Declarations (all forms)

```typescript
// BLOCKED: Regular enum (generates an IIFE with a runtime object)
enum Direction {
  Up,
  Down,
  Left,
  Right,
}
// Error: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.

// BLOCKED: String enum
enum Color {
  Red = "RED",
  Green = "GREEN",
  Blue = "BLUE",
}

// BLOCKED: const enum (even though it inlines values at compile time)
const enum Status {
  Active,
  Inactive,
}

// BLOCKED: declare enum (ambient enum)
declare enum Foo {
  A,
  B,
}
```

The rationale for blocking `const enum` despite it being "erased" at compile time is that tools like SWC and Node's type stripper treat the `enum` keyword as universally illegal -- they do not distinguish between `enum` and `const enum`. This was confirmed in [TypeScript issue #61490](https://github.com/microsoft/TypeScript/issues/61490), which was closed with the conclusion that matching SWC and Node behavior is the correct choice.

#### 2. Namespaces and Modules with Runtime Code

```typescript
// BLOCKED: Namespace with runtime values
namespace MathUtils {
  export function add(a: number, b: number) {
    return a + b;
  }
}
// Error: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.

// BLOCKED: Module keyword (legacy)
module MyModule {
  export const value = 42;
}

// ALLOWED: Namespace with only types (purely erasable)
namespace Types {
  export interface Config {
    debug: boolean;
  }
}
```

#### 3. Parameter Properties

```typescript
// BLOCKED: Parameter properties generate assignment code
class User {
  constructor(public name: string, private age: number) {}
}
// Error: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.

// ALLOWED: Explicit property declaration (the classic pattern)
class User {
  public name: string;
  private age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }
}
```

#### 4. Import Aliases (`import =`)

```typescript
// BLOCKED: import = require
import fs = require("fs");
// Error: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.

// BLOCKED: import = namespace alias
import Bar = Foo.Bar;

// ALLOWED: Standard ES module imports
import fs from "fs";
import { readFile } from "fs";
```

#### 5. Export Assignments (`export =`)

```typescript
// BLOCKED: CommonJS-style export assignment
export = myFunction;
// Error: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.

// ALLOWED: Standard ES module exports
export default myFunction;
export { myFunction };
```

#### 6. Legacy Angle-Bracket Type Assertions

```typescript
// BLOCKED in .tsx files or with erasableSyntaxOnly:
// Prefix-style (angle-bracket) assertions
const num = <number>someValue;

// ALLOWED: 'as' assertions
const num = someValue as number;
```

### What Remains Allowed

All standard TypeScript type-level constructs remain fully functional:

- Type annotations (`: Type`)
- Interfaces and type aliases
- Generics (`<T>`)
- `as` type assertions and `satisfies`
- `type` imports and exports
- `declare` statements (ambient declarations, except `declare enum`)
- Decorators (these are now a JavaScript standard)
- `abstract` classes (the `abstract` keyword is erased; the class itself is JS)
- Optional chaining, nullish coalescing (these are JS features)
- Overload signatures (only the implementation produces output)

### Recommended Companion Flag: `verbatimModuleSyntax`

The TypeScript documentation and community consistently recommend pairing `erasableSyntaxOnly` with `verbatimModuleSyntax`:

```jsonc
{
  "compilerOptions": {
    "erasableSyntaxOnly": true,
    "verbatimModuleSyntax": true
  }
}
```

`verbatimModuleSyntax` ensures that import/export statements are written exactly as they should appear in the output. It forces you to use `import type` for type-only imports, preventing runtime import of modules that only provide types. Together, these two flags ensure that your TypeScript source files are a nearly 1:1 representation of the JavaScript that will run.

## When Was It Introduced?

- **TypeScript 5.8 Beta**: January 2025 (first public availability)
- **TypeScript 5.8 Release**: February 28, 2025

The flag was announced by **Rob Palmer** (Bloomberg, TC39 delegate) on the same day as the beta release, describing it as designed to "pair with Node's built-in TypeScript support, guiding users away from TS-only runtime features."

Sources:
- [TypeScript 5.8 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html)
- [Announcing TypeScript 5.8 -- Microsoft DevBlogs](https://devblogs.microsoft.com/typescript/announcing-typescript-5-8/)
- [Rob Palmer on X (formerly Twitter)](https://x.com/robpalmer2/status/1884715044585259127)

## Relationship to Node.js Native Type Stripping

### Timeline of Node.js TypeScript Support

| Version | Date | Milestone |
|---------|------|-----------|
| v22.6.0 | 2024 | `--experimental-strip-types` flag added |
| v22.7.0 | 2024 | `--experimental-transform-types` flag added (for enums etc.) |
| v23.6.0 | Jan 2025 | Type stripping enabled by default (no flag needed), experimental warning removed |
| v22.18.0 | 2025 | Type stripping backported to LTS, marked stable |
| v24.3.0 | 2025 | Type stripping stable in Current branch |

### How Node.js Type Stripping Works

Node.js uses [amaro](https://github.com/nodejs/amaro) (which wraps SWC) to replace TypeScript syntax with whitespace. This is a fundamentally simple operation:

1. Parse the source file
2. Identify TypeScript-specific tokens (type annotations, interfaces, etc.)
3. Replace them with whitespace (preserving source maps / line numbers)
4. Execute the resulting JavaScript

This approach **cannot** handle constructs that require code generation. An `enum` does not just get deleted -- it must be transformed into a JavaScript object. A parameter property like `constructor(public x: number)` must generate `this.x = x;` in the constructor body.

### The Connection

`erasableSyntaxOnly` was explicitly created as a collaboration between the TypeScript team and Node.js core team. When you enable it:

- TypeScript will error at compile time on any syntax that Node's type stripper cannot handle
- You get immediate feedback during development rather than runtime failures in Node.js

Node.js documentation was updated via [PR #57271](https://github.com/nodejs/node/pull/57271) (submitted by Rob Palmer, merged March 4, 2025) to recommend `erasableSyntaxOnly` in the official TypeScript configuration guidance.

The current [Node.js TypeScript documentation](https://nodejs.org/api/typescript.html) recommends TypeScript 5.8+ with `erasableSyntaxOnly: true`.

### `--experimental-transform-types`

For projects that need enums or other non-erasable syntax, Node.js provides `--experimental-transform-types`. This flag tells Node to perform actual code transformation (not just stripping). However, this remains experimental and is not the recommended path.

Sources:
- [Node.js TypeScript Documentation](https://nodejs.org/api/typescript.html)
- [Node.js PR #57271 -- Recommend erasableSyntaxOnly](https://github.com/nodejs/node/pull/57271)
- [2ality -- Node's new built-in support for TypeScript](https://2ality.com/2025/01/nodejs-strip-type.html)

## `const enum` -- Allowed or Blocked?

**`const enum` is blocked by `erasableSyntaxOnly`.** This is a deliberate design decision.

### Why Block `const enum`?

At first glance, `const enum` seems like it should be allowed -- after all, the TypeScript compiler inlines the values at every usage site and removes the enum declaration entirely. There is no runtime object. However:

1. **Tool compatibility**: SWC, the engine behind Node's type stripping, treats the `enum` keyword as universally illegal. It does not differentiate between `enum` and `const enum`.
2. **Processing requirement**: Even though `const enum` does not generate a runtime object, it still requires the tool to *understand and inline* the enum values. Simple whitespace replacement is not sufficient.
3. **Cross-file issues**: `const enum` from external `.d.ts` files requires the consuming tool to resolve and inline values from declaration files, which is beyond the scope of simple type stripping.

This was discussed and confirmed in [TypeScript GitHub issue #61490](https://github.com/microsoft/TypeScript/issues/61490), which was closed as "completed" with the conclusion: matching SWC and Node behavior where the `enum` keyword is universally illegal is the correct approach.

### Recommended Alternative

```typescript
// Instead of:
const enum Direction {
  Up = "UP",
  Down = "DOWN",
  Left = "LEFT",
  Right = "RIGHT",
}

// Use an 'as const' object:
const Direction = {
  Up: "UP",
  Down: "DOWN",
  Left: "LEFT",
  Right: "RIGHT",
} as const;

type Direction = (typeof Direction)[keyof typeof Direction];
// Resulting type: "UP" | "DOWN" | "LEFT" | "RIGHT"
```

The `as const` pattern produces identical type safety and is pure JavaScript with a type-level assertion that gets erased.

Sources:
- [TypeScript Issue #61490 -- Should erasableSyntaxOnly prevent const enums?](https://github.com/microsoft/TypeScript/issues/61490)
- [Total TypeScript -- erasableSyntaxOnly](https://www.totaltypescript.com/erasable-syntax-only)

## Is It Becoming a Mainstream Recommendation?

**Yes.** The evidence is strong that `erasableSyntaxOnly` is becoming a default recommendation for new TypeScript projects.

### Who Recommends It?

1. **Node.js Official Documentation**: The [Node.js TypeScript docs](https://nodejs.org/api/typescript.html) recommend TS 5.8+ with `erasableSyntaxOnly: true` for projects using native type stripping.

2. **`@tsconfig/bases` Project**: The shared TSConfig base configurations (used by many projects as starting points) include `erasableSyntaxOnly` in the recommended Node.js configurations.

3. **Matt Pocock / Total TypeScript**: In his [TSConfig Cheat Sheet](https://www.totaltypescript.com/tsconfig-cheat-sheet) and [dedicated article on erasableSyntaxOnly](https://www.totaltypescript.com/erasable-syntax-only), Pocock describes it as "a really good option" especially for Node.js projects.

4. **Rob Palmer (Bloomberg / TC39)**: As a TC39 delegate and co-champion of the Type Annotations proposal, Palmer actively advocates for `erasableSyntaxOnly` and submitted the Node.js documentation PR himself.

5. **Community Projects**: Projects like Angular have started [replacing enums with constant objects](https://github.com/angular/angular/issues/64767) to align with this direction. The Kubernetes JavaScript client has [discussed enabling it](https://github.com/kubernetes-client/javascript/issues/2297).

6. **Deno**: There is an [open issue](https://github.com/denoland/deno/issues/28936) to support the `erasableSyntaxOnly` TSConfig option in Deno as well.

### Ecosystem Tooling

- **[eslint-plugin-erasable-syntax-only](https://github.com/JoshuaKGoldberg/eslint-plugin-erasable-syntax-only)**: Created by Josh Goldberg, this ESLint plugin provides granular, per-rule enforcement of erasable syntax. This is useful for gradual migrations since the TSConfig flag is all-or-nothing, while ESLint rules can be set to "warn" or applied to specific files.

## Pros and Cons

### Pros

| Benefit | Details |
|---------|---------|
| **Node.js compatibility** | Code runs directly in Node.js v23.6+ without any build step or additional flags |
| **Future-proofing** | Aligns with the TC39 Type Annotations proposal direction; code written this way will work natively in JS engines if the proposal advances |
| **Simpler toolchain** | No need for `tsc`, `tsx`, `ts-node`, or any transpiler for execution -- just `node file.ts` |
| **Tool agnostic** | Code works with SWC, esbuild, Bun, Deno, and any tool that strips types |
| **Clearer mental model** | TypeScript becomes "JavaScript + erasable type annotations" -- no hidden code generation |
| **Better library APIs** | Avoids enums in public APIs, which cause compatibility issues across different `isolatedModules` settings and bundler configurations |
| **Faster builds** | Type stripping is dramatically faster than full transpilation |

### Cons

| Drawback | Details |
|----------|---------|
| **Loses `enum` syntax** | Must use `as const` objects instead, which are slightly more verbose |
| **Loses parameter properties** | Must write explicit property declarations and constructor assignments (more boilerplate) |
| **Loses namespace code organization** | Must use ES modules instead (generally considered better anyway) |
| **Migration effort** | Existing projects may have many errors to fix (one project reported 49 errors upon enabling) |
| **No downlevel emit** | Cannot transpile modern syntax to older ES targets; you must target the ES version your runtime supports |
| **`const enum` loss** | Even though `const enum` has no runtime cost, it is still blocked |
| **Path aliases** | `paths` in tsconfig won't work for runtime resolution; must use `package.json` `imports` field instead |

## Relationship to the TC39 Type Annotations Proposal

### Proposal Overview

The [TC39 Type Annotations proposal](https://github.com/tc39/proposal-type-annotations) (formerly "Types as Comments") proposes that JavaScript engines treat type annotation syntax as comments -- ignored at runtime with no semantic effect. It is currently at **Stage 1** (as of March 2026).

**Champions**: Rob Palmer (Bloomberg), Romulo Cintra (Igalia), and others from the TypeScript team.

### What Would Be In Scope

The proposal covers syntax that is purely erasable:
- Type annotations on variables, parameters, return types (`: Type`)
- Interface and type alias declarations
- Generics (`<T>`)
- Optional parameter markers (`?`)
- Type assertion operators (`as`, `!`)
- `import type` / `export type`

### What Is Explicitly Out of Scope

- **Enums** (have runtime behavior)
- **Namespaces** (have runtime behavior)
- **Parameter properties** (have runtime behavior)
- **Decorators** (already standardized separately in TC39)

### The Connection

`erasableSyntaxOnly` is essentially **TypeScript's enforcement mechanism for the subset of syntax that the TC39 proposal covers**. If the TC39 proposal advances to later stages and eventually becomes part of the JavaScript specification, code written with `erasableSyntaxOnly` enabled would run natively in browsers and runtimes without any tooling at all.

This is why some describe the flag as "betting on the future of JavaScript." The TypeScript team is signaling that the long-term direction is TypeScript-as-erasable-types, not TypeScript-as-a-language-with-its-own-runtime-features.

Sources:
- [TC39 Type Annotations Proposal](https://github.com/tc39/proposal-type-annotations)
- [TypeScript Blog -- A Proposal For Type Syntax in JavaScript](https://devblogs.microsoft.com/typescript/a-proposal-for-type-syntax-in-javascript/)
- [TC39 Proposal Spec Text](https://tc39.es/proposal-type-annotations/)

## Community Articles and Resources

### Total TypeScript (Matt Pocock)
- [TypeScript 5.8 Ships --erasableSyntaxOnly To Disable Enums](https://www.totaltypescript.com/erasable-syntax-only) -- Dedicated article explaining the flag, what it blocks, why it exists, and the connection to Node.js
- [Why I Don't Like Enums](https://www.totaltypescript.com/why-i-dont-like-typescript-enums) -- Earlier article making the case against enums
- [The TSConfig Cheat Sheet](https://www.totaltypescript.com/tsconfig-cheat-sheet) -- Includes `erasableSyntaxOnly` in recommended settings

### Other Notable Articles
- [oida.dev -- TypeScript's erasableSyntaxOnly Flag](https://oida.dev/erasable-syntax-only/) -- Thorough technical explanation
- [egghead.io -- Use the erasableSyntaxOnly TypeScript compilation flag](https://egghead.io/use-the-erasable-syntax-only-type-script-compilation-flag~8m293) -- Video tutorial
- [typescript.tv -- Why TypeScript Enums Are Dead](https://typescript.tv/best-practices/why-typescript-enums-are-dead/) -- Explains the broader shift away from enums
- [AngularSpace -- Breaking the Enum Habit](https://www.angularspace.com/breaking-the-enum-habit-why-typescript-developers-need-a-new-approach/) -- Angular-specific perspective
- [2ality -- Node's new built-in support for TypeScript](https://2ality.com/2025/01/nodejs-strip-type.html) -- Dr. Axel Rauschmayer's thorough coverage
- [Effective TypeScript -- A Small Year for tsc, a Giant Year for TypeScript](https://effectivetypescript.com/2025/12/19/ts-2025/) -- Dan Vanderkam's year-in-review covering the erasable syntax shift

### ESLint Plugin
- [eslint-plugin-erasable-syntax-only](https://github.com/JoshuaKGoldberg/eslint-plugin-erasable-syntax-only) by Josh Goldberg -- Provides granular per-rule enforcement for gradual migration

## Should Library Authors Enable It? What About App Developers?

### Library Authors: Strong Yes

Library authors benefit the most from `erasableSyntaxOnly`:

1. **Wider compatibility**: Libraries that avoid enums in their public API work seamlessly for consumers regardless of their `isolatedModules`, bundler, or runtime configuration. `const enum` in `.d.ts` files is a known pain point for consumers.

2. **Future-proofing the API surface**: If a library exposes enums in its public API and later needs to remove them, it is a breaking change. String literal unions and `as const` objects are more stable API choices.

3. **Toolchain independence**: Libraries built with erasable-only syntax can be consumed and processed by any tool (SWC, esbuild, tsc, Bun, Deno, Node.js native).

4. **Aligns with ecosystem direction**: Major frameworks (Angular, etc.) are already moving away from enums.

### App Developers: Recommended, Especially for New Projects

For application developers:

- **New projects**: Enable `erasableSyntaxOnly` from the start. The alternatives to enums and parameter properties are well-established. You gain the ability to run TypeScript directly in Node.js without any build step.

- **Existing projects**: Consider a gradual migration using `eslint-plugin-erasable-syntax-only` with rules set to "warn" before enabling the TSConfig flag. The migration cost depends on how heavily the project uses enums, namespaces, and parameter properties.

- **Projects using bundlers**: Even if you use Vite, webpack, or another bundler (and thus do not rely on Node's native type stripping), enabling the flag still future-proofs your code and enforces a cleaner separation between types and runtime code.

### When NOT to Enable It

- Legacy projects heavily invested in enums and namespaces where migration cost is prohibitive
- Projects that intentionally use `--experimental-transform-types` in Node.js
- Projects targeting very old runtimes where downlevel transpilation of modern JS features is needed (though this is about `target`, not about `erasableSyntaxOnly` per se)

## Key Takeaways

- `erasableSyntaxOnly` was introduced in **TypeScript 5.8** (February 2025) and blocks enums (all forms including `const enum`), namespaces with runtime code, parameter properties, `import =` / `export =` aliases, and angle-bracket type assertions
- It was designed in **collaboration with the Node.js core team** to align TypeScript with Node's native type stripping (stable since v22.18.0 / v24.3.0)
- `const enum` **is blocked**, despite being inlined at compile time, to match SWC and Node.js behavior
- The flag aligns with the **TC39 Type Annotations proposal** (Stage 1), which envisions types as erasable comments in JavaScript
- It is **recommended by Node.js official docs**, `@tsconfig/bases`, Matt Pocock (Total TypeScript), Rob Palmer (TC39), and increasingly by the broader community
- The `as const` object pattern is the recommended replacement for enums
- **Library authors** should strongly consider enabling it; **new app projects** should enable it by default
- Pair it with `verbatimModuleSyntax` for the full "erasable TypeScript" experience
- Use `eslint-plugin-erasable-syntax-only` for gradual migration of existing codebases

## Open Questions and Limitations

- **TC39 proposal timeline**: The Type Annotations proposal remains at Stage 1. There is no guarantee it will advance, though momentum is strong. If it stalls, `erasableSyntaxOnly` still has value for Node.js compatibility.
- **`declare enum` edge cases**: The exact handling of `declare enum` in all contexts with `erasableSyntaxOnly` has some nuance; there have been [GitHub issues](https://github.com/microsoft/TypeScript/issues/61326) about `export import` of types within namespaces.
- **Bun and Deno**: While both runtimes handle TypeScript natively (and more completely than Node's type stripping), Deno has an [open issue](https://github.com/denoland/deno/issues/28936) for supporting the `erasableSyntaxOnly` flag. Bun handles all TypeScript syntax including enums, so the flag is less critical there but still useful for portability.
- **Performance of `as const` vs `enum`**: While `as const` objects are generally equivalent in performance to regular enums, they lack the reverse mapping feature of numeric enums (`Direction[0] === "Up"`). This is rarely needed in practice but is a capability gap.

## Sources

1. [TypeScript 5.8 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html) -- Official TypeScript documentation
2. [Announcing TypeScript 5.8 -- Microsoft DevBlogs](https://devblogs.microsoft.com/typescript/announcing-typescript-5-8/) -- Official announcement blog post
3. [Total TypeScript -- erasableSyntaxOnly](https://www.totaltypescript.com/erasable-syntax-only) -- Matt Pocock's detailed article
4. [Total TypeScript -- TSConfig Cheat Sheet](https://www.totaltypescript.com/tsconfig-cheat-sheet) -- Recommended TSConfig settings
5. [oida.dev -- TypeScript's erasableSyntaxOnly Flag](https://oida.dev/erasable-syntax-only/) -- Community technical article
6. [Node.js TypeScript Documentation](https://nodejs.org/api/typescript.html) -- Official Node.js docs on TypeScript support
7. [Node.js PR #57271](https://github.com/nodejs/node/pull/57271) -- Rob Palmer's PR to recommend erasableSyntaxOnly in Node docs
8. [TypeScript Issue #61490](https://github.com/microsoft/TypeScript/issues/61490) -- Discussion on const enum behavior
9. [TC39 Type Annotations Proposal](https://github.com/tc39/proposal-type-annotations) -- The ECMAScript proposal for erasable type syntax
10. [TypeScript Blog -- A Proposal For Type Syntax in JavaScript](https://devblogs.microsoft.com/typescript/a-proposal-for-type-syntax-in-javascript/) -- TypeScript team's perspective on the TC39 proposal
11. [2ality -- Node's new built-in support for TypeScript](https://2ality.com/2025/01/nodejs-strip-type.html) -- Dr. Axel Rauschmayer's technical overview
12. [eslint-plugin-erasable-syntax-only](https://github.com/JoshuaKGoldberg/eslint-plugin-erasable-syntax-only) -- Josh Goldberg's ESLint plugin for gradual adoption
13. [Rob Palmer on X](https://x.com/robpalmer2/status/1884715044585259127) -- Original announcement of the flag
14. [Kubernetes Client JS -- Issue #2297](https://github.com/kubernetes-client/javascript/issues/2297) -- Real-world project discussion on enabling the flag
15. [Angular -- Issue #64767](https://github.com/angular/angular/issues/64767) -- Angular replacing HttpStatusCode enum with constant object
16. [Effective TypeScript -- 2025 Year in Review](https://effectivetypescript.com/2025/12/19/ts-2025/) -- Dan Vanderkam's perspective on the broader shift
17. [typescript.tv -- Why TypeScript Enums Are Dead](https://typescript.tv/best-practices/why-typescript-enums-are-dead/) -- Community best practices article
18. [TypeScript TSConfig Reference -- erasableSyntaxOnly](https://www.typescriptlang.org/tsconfig/erasableSyntaxOnly.html) -- Official TSConfig documentation

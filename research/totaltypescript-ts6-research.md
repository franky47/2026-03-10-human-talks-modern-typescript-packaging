# Total TypeScript (Matt Pocock) Coverage of TypeScript 6.0 and Recent TS Changes

## Executive Summary

As of March 2026, **Total TypeScript has not published a dedicated article about TypeScript 6.0** on its blog. Despite Matt Pocock being one of the most prolific TypeScript educators, the site does not yet have a "TypeScript 6.0" breakdown post comparable to his coverage of previous versions (5.0, 5.1, 5.3, 5.5, 5.8). However, Total TypeScript has published several highly relevant articles about the individual features and compiler options that TypeScript 6.0 builds upon, particularly around `erasableSyntaxOnly`, `verbatimModuleSyntax`, Node.js TypeScript support, and the Go rewrite announcement.

The `@total-typescript/tsconfig` npm package appears to be at version 1.0.4 and has not been updated specifically for TypeScript 6.0 yet. Given that TS 6.0 beta was only released on February 11, 2026, updates may be forthcoming.

TypeScript 6.0 beta itself is a **transition release** -- the last JavaScript-based compiler -- designed to clean up defaults and deprecations before TypeScript 7.0 (the Go rewrite). Its key changes (strict by default, ESM-first defaults, `esModuleInterop` always on) align with what Total TypeScript has been recommending for years.

## Key Total TypeScript Articles Relevant to TS 6.0 and Library Packaging

### 1. TypeScript 5.8 Ships --erasableSyntaxOnly To Disable Enums
**URL:** https://www.totaltypescript.com/erasable-syntax-only

This is one of the most directly relevant articles for the TS 6.0 era. Key points:
- `erasableSyntaxOnly` marks enums, namespaces, and class parameter properties as errors
- "Erasable" syntax means it can be deleted without affecting runtime behavior (pure type annotations)
- **Critical connection to Node.js**: Node's built-in TypeScript support only supports erasable syntax by default
- Recommendation: turning on `erasableSyntaxOnly` is a good choice, especially for code targeting Node's native TS support
- This flag effectively nudges the ecosystem toward the same constraints that TypeScript 6.0/7.0 will further codify

### 2. Node.js Now Supports TypeScript By Default
**URL:** https://www.totaltypescript.com/typescript-is-coming-to-node-23

Key points relevant to packaging:
- Node 23 can run TypeScript files without extra configuration (via `--experimental-strip-types` unflagged)
- Node's TS support only handles erasable syntax by default
- For enums/namespaces, you need `--experimental-transform-types`
- This shapes how library authors should think about their output and source compatibility

### 3. TypeScript Announces Go Rewrite, Achieves 10x Speedup
**URL:** https://www.totaltypescript.com/typescript-announces-go-rewrite

Published around March 11, 2025. Matt Pocock called it "the biggest TS announcement I can remember." Key context:
- TypeScript compiler rewritten in Go (codename "tsgo")
- 10x speedup in some repos, up to 15x in others
- This rewrite becomes TypeScript 7.0
- TypeScript 6.0 is the bridge release preparing the ecosystem

### 4. Relative Import Paths Need Explicit File Extensions
**URL:** https://www.totaltypescript.com/relative-import-paths-need-explicit-file-extensions-in-ecmascript-imports

Covers the common pain point with `moduleResolution: NodeNext` requiring `.js` extensions. Two solutions offered:
- Use an external compiler (like esbuild) that handles resolution
- Use `moduleResolution: Bundler` instead of `NodeNext`

Related: `rewriteRelativeImportExtensions` (introduced in TS 5.7) is mentioned in the context of the tsconfig cheat sheet but does not have a dedicated article on Total TypeScript.

### 5. The TSConfig Cheat Sheet
**URL:** https://www.totaltypescript.com/tsconfig-cheat-sheet

This is a living document that Matt Pocock updates. It references:
- `verbatimModuleSyntax`
- `isolatedDeclarations`
- `rewriteRelativeImportExtensions`
- `erasableSyntaxOnly`
- Recommended base configurations for apps vs. libraries

The cheat sheet is also the basis for the `@total-typescript/tsconfig` npm package.

### 6. How To Create An NPM Package
**URL:** https://www.totaltypescript.com/how-to-create-an-npm-package

A comprehensive guide covering the full workflow from empty directory to published npm package. Recommended tsconfig settings include:
- `esModuleInterop: true`
- `skipLibCheck: true`
- `target: "es2022"`
- `strict: true`
- `declaration: true`
- `declarationMap: true` (for monorepos)

Uses `@changesets/cli` for versioning and publishing.

### 7. Why I Don't Like Enums
**URL:** https://www.totaltypescript.com/why-i-dont-like-typescript-enums

Relevant context: Matt Pocock has long advocated against enums, which aligns with `erasableSyntaxOnly` and the direction of TS 6.0/Node's native TS support.

## The @total-typescript/tsconfig Package

**npm:** https://www.npmjs.com/package/@total-typescript/tsconfig
**GitHub:** https://github.com/total-typescript/tsconfig
**Releases:** https://github.com/total-typescript/tsconfig/releases

### Current State
- Latest known version: **1.0.4** (published approximately early-to-mid 2025)
- No confirmed update for TypeScript 6.0 as of March 2026

### Configuration Options
The package provides preset configs organized by:
- **Runtime environment**: `bundler/` (for projects using an external bundler)
- **DOM availability**: `dom/` vs `no-dom/`
- **Project type**: `app` vs `library` vs `library-monorepo`

Example usage:
```json
{
  "extends": "@total-typescript/tsconfig/bundler/dom/library"
}
```

### Relevance to TS 6.0
Many of the defaults that TS 6.0 now enforces (strict mode, ESM-first, esModuleInterop always on) were already the recommended settings in `@total-typescript/tsconfig`. This means:
- Projects already using this package may have a smoother migration to TS 6.0
- The package may need fewer changes than expected, since it was already opinionated in the "right" direction
- However, it will likely need an update to account for deprecated/removed options

## TypeScript 6.0 Beta Context (for reference)

Released February 11, 2026. Key changes that library authors need to know:

| Change | TS 5.x Default | TS 6.0 Default |
|--------|----------------|----------------|
| `strict` | `false` | **`true`** |
| `module` | `commonjs` | **`esnext`** |
| `target` | `es5` | **current-year ES** |
| `esModuleInterop` | `false` | **always `true`** (cannot be `false`) |
| `noUncheckedSideEffectImports` | `false` | **`true`** |
| `libReplacement` | `true` | **`false`** |

Other breaking changes:
- `allowSyntheticDefaultImports` cannot be set to `false`
- All code assumed to be in JS strict mode
- `outFile` deprecated
- `rootDir` inference changes (diagnostic 5011 guides you)
- Can use `"ignoreDeprecations": "6.0"` as an escape hatch

**Migration guide:** https://gist.github.com/privatenumber/3d2e80da28f84ee30b77d53e1693378f

## Key Takeaways for Your Talk

- **Total TypeScript has been ahead of the curve**: Most TS 6.0 defaults (strict, ESM, esModuleInterop) were already recommended by Matt Pocock's cheat sheet and tsconfig package
- **`erasableSyntaxOnly` is the bridge concept**: It connects Node's native TS support, library packaging best practices, and the direction of TS 6.0/7.0
- **No dedicated TS 6.0 article yet**: As of March 10, 2026, Matt Pocock has not published a TS 6.0 breakdown on totaltypescript.com (the beta is only ~1 month old)
- **The tsconfig cheat sheet is the canonical reference**: https://www.totaltypescript.com/tsconfig-cheat-sheet -- check it again closer to your talk date as it may be updated
- **`@total-typescript/tsconfig` may get an update**: Watch the GitHub releases page at https://github.com/total-typescript/tsconfig/releases
- **`verbatimModuleSyntax`** has been recommended since TS 5.0 and remains important for library packaging
- **`rewriteRelativeImportExtensions`** (TS 5.7+) is referenced in the cheat sheet but lacks a dedicated deep-dive article

## Open Questions & Limitations

- The tsconfig cheat sheet page could not be fetched directly (WebFetch was unavailable); its current exact content is inferred from search snippets
- It is unclear whether `@total-typescript/tsconfig` has had any releases beyond 1.0.4 -- the npm/GitHub pages could not be directly crawled
- Matt Pocock may have posted TS 6.0 reactions on X/Twitter that are not indexed by web search
- A Total TypeScript blog post about TS 6.0 may be published any day given the beta is recent

## Sources

1. [TypeScript 5.8 Ships --erasableSyntaxOnly To Disable Enums](https://www.totaltypescript.com/erasable-syntax-only) - Total TypeScript
2. [Node.js Now Supports TypeScript By Default](https://www.totaltypescript.com/typescript-is-coming-to-node-23) - Total TypeScript
3. [TypeScript Announces Go Rewrite, Achieves 10x Speedup](https://www.totaltypescript.com/typescript-announces-go-rewrite) - Total TypeScript
4. [Relative Import Paths Need Explicit File Extensions](https://www.totaltypescript.com/relative-import-paths-need-explicit-file-extensions-in-ecmascript-imports) - Total TypeScript
5. [The TSConfig Cheat Sheet](https://www.totaltypescript.com/tsconfig-cheat-sheet) - Total TypeScript
6. [How To Create An NPM Package](https://www.totaltypescript.com/how-to-create-an-npm-package) - Total TypeScript
7. [Why I Don't Like Enums](https://www.totaltypescript.com/why-i-dont-like-typescript-enums) - Total TypeScript
8. [@total-typescript/tsconfig on npm](https://www.npmjs.com/package/@total-typescript/tsconfig)
9. [total-typescript/tsconfig on GitHub](https://github.com/total-typescript/tsconfig)
10. [Announcing TypeScript 6.0 Beta](https://devblogs.microsoft.com/typescript/announcing-typescript-6-0-beta/) - Microsoft DevBlogs
11. [TypeScript 5.x to 6.0 Migration Guide](https://gist.github.com/privatenumber/3d2e80da28f84ee30b77d53e1693378f) - privatenumber (GitHub Gist)
12. [TypeScript 6 Beta Released: Developers Invited to Upgrade](https://www.infoq.com/news/2026/02/typescript-6-released-beta/) - InfoQ
13. [Matt Pocock on X - Go rewrite announcement](https://x.com/mattpocockuk/status/1899469836897362136)
14. [TypeScript Articles by Matt Pocock](https://www.totaltypescript.com/articles) - Total TypeScript
15. [What's Coming In TypeScript 5.3](https://www.totaltypescript.com/typescript-5-3) - Total TypeScript (covers isolatedDeclarations)
16. [Configuring TypeScript](https://www.totaltypescript.com/books/total-typescript-essentials/configuring-typescript) - Total TypeScript Essentials book

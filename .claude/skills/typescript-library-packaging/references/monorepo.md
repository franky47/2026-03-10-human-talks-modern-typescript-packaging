# Monorepo Setup

## The "Internal Packages" pattern (recommended)

Internal packages (not published to npm) skip the build step entirely. The app's bundler handles transpilation.

```
packages/utils/
  src/index.ts         <- raw TypeScript, no build step
  package.json         <- exports point to src/*.ts
```

```json
{
  "name": "@myorg/utils",
  "private": true,
  "type": "module",
  "exports": {
    ".": "./src/index.ts",
    "./math": "./src/math.ts"
  }
}
```

```json
// apps/web/package.json
{
  "dependencies": {
    "@myorg/utils": "workspace:*"
  }
}
```

**Why this works**: Next.js, Vite, Turbopack, esbuild all handle `.ts` imports natively. No separate build step, changes reflected instantly.

## Workspace protocols

| Protocol | Syntax | Behavior |
|---|---|---|
| `workspace:*` | `"@myorg/utils": "workspace:*"` | Local symlink; on publish, replaced with actual version. **pnpm/yarn only.** |
| `workspace:^` | `"@myorg/utils": "workspace:^"` | On publish -> `^1.0.0` |
| `file:` | `"@myorg/utils": "file:../utils"` | Creates a copy. All package managers. |
| `link:` | `"@myorg/utils": "link:../utils"` | Creates a symlink. |

**Recommendation**: `workspace:*` with pnpm.

## TypeScript project references (for large monorepos)

```jsonc
// packages/utils/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true
  }
}

// apps/web/tsconfig.json
{
  "references": [{ "path": "../../packages/utils" }]
}
```

Helps the TS language server with performance in large monorepos. Not strictly required if editor performance is acceptable.

## Recommended stack

- **pnpm** for package management (disk-efficient, strict, `workspace:` protocol)
- **Turborepo** for build orchestration + caching (if you have buildable packages)
- **Internal packages pattern** for everything that isn't published

## Example structure

```
my-monorepo/
  pnpm-workspace.yaml
  turbo.json (optional)
  tsconfig.base.json

  packages/
    utils/              <- internal (no build)
      src/index.ts
      package.json      <- exports: ./src/index.ts
    shared-lib/         <- published (has build)
      src/index.ts
      dist/
      package.json      <- exports: ./dist/index.js

  apps/
    web/
      package.json      <- depends on @myorg/utils, @myorg/shared-lib
```

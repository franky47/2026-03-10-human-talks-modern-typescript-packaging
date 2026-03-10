# Build Tools Comparison

## Comparison matrix

| Tool | Engine | Speed | Config | .d.ts | Tree-shaking | Best for |
|---|---|---|---|---|---|---|
| **tsdown** | Rolldown (Rust) | Very fast | Simple | Yes (via tsc) | Excellent | New libraries (forward-looking) |
| **tsup** | esbuild (Go) | Very fast | Simple | Yes (via tsc) | Good | Proven choice, large community |
| **unbuild** | Rollup + mkdist | Moderate | Zero-config | Yes | Excellent | Multi-export libs, stub mode DX |
| **tsc/tsgo** | Native TS | Slow/Fast | Minimal | Native | N/A | No-bundle approach, max correctness |
| **pkgroll** | Rollup + esbuild | Moderate | Zero-config | Yes | Excellent | package.json-as-config approach |
| **esbuild** | Go | Fastest | Manual | No (need tsc) | Good | Custom pipelines, embedded engine |
| **swc** | Rust | Very fast | Manual | No (need tsc) | N/A | Embedded engine (Next.js, etc.) |

## Decision framework

- **New library, simple** -> `tsdown` (or `tsup` if tsdown isn't stable enough)
- **Multi-export library** -> `unbuild` (file-to-file via mkdist, zero-config, stub mode)
- **No bundling needed** -> `tsc` alone (consumers will bundle)
- **Already using Vite** -> Vite library mode
- **Zero config** -> `pkgroll` or `unbuild`

## Bundle vs no-bundle

| Factor | Bundle (tsup, tsdown) | No-Bundle (tsc, file-to-file) |
|---|---|---|
| Tree-shaking | Consumer relies on their bundler | Natural — consumers import only what they need |
| Internal structure | Hidden from consumers | May leak internal paths |
| Package size | Smaller (single file) | Larger (many files) |
| Debugging | Harder without source maps | Easier — 1:1 file mapping |
| Circular deps | Bundler resolves them | Can cause runtime issues |

## The big stories

### tsdown (v0.21.1)

Spiritual successor to tsup, built on **Rolldown** (Rust-based Rollup replacement by VoidZero/Evan You, currently v1 RC). Better tree-shaking, scope hoisting, code splitting than esbuild. Rollup plugin compatible. Still pre-1.0 but rapidly maturing.

### tsgo (beta)

Native **Go port** of the TypeScript compiler by Microsoft (led by Anders Hejlsberg). ~10x performance improvement. TS 6.0 is the last TS-written major; TS 7.0 will be the Go port. Biggest impact: `.d.ts` generation becomes near-instant.

### Isolated declarations (TS 5.5)

Enables non-tsc tools to generate `.d.ts` without full type-checking. Requires explicit return type annotations on exported functions. Significantly speeds up tsdown/tsup for `.d.ts` bundling. Enable with `isolatedDeclarations: true` in tsconfig.

## Tools that are NOT build tools

- **Biome** — linting + formatting only
- **SWC** — embedded engine, rarely used standalone for library builds
- **esbuild** — increasingly embedded (powers tsup, Vite dev), not standalone library tool

# Registries: npm, JSR, vlt

## npm (default)

- Ship compiled JS + `.d.ts`
- Use `"files"` allowlist (not `.npmignore`)
- Use `npm pack --dry-run` to preview published files
- Use `--provenance` flag for supply chain security (from CI)

## JSR (jsr.io)

Created by the Deno team. **Publish raw `.ts` directly** — no build step needed.

| Feature | JSR | npm |
|---|---|---|
| TypeScript | First-class (raw `.ts`) | Compiled JS + `.d.ts` |
| Module format | ESM only | ESM and/or CJS |
| Auto-generated docs | Yes (from JSDoc/TSDoc) | No |
| Type checking on publish | Yes | No |
| Quality scoring | Yes | No |
| npm compatibility | Yes (via compatibility layer) | Native |
| Scoping | Required (`@scope/name`) | Optional |

### Publishing to JSR

```json
// jsr.json
{
  "name": "@myorg/my-package",
  "version": "1.0.0",
  "exports": "./mod.ts"
}
```

```bash
npx jsr publish   # or: deno publish
```

Consumers install via: `npx jsr add @myorg/my-package`

## vlt (vlt.sh)

New package manager + registry by former npm engineers. Still early stage.

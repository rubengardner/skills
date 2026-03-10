# Specialist I: tsconfig

## Scope
`tsconfig.json` flags, strict mode, path aliases, project references, `lib`, performance settings.

---

## 1. Required Strict Mode Settings

Always start with `"strict": true`. This enables:

| Flag | Effect |
|------|--------|
| `noImplicitAny` | Error on inferred `any` |
| `strictNullChecks` | `null`/`undefined` are not assignable to other types |
| `strictFunctionTypes` | Contravariant function parameter checking |
| `strictBindCallApply` | Strict types on `.bind()`, `.call()`, `.apply()` |
| `strictPropertyInitialization` | Class properties must be initialized |
| `noImplicitThis` | Error on `this` with implicit `any` type |
| `useUnknownInCatchVariables` | `catch (e)` is `unknown` not `any` (TS 4.4+) |

---

## 2. Recommended Additional Flags

```json
{
  "compilerOptions": {
    "strict": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

| Flag | Effect |
|------|--------|
| `exactOptionalPropertyTypes` | `x?: T` means absent, not `T \| undefined` |
| `noUncheckedIndexedAccess` | Array/index access returns `T \| undefined` |
| `noImplicitOverride` | Overriding methods must have `override` keyword |
| `noPropertyAccessFromIndexSignature` | Must use bracket notation for index signature access |
| `noFallthroughCasesInSwitch` | Error on switch cases without break/return |

**`noUncheckedIndexedAccess` impact:**
```typescript
const items = ['a', 'b', 'c'];
const first = items[0]; // type: string | undefined (not string!)
// Must check before use:
if (first !== undefined) {
  console.log(first.toUpperCase());
}
```

---

## 3. Full Recommended `tsconfig.json` (App)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,

    "strict": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true,

    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src", "types"],
  "exclude": ["node_modules", "dist"]
}
```

---

## 4. `tsconfig.json` for Node.js (no DOM)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "dist",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,

    "strict": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

---

## 5. Path Aliases

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@components/*": ["./src/components/*"],
      "@types/*": ["./src/types/*"]
    }
  }
}
```

**Rules:**
- Path aliases must also be configured in the bundler (Vite, webpack, etc.) separately.
- Keep aliases minimal — deep nesting creates cognitive overhead.
- `@/` as the root alias is the community standard.

---

## 6. `isolatedModules`

Set `"isolatedModules": true` when using a transpiler (esbuild, swc, Babel) that processes files independently:

- Each file must be a module (have at least one `import`/`export`)
- `const enum` is not allowed (can't be inlined across files)
- `export type { T }` must be used for type-only exports (they're erased)

---

## 7. Performance Tips

- Use `"skipLibCheck": true` to skip checking `.d.ts` files in `node_modules`. Almost always safe and significantly speeds up compilation.
- Use `"incremental": true` to cache build state.
- Use project references (`references` + `composite`) for monorepos to enable per-package type checking.
- Use `"types": ["node"]` in `compilerOptions` to limit auto-included type packages to what you actually use.

---

## 8. Common Errors and Fixes

| Error | Fix |
|-------|-----|
| `'X' is possibly 'undefined'` | Check `noUncheckedIndexedAccess`; add a null check |
| `Object is possibly 'null'` | Enable `strictNullChecks`; add null check |
| `Property 'X' has no initializer` | Initialize in constructor, or use `!` assertion only if guaranteed externally (e.g., dependency injection) |
| `Type 'X' is not assignable to type 'never'` | Missing case in exhaustive switch; add the case |
| `Cannot find module '@/...'` | Configure `paths` in tsconfig AND in bundler |
| `Re-exporting a type when '--isolatedModules' is set requires using 'export type'` | Use `export type { T }` instead of `export { T }` |

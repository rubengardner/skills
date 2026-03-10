# Specialist H: Module & Declaration Patterns

## Scope
Module augmentation, declaration merging, ambient modules, `declare`, `namespace`, `/// <reference>`, `.d.ts` files.

---

## 1. Declaration Merging (Interfaces)

Interfaces with the same name in the same scope are merged:

```typescript
interface Window {
  myCustomProperty: string;
}

interface Window {
  anotherCustomProperty: number;
}

// Result:
// interface Window {
//   myCustomProperty: string;
//   anotherCustomProperty: number;
// }
```

**Rule:** Only use declaration merging to augment types you don't own (e.g., 3rd party libraries, global objects). Never use it to split your own interface definitions — write them together.

---

## 2. Module Augmentation

Extend types from external modules:

```typescript
// express.d.ts (in your project)
import 'express';

declare module 'express' {
  interface Request {
    user?: AuthUser;
    requestId: string;
  }
}

// Now in route handlers:
app.get('/me', (req, res) => {
  console.log(req.user?.id); // typed ✓
});
```

**Rules:**
- The augmentation file must have at least one import or export to be treated as a module (not a script).
- Place augmentations in a dedicated `types/` directory (e.g., `src/types/express.d.ts`).
- Include the `types/` directory in `tsconfig.json` under `include` or `typeRoots`.

---

## 3. Ambient Module Declarations

Declare types for modules that have no TypeScript types:

```typescript
// src/types/untyped-library.d.ts
declare module 'untyped-legacy-library' {
  export function doThing(input: string): number;
  export const version: string;
}

// Wildcard — for asset imports (e.g., with webpack/vite)
declare module '*.svg' {
  const content: string;
  export default content;
}

declare module '*.png' {
  const src: string;
  export default src;
}
```

---

## 4. `declare` — Ambient Declarations

`declare` tells TypeScript a value exists at runtime but is not defined in the current file:

```typescript
// Global injected by build tool (e.g., webpack DefinePlugin)
declare const __DEV__: boolean;
declare const __VERSION__: string;

// Global variable from a script tag
declare const analytics: {
  track(event: string, properties?: Record<string, unknown>): void;
};
```

---

## 5. `namespace` (Legacy — Avoid)

Namespaces are a pre-module pattern. Only use them for:
1. Augmenting 3rd party global namespaces (e.g., `jQuery`, `google.maps`)
2. Organizing declaration files for libraries that expose a global variable

```typescript
// Augmenting a global namespace (legacy lib)
declare namespace MyLib {
  function makeGreeting(s: string): string;
  let numberOfGreetings: number;
}
```

**Rule:** Do NOT use `namespace` in modern module-based code. Use ES modules (`import`/`export`) instead.

---

## 6. `/// <reference>` Directives

Reference directives include type files explicitly:

```typescript
/// <reference types="node" />       // include @types/node
/// <reference path="./my-types.d.ts" /> // include a local .d.ts
/// <reference lib="dom" />           // include DOM lib types
```

**Rules:**
- Prefer `tsconfig.json` `types` and `lib` fields over reference directives.
- Use path references only when you need types from a `.d.ts` that's outside the project's include paths.

---

## 7. The `satisfies` + `as const` Pattern for Config Objects

```typescript
// Validate config shape at definition site without widening
const routes = {
  home: '/',
  about: '/about',
  users: '/users/:id',
} as const satisfies Record<string, string>;

type RouteName = keyof typeof routes; // 'home' | 'about' | 'users'
type RoutePath = typeof routes[RouteName]; // '/' | '/about' | '/users/:id'
```

---

## 8. Re-exporting and Barrel Files

```typescript
// src/types/index.ts
export type { User, UserRole } from './user';
export type { Order, OrderStatus } from './order';
export type { ApiResponse, ApiError } from './api';

// Consumer
import type { User, Order } from '@/types';
```

**Rules:**
- Use `export type { ... }` (not `export { ... }`) for type-only re-exports — this ensures the import is erased at compile time.
- Avoid deep barrel files that re-export implementation — they slow down compilation.
- Barrel files for types only are fine and encouraged.

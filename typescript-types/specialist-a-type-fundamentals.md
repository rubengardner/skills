# Specialist A: Type Fundamentals

## Scope
Primitive types, literal types, type inference, widening vs narrowing, `as const`, `satisfies`, `typeof`, type aliases vs direct annotation.

---

## 1. Primitive Types

| TypeScript | JavaScript runtime |
|------------|--------------------|
| `string` | string |
| `number` | number |
| `boolean` | boolean |
| `bigint` | bigint |
| `symbol` | symbol |
| `null` | null |
| `undefined` | undefined |
| `void` | undefined (return only) |
| `never` | unreachable |
| `unknown` | anything — must narrow before use |
| `any` | opt-out of type checking — **do not use** |

**Rules:**
- `void` is for return types of functions that don't return a meaningful value. Never use it as a variable type.
- `null` and `undefined` are separate types under `strictNullChecks`. Never conflate them.
- `never` is the bottom type. A function that throws unconditionally or loops forever returns `never`.
- `unknown` is the top type for unsafe input. Always narrow before use.

---

## 2. Literal Types

TypeScript can represent exact values as types:

```typescript
type Direction = 'north' | 'south' | 'east' | 'west';
type StatusCode = 200 | 400 | 404 | 500;
type Enabled = true; // only true, not any boolean
```

**Widening:** By default, string/number literals in `let` declarations widen to their primitive type:
```typescript
let x = 'hello'; // type: string (widened)
const y = 'hello'; // type: 'hello' (literal — const doesn't reassign)
```

---

## 3. `as const` — Preserve Literals

Use `as const` to prevent widening and make all properties `readonly`:

```typescript
// Without as const
const config = { env: 'prod', port: 3000 };
// type: { env: string; port: number }

// With as const
const config = { env: 'prod', port: 3000 } as const;
// type: { readonly env: 'prod'; readonly port: 3000 }

// Arrays
const ROLES = ['admin', 'user', 'guest'] as const;
// type: readonly ['admin', 'user', 'guest']
type Role = typeof ROLES[number]; // 'admin' | 'user' | 'guest'
```

**Rule:** Use `as const` on any config object, lookup table, or tuple where literal types must be preserved.

---

## 4. `satisfies` — Validate Without Widening (TS 4.9+)

`satisfies` checks a value against a type **without** changing the inferred type:

```typescript
type Palette = Record<string, [number, number, number] | string>;

// With `as` — loses specific type information
const palette = {
  red: [255, 0, 0],
  blue: '#0000ff',
} as Palette;
palette.red.toUpperCase(); // allowed! type was widened to string | [number,number,number]

// With `satisfies` — validates AND preserves specific types
const palette = {
  red: [255, 0, 0],
  blue: '#0000ff',
} satisfies Palette;
palette.red.toUpperCase(); // Error! red is [number,number,number], not string ✓
palette.blue.toUpperCase(); // OK — blue is string ✓
```

**Rule:** Prefer `satisfies` over `as Type` for object literals. Use `as` only at validated runtime boundaries.

---

## 5. Type Inference

TypeScript infers types from:
- Variable declarations (`const x = 42` → `number`)
- Function return values (inferred from `return` statements)
- Destructuring
- Generic type arguments (from usage)

**Explicit annotation is required when:**
- The inferred type is wider than intended
- The function is part of a public API
- Inference would be `any` (e.g., JSON.parse)
- A generic cannot be inferred from arguments

```typescript
// Inferred — fine for private, internal use
const add = (a: number, b: number) => a + b;

// Explicit — required for public API
export function parseUser(raw: unknown): User { ... }
```

---

## 6. `typeof` in Type Position

Use `typeof` to derive a type from a value:

```typescript
const defaultConfig = { timeout: 5000, retries: 3 } as const;
type Config = typeof defaultConfig;
// { readonly timeout: 5000; readonly retries: 3 }

function createUser(name: string) {
  return { id: crypto.randomUUID(), name };
}
type User = ReturnType<typeof createUser>;
// { id: string; name: string }
```

---

## 7. Type Aliases

```typescript
// Simple alias
type UserId = string;

// Alias for complex shapes
type ApiResponse<T> = {
  data: T;
  status: number;
  message: string;
};

// Union alias
type StringOrNumber = string | number;
```

**Rules:**
- Type aliases are not opaque — `UserId` and `string` are fully interchangeable. Use branded types if you need opaque IDs.
- Name types with PascalCase.
- Avoid aliases that just rename a primitive with no added meaning.

---

## 8. Branded / Nominal Types

TypeScript is structurally typed. To simulate nominal typing (prevent mixing of IDs):

```typescript
type UserId = string & { readonly __brand: 'UserId' };
type OrderId = string & { readonly __brand: 'OrderId' };

function createUserId(id: string): UserId {
  return id as UserId; // only at creation boundary
}

function getUser(id: UserId): User { ... }

const orderId = createOrderId('abc');
getUser(orderId); // Error! OrderId is not UserId ✓
```

**Rule:** Use branded types for IDs and domain primitives where structural equivalence would cause bugs.

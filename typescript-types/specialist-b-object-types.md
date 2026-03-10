# Specialist B: Object Types

## Scope
`interface` vs `type`, object shape definitions, optional/readonly properties, index signatures, excess property checking, extending and implementing.

---

## 1. `interface` vs `type` — Decision Rule

| Use `interface` when... | Use `type` when... |
|------------------------|-------------------|
| Defining an object contract others can implement or extend | Defining unions, intersections, or tuples |
| Declaration merging is needed (e.g., augmenting 3rd party types) | Computing a shape from other types (mapped, conditional) |
| The shape represents a class contract | Aliasing a primitive or utility type result |
| Working in a public library API | Internal, one-off computed shapes |

**Rule:** Default to `type` for app code. Reserve `interface` for class contracts and library-facing types.

```typescript
// Use interface — a contract a class can implement
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  save(entity: T): Promise<void>;
  delete(id: string): Promise<void>;
}

// Use type — computed from other types
type UserDTO = Pick<User, 'id' | 'name' | 'email'>;
type ApiResult<T> = { data: T; status: number };
```

---

## 2. Optional Properties

```typescript
type User = {
  id: string;
  name: string;
  bio?: string;         // optional: string | undefined
  avatar?: string;
};
```

**Rules:**
- `prop?: T` and `prop: T | undefined` are NOT the same under `exactOptionalPropertyTypes`.
  - `prop?: T` — property may be absent entirely
  - `prop: T | undefined` — property must be present, but its value may be `undefined`
- Enable `"exactOptionalPropertyTypes": true` in tsconfig for precise semantics.

```typescript
// With exactOptionalPropertyTypes: true
type A = { x?: number };
const a: A = {};            // OK — x is absent
const b: A = { x: 1 };     // OK
const c: A = { x: undefined }; // Error! Must be absent or number, not undefined
```

---

## 3. `readonly` Properties

```typescript
type Config = {
  readonly apiUrl: string;
  readonly timeout: number;
};

// Arrays
type State = {
  readonly items: readonly string[];
};
```

**Rules:**
- `readonly` prevents reassignment, not mutation of nested objects.
- For deep immutability use `as const` or `Readonly<T>` (shallow) or `ReadonlyDeep` from type-fest.
- Prefer `readonly` on all config/state types.

```typescript
// Readonly<T> is shallow
type Config = Readonly<{ nested: { value: number } }>;
const c: Config = { nested: { value: 1 } };
c.nested = {}; // Error ✓
c.nested.value = 2; // ALLOWED — Readonly is shallow
```

---

## 4. Index Signatures

Use index signatures when keys are dynamic but values are homogeneous:

```typescript
type StringMap = {
  [key: string]: string;
};

// With known keys + index signature
type HttpHeaders = {
  'content-type': string;
  [key: string]: string; // all other string keys also must be string
};
```

**Rules:**
- Index signature value type must be a supertype of all named property types.
- Prefer `Record<K, V>` for simple key-value maps.
- Use `Map<K, V>` for runtime maps with non-string keys.

```typescript
// Prefer Record over index signature for simple cases
type StatusMessages = Record<number, string>;
type RouteHandlers = Record<string, RequestHandler>;
```

---

## 5. Excess Property Checking

TypeScript checks for extra properties only on **object literals** assigned directly to a typed variable:

```typescript
type Point = { x: number; y: number };

const p: Point = { x: 1, y: 2, z: 3 }; // Error! z is excess ✓

// Structural compatibility check (no excess property check)
const raw = { x: 1, y: 2, z: 3 };
const p: Point = raw; // OK — structural subtype
```

**Rule:** This is a feature, not a bug. Don't use intermediate variables to bypass excess property checks — fix the type.

---

## 6. Extending Interfaces and Types

```typescript
// Interface extension
interface Animal {
  name: string;
}
interface Dog extends Animal {
  breed: string;
}

// Type intersection (equivalent for extension)
type Animal = { name: string };
type Dog = Animal & { breed: string };

// Interface extending type (and vice versa) — works
interface Named { name: string }
type Identified = Named & { id: string };
interface User extends Identified { email: string }
```

**Rules:**
- Intersection (`&`) merges types structurally. If the same key has incompatible types, the result is `never`.
- Don't use deep inheritance chains — prefer composition via intersection or utility types.

---

## 7. Class Implements

```typescript
interface Serializable {
  serialize(): string;
  deserialize(raw: string): void;
}

class User implements Serializable {
  constructor(public name: string) {}

  serialize(): string {
    return JSON.stringify({ name: this.name });
  }

  deserialize(raw: string): void {
    const { name } = JSON.parse(raw);
    this.name = name;
  }
}
```

**Rules:**
- `implements` is a compile-time contract only. It does not affect the generated JS.
- A class can implement multiple interfaces: `class Foo implements A, B`.
- Prefer `interface` (over `type`) for contracts intended to be `implements`-ed.

# Specialist E: Utility Types

## Scope
All built-in TypeScript utility types: Partial, Required, Readonly, Pick, Omit, Record, Extract, Exclude, NonNullable, ReturnType, Parameters, ConstructorParameters, InstanceType, Awaited, ThisType.

---

## 1. Object Shape Utilities

### `Partial<T>` — All properties optional
```typescript
type User = { id: string; name: string; email: string };
type UserUpdate = Partial<User>;
// { id?: string; name?: string; email?: string }

function updateUser(id: string, patch: Partial<User>): Promise<User> { ... }
```

### `Required<T>` — All properties required
```typescript
type Config = { host?: string; port?: number; tls?: boolean };
type ResolvedConfig = Required<Config>;
// { host: string; port: number; tls: boolean }
```

### `Readonly<T>` — All properties readonly (shallow)
```typescript
type ImmutableUser = Readonly<User>;
// { readonly id: string; readonly name: string; readonly email: string }
```

### `Pick<T, K>` — Select a subset of keys
```typescript
type UserPreview = Pick<User, 'id' | 'name'>;
// { id: string; name: string }
```

### `Omit<T, K>` — Remove keys
```typescript
type UserWithoutEmail = Omit<User, 'email'>;
// { id: string; name: string }

// Common pattern: Omit generated fields for create input
type CreateUserInput = Omit<User, 'id' | 'createdAt' | 'updatedAt'>;
```

### `Record<K, V>` — Map from key type to value type
```typescript
type StatusMessage = Record<'success' | 'error' | 'loading', string>;
type UserMap = Record<string, User>;
type Cache<T> = Record<string, T>;
```

**Rule:** Prefer `Record<string, T>` over `{ [key: string]: T }` index signature syntax — it's more readable.

---

## 2. Union Manipulation Utilities

### `Extract<T, U>` — Keep members of T assignable to U
```typescript
type A = 'cat' | 'dog' | 'fish';
type Mammal = Extract<A, 'cat' | 'dog'>; // 'cat' | 'dog'

type Event = { type: 'click' } | { type: 'scroll' } | { type: 'focus' };
type ClickEvent = Extract<Event, { type: 'click' }>; // { type: 'click' }
```

### `Exclude<T, U>` — Remove members of T assignable to U
```typescript
type NonFish = Exclude<A, 'fish'>; // 'cat' | 'dog'
type NonNullString = Exclude<string | null | undefined, null | undefined>; // string
```

### `NonNullable<T>` — Remove `null` and `undefined`
```typescript
type T = NonNullable<string | null | undefined>; // string
```

---

## 3. Function Utilities

### `ReturnType<T>` — Extract the return type of a function
```typescript
function createUser(name: string) {
  return { id: crypto.randomUUID(), name, createdAt: new Date() };
}
type User = ReturnType<typeof createUser>;
// { id: string; name: string; createdAt: Date }
```

### `Parameters<T>` — Extract parameter types as a tuple
```typescript
function login(email: string, password: string): Promise<User> { ... }
type LoginArgs = Parameters<typeof login>;
// [email: string, password: string]

// Useful for wrapping functions
function withLogging<T extends (...args: any[]) => any>(
  fn: T
): (...args: Parameters<T>) => ReturnType<T> {
  return (...args) => {
    console.log('calling', fn.name);
    return fn(...args);
  };
}
```

### `ConstructorParameters<T>` — Constructor arg types
```typescript
class HttpClient {
  constructor(baseUrl: string, timeout: number) {}
}
type ClientArgs = ConstructorParameters<typeof HttpClient>;
// [baseUrl: string, timeout: number]
```

### `InstanceType<T>` — Instance type of a class constructor
```typescript
type Client = InstanceType<typeof HttpClient>;
// HttpClient
```

---

## 4. `Awaited<T>` — Deep Promise Unwrap (TS 4.5+)

```typescript
type T1 = Awaited<Promise<string>>;           // string
type T2 = Awaited<Promise<Promise<number>>>;  // number
type T3 = Awaited<ReturnType<typeof fetchUser>>; // User (if fetchUser returns Promise<User>)
```

**Rule:** Use `Awaited<ReturnType<typeof fn>>` to derive the resolved type of async functions without manually annotating it.

---

## 5. Combining Utility Types

```typescript
// Immutable partial for config merging
type ConfigOverride = Readonly<Partial<Config>>;

// Create input: no ID, no timestamps, all required
type CreateInput<T extends { id: string; createdAt: Date; updatedAt: Date }> =
  Required<Omit<T, 'id' | 'createdAt' | 'updatedAt'>>;

// Update input: ID required, everything else optional
type UpdateInput<T extends { id: string }> =
  Pick<T, 'id'> & Partial<Omit<T, 'id'>>;
```

---

## 6. Utility Type Pitfalls

```typescript
// Omit loses union discrimination — use DistributiveOmit for unions
type Event = { type: 'click'; x: number } | { type: 'key'; key: string };

type BadOmit = Omit<Event, 'type'>;
// { x?: number; key?: string } — wrong! merged into one object

// DistributiveOmit — distributes over union members
type DistributiveOmit<T, K extends keyof T> = T extends any ? Omit<T, K> : never;
type GoodOmit = DistributiveOmit<Event, 'type'>;
// { x: number } | { key: string } — correct ✓
```

**Rule:** When using `Omit`/`Pick` on union types, use a distributive version to preserve the union structure.

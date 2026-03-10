# Specialist C: Union & Intersection Types

## Scope
Union types (`|`), intersection types (`&`), discriminated unions, exhaustive checks.

---

## 1. Union Types

A union type means a value can be **one of several types**:

```typescript
type StringOrNumber = string | number;
type NullableString = string | null;
type Id = string | number;
```

**Rules:**
- Narrow before using. TypeScript will not let you call `.toUpperCase()` on `string | number` without checking.
- `string | null` and `string | undefined` are different. Know which is semantically correct.
- Avoid large untagged unions — use discriminated unions instead.

---

## 2. Discriminated Unions (Tagged Unions)

A discriminated union has a common **literal property** (the discriminant) that TypeScript uses to narrow:

```typescript
type LoadingState = { status: 'loading' };
type SuccessState = { status: 'success'; data: User };
type ErrorState   = { status: 'error'; error: Error };

type FetchState = LoadingState | SuccessState | ErrorState;

function render(state: FetchState) {
  switch (state.status) {
    case 'loading': return <Spinner />;
    case 'success': return <UserCard user={state.data} />;
    case 'error':   return <ErrorBanner error={state.error} />;
    default:
      // exhaustive check — if a new case is added, this fails at compile time
      assertNever(state);
  }
}

function assertNever(x: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(x)}`);
}
```

**Rules:**
- Always use a string literal discriminant (e.g., `status`, `kind`, `type`), not a boolean.
- Every arm of a `switch`/`if` on the discriminant should be covered.
- Add `assertNever` in the `default` branch for exhaustive checking.
- Model all async/fetch state, result types, and event variants as discriminated unions.

---

## 3. Result Type Pattern

```typescript
type Ok<T>  = { ok: true;  value: T };
type Err<E> = { ok: false; error: E };
type Result<T, E = Error> = Ok<T> | Err<E>;

function ok<T>(value: T): Ok<T> { return { ok: true, value }; }
function err<E>(error: E): Err<E> { return { ok: false, error }; }

// Usage
async function fetchUser(id: string): Promise<Result<User>> {
  try {
    const user = await db.users.find(id);
    return user ? ok(user) : err(new Error('Not found'));
  } catch (e) {
    return err(e instanceof Error ? e : new Error(String(e)));
  }
}

const result = await fetchUser('123');
if (result.ok) {
  console.log(result.value.name); // narrowed to User
} else {
  console.error(result.error.message); // narrowed to Error
}
```

---

## 4. Intersection Types

An intersection type means a value must satisfy **all** the constituent types simultaneously:

```typescript
type Named = { name: string };
type Dated = { createdAt: Date };
type NamedAndDated = Named & Dated;
// equivalent to: { name: string; createdAt: Date }
```

**Rules:**
- If the same key has incompatible types across intersected types, the result is `never` for that key:
  ```typescript
  type A = { x: string };
  type B = { x: number };
  type C = A & B; // C['x'] is never
  ```
- Use intersections to compose types from smaller pieces (mixins, extensions).
- Prefer intersection over deep `interface extends` for flat composition.

---

## 5. Union Narrowing Patterns

```typescript
// typeof
function process(val: string | number) {
  if (typeof val === 'string') {
    return val.toUpperCase(); // string
  }
  return val.toFixed(2); // number
}

// in operator
type Cat = { meow(): void };
type Dog = { bark(): void };

function speak(animal: Cat | Dog) {
  if ('meow' in animal) {
    animal.meow();
  } else {
    animal.bark();
  }
}

// instanceof
function format(val: Date | string): string {
  if (val instanceof Date) {
    return val.toISOString();
  }
  return val;
}
```

---

## 6. Optional vs Union with `null`/`undefined`

```typescript
// Optional property: may be absent
type A = { name?: string };

// Property present but nullable
type B = { name: string | null };

// Property present but possibly undefined
type C = { name: string | undefined };
```

**Rule:** Choose based on domain semantics:
- Use `?` when the property might not apply at all.
- Use `| null` when the absence of a value is a meaningful state (e.g., a field explicitly cleared).
- Avoid `| undefined` for properties; use `?` instead.

# Specialist G: Type Narrowing

## Scope
`typeof`, `instanceof`, `in`, truthiness narrowing, equality narrowing, custom type guards, assertion functions, `never` and `assertNever`, control flow analysis.

---

## 1. `typeof` Narrowing

```typescript
function process(val: string | number | boolean) {
  if (typeof val === 'string') {
    return val.toUpperCase(); // string
  }
  if (typeof val === 'number') {
    return val.toFixed(2); // number
  }
  return val; // boolean
}
```

`typeof` narrowing works for: `string`, `number`, `boolean`, `symbol`, `bigint`, `object`, `function`, `undefined`.

**Limitation:** `typeof null === 'object'` — use `val !== null` for null checks.

---

## 2. `instanceof` Narrowing

```typescript
function formatDate(val: Date | string): string {
  if (val instanceof Date) {
    return val.toISOString(); // Date
  }
  return val; // string
}

// Works for custom classes
class ApiError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
  }
}

function handleError(e: unknown): string {
  if (e instanceof ApiError) {
    return `API ${e.statusCode}: ${e.message}`;
  }
  if (e instanceof Error) {
    return e.message;
  }
  return String(e);
}
```

---

## 3. `in` Narrowing

```typescript
type Cat = { meow(): void; purr(): void };
type Dog = { bark(): void; fetch(): void };

function makeSound(animal: Cat | Dog) {
  if ('meow' in animal) {
    animal.meow(); // Cat
  } else {
    animal.bark(); // Dog
  }
}
```

**Rule:** Use `in` for structural narrowing when there's no common discriminant property.

---

## 4. Discriminated Union Narrowing

```typescript
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'rect'; width: number; height: number }
  | { kind: 'triangle'; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':   return Math.PI * shape.radius ** 2;
    case 'rect':     return shape.width * shape.height;
    case 'triangle': return 0.5 * shape.base * shape.height;
    default:         return assertNever(shape);
  }
}
```

---

## 5. Truthiness Narrowing

```typescript
function greet(name: string | null | undefined) {
  if (name) {
    return `Hello, ${name}`; // string (truthy — excludes null, undefined, '')
  }
  return 'Hello, stranger';
}
```

**Caution:** Truthiness narrowing excludes `0`, `''`, `false`, `NaN`, `null`, `undefined`. This can be wrong:
```typescript
function double(n: number | null) {
  if (n) return n * 2; // BUG! n === 0 is falsy but valid
  return 0;
}

// CORRECT
function double(n: number | null) {
  if (n !== null) return n * 2;
  return 0;
}
```

**Rule:** For nullable numbers, use explicit `!== null` not truthiness.

---

## 6. Custom Type Guards (`is` predicate)

```typescript
function isString(val: unknown): val is string {
  return typeof val === 'string';
}

function isUser(val: unknown): val is User {
  return (
    typeof val === 'object' &&
    val !== null &&
    'id' in val &&
    'name' in val &&
    typeof (val as any).id === 'string' &&
    typeof (val as any).name === 'string'
  );
}

// Usage
const data: unknown = JSON.parse(raw);
if (isUser(data)) {
  console.log(data.name); // narrowed to User ✓
}
```

**Rules:**
- Type guard functions must have `value is T` as their return type.
- The guard is your responsibility — TypeScript trusts whatever you assert. Make the checks thorough.
- For complex validation, prefer a schema library (Zod, Valibot) over hand-written guards.

---

## 7. Assertion Functions

Assertion functions narrow based on a `throws` contract — if they don't throw, the type is asserted:

```typescript
function assertIsUser(val: unknown): asserts val is User {
  if (!isUser(val)) {
    throw new Error('Expected User, got: ' + JSON.stringify(val));
  }
}

function assertDefined<T>(val: T | null | undefined): asserts val is T {
  if (val == null) throw new Error('Expected defined value');
}

// Usage
const data: unknown = JSON.parse(raw);
assertIsUser(data); // throws if not User
console.log(data.name); // narrowed to User after assertion ✓
```

---

## 8. `never` and `assertNever`

Use `never` to mark code that should be unreachable:

```typescript
function assertNever(x: never): never {
  throw new Error(`Unhandled case: ${JSON.stringify(x)}`);
}

// Exhaustive switch
type Status = 'active' | 'inactive' | 'pending';

function describe(s: Status): string {
  switch (s) {
    case 'active':   return 'Currently active';
    case 'inactive': return 'Not active';
    case 'pending':  return 'Waiting';
    default:         return assertNever(s); // compile error if a case is missing ✓
  }
}
```

**Rule:** Every switch/if-else on a discriminated union or enum must end with `assertNever`. This ensures adding a new variant is a compile-time error, not a silent runtime bug.

---

## 9. Control Flow Analysis

TypeScript tracks types through control flow:

```typescript
function process(val: string | null): string {
  if (val === null) {
    return 'default'; // return exits
  }
  // val is string here — TypeScript knows null was handled
  return val.toUpperCase();
}

// After assignment
let x: string | number = Math.random() > 0.5 ? 'hello' : 42;
if (typeof x === 'string') {
  x.toUpperCase(); // string
}
x; // string | number (narrowing doesn't persist past the block)
```

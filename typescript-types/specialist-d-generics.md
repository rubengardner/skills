# Specialist D: Generics

## Scope
Generic functions, generic interfaces/types, type parameter constraints, defaults, `const` type parameters, `NoInfer`.

---

## 1. Basic Generics

```typescript
// Generic function
function identity<T>(value: T): T {
  return value;
}

// Generic type alias
type Pair<A, B> = { first: A; second: B };

// Generic interface
interface Container<T> {
  value: T;
  map<U>(fn: (val: T) => U): Container<U>;
}
```

**Rules:**
- Use single-letter names for simple type parameters: `T`, `U`, `K`, `V`.
- Use descriptive names for domain-specific parameters: `TEntity`, `TResponse`, `TError`.
- Don't add generics unless the type parameter is actually used to connect input/output types.

---

## 2. Constraints (`extends`)

Constrain a type parameter to a subset of types:

```typescript
// T must have a name property
function getName<T extends { name: string }>(item: T): string {
  return item.name;
}

// K must be a key of T
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// T must be an array
function first<T extends unknown[]>(arr: T): T[0] | undefined {
  return arr[0];
}
```

**Rules:**
- Always constrain when you're accessing a property on `T`.
- `extends object` is too broad — be specific.
- `extends Record<string, unknown>` is a common constraint for "any plain object".

---

## 3. Default Type Parameters (TS 4.7+)

```typescript
type ApiResponse<T = unknown> = {
  data: T;
  status: number;
};

// With default — T is unknown
const res: ApiResponse = { data: JSON.parse('{}'), status: 200 };

// With explicit — T is User
const res: ApiResponse<User> = { data: user, status: 200 };
```

---

## 4. `const` Type Parameters (TS 5.0+)

Prevents widening of literal types passed as arguments:

```typescript
// Without const — literals widen to string
function createTuple<T extends string[]>(...args: T): T {
  return args;
}
createTuple('a', 'b'); // type: string[]

// With const — preserves literals
function createTuple<const T extends string[]>(...args: T): T {
  return args;
}
createTuple('a', 'b'); // type: ['a', 'b'] ✓
```

**Rule:** Use `const` type parameters when the function's purpose is to preserve literal types (e.g., building readonly tuples, route definitions, event maps).

---

## 5. `NoInfer<T>` — Prevent Inference from a Site (TS 5.4+)

```typescript
// Without NoInfer — TypeScript infers T from BOTH arguments
function createEvent<T extends string>(type: T, fallback: T): T {
  return fallback ?? type;
}
createEvent('click', 'scroll'); // T inferred as 'click' | 'scroll' (not what we want)

// With NoInfer — T is only inferred from `type`, `fallback` just must match
function createEvent<T extends string>(type: T, fallback: NoInfer<T>): T {
  return fallback ?? type;
}
createEvent('click', 'scroll'); // Error! 'scroll' is not assignable to 'click' ✓
```

---

## 6. Generic Classes

```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }
}

const stack = new Stack<number>();
stack.push(1);
stack.push('hello'); // Error! string is not number ✓
```

---

## 7. Variance (Covariance / Contravariance)

TypeScript uses structural compatibility. In strict function types:
- Function **parameters** are checked **contravariantly** (with `strictFunctionTypes`)
- Function **return types** are checked **covariantly**

```typescript
type Handler<T> = (event: T) => void;

// A Handler<MouseEvent> is NOT assignable to Handler<Event>
// because the parameter is contravariant
```

Explicit variance annotations (TS 4.7+):
```typescript
type Provider<out T> = () => T;   // covariant — T only produced
type Consumer<in T> = (x: T) => void; // contravariant — T only consumed
```

**Rule:** Explicit variance annotations are only needed in complex generic hierarchies or performance-sensitive type checking. For most app code, let TypeScript infer variance.

---

## 8. Avoid Generic Anti-Patterns

```typescript
// BAD — T not used to connect types; just use `unknown`
function log<T>(value: T): void {
  console.log(value);
}

// GOOD — T connects input and output
function wrap<T>(value: T): { value: T } {
  return { value };
}

// BAD — T extends any (pointless constraint)
function process<T extends any>(value: T): T { return value; }

// BAD — unnecessary generic when overloads are clearer
function parse<T>(raw: string): T { return JSON.parse(raw) as T; }
// Caller writes: parse<User>(raw) — forces an unsafe cast
// BETTER — use a validator/decoder parameter
function parse<T>(raw: string, decode: (x: unknown) => T): T {
  return decode(JSON.parse(raw));
}
```

# Specialist F: Advanced Types

## Scope
Conditional types, `infer`, mapped types, template literal types, recursive types.

---

## 1. Conditional Types

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;   // true
type B = IsString<number>;   // false
```

**Distributive conditional types** — when `T` is a bare type parameter, the condition is distributed over union members:
```typescript
type ToArray<T> = T extends unknown ? T[] : never;

type A = ToArray<string | number>; // string[] | number[]
// (NOT (string | number)[])
```

To prevent distribution, wrap `T` in a tuple:
```typescript
type ToArrayNonDist<T> = [T] extends [unknown] ? T[] : never;
type B = ToArrayNonDist<string | number>; // (string | number)[]
```

---

## 2. `infer` — Extract Types Within Conditionals

`infer` introduces a type variable to capture part of a matched type:

```typescript
// Extract return type
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

// Extract first element of tuple
type Head<T extends unknown[]> = T extends [infer H, ...unknown[]] ? H : never;
type H = Head<[string, number, boolean]>; // string

// Extract Promise inner type
type Unwrap<T> = T extends Promise<infer U> ? U : T;
type U = Unwrap<Promise<string>>; // string

// Extract array element type
type ElementOf<T> = T extends (infer E)[] ? E : never;
type E = ElementOf<string[]>; // string
```

**Rule:** `infer` is only valid inside `extends` in a conditional type. Use it to build utility types that extract nested type information.

---

## 3. Mapped Types

Transform the properties of an existing type:

```typescript
// Basic mapped type — same as Readonly<T>
type ReadonlyT<T> = {
  readonly [K in keyof T]: T[K];
};

// Mutable — remove readonly
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

// Required — remove optional
type Required<T> = {
  [K in keyof T]-?: T[K];
};

// Nullable — add null
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};
```

### Key remapping with `as` (TS 4.1+)
```typescript
// Prefix all keys
type Prefixed<T> = {
  [K in keyof T as `get_${string & K}`]: T[K];
};

type User = { id: string; name: string };
type UserGetters = Prefixed<User>;
// { get_id: string; get_name: string }

// Filter keys
type PickByValue<T, V> = {
  [K in keyof T as T[K] extends V ? K : never]: T[K];
};

type StringFields = PickByValue<{ a: string; b: number; c: string }, string>;
// { a: string; c: string }
```

---

## 4. Template Literal Types (TS 4.1+)

Build string types from template patterns:

```typescript
type Greeting = `Hello, ${string}`;
type Color = 'red' | 'blue' | 'green';
type ColorKey = `${Color}Color`; // 'redColor' | 'blueColor' | 'greenColor'

// Combine with keyof
type EventName<T> = `on${Capitalize<string & keyof T>}`;

type ButtonEvents = EventName<{ click: void; focus: void; blur: void }>;
// 'onClick' | 'onFocus' | 'onBlur'
```

### Built-in string manipulation types
```typescript
Uppercase<'hello'>     // 'HELLO'
Lowercase<'HELLO'>     // 'hello'
Capitalize<'hello'>    // 'Hello'
Uncapitalize<'Hello'>  // 'hello'
```

### Practical: typed CSS property names
```typescript
type CSSProperty = 'margin' | 'padding' | 'border';
type CSSDirectional = `${CSSProperty}-${'top' | 'right' | 'bottom' | 'left'}`;
// 'margin-top' | 'margin-right' | ... | 'border-bottom' | 'border-left'
```

---

## 5. Recursive Types

```typescript
// JSON value
type Json =
  | string
  | number
  | boolean
  | null
  | Json[]
  | { [key: string]: Json };

// Deep readonly
type DeepReadonly<T> = T extends (infer U)[]
  ? ReadonlyArray<DeepReadonly<U>>
  : T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

// Deep partial
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;
```

**Rules:**
- TypeScript limits recursion depth. For very deep recursive types, you may hit "Type instantiation is excessively deep" errors. Keep recursive utility types bounded.
- Recursive types work correctly in TypeScript 4.1+. Earlier versions required workarounds.

---

## 6. When NOT to Use Advanced Types

Advanced types add cognitive overhead. Only use them when:
1. A simpler type would require manual repetition across 3+ places
2. You're building a reusable utility type for a library/shared code
3. The type correctly captures a runtime invariant that would otherwise be lost

**Do NOT use conditional/mapped types to:**
- Work around a design smell (fix the architecture instead)
- Show off — prioritize readability over cleverness
- Solve a one-time problem (just write the type out manually)

# Specialist B: Hook Typing

## Scope
`useState`, `useRef`, `useReducer`, `useEffect`, `useCallback`, `useMemo`, custom hooks.

---

## 1. `useState`

```tsx
// TypeScript infers the type from the initial value
const [count, setCount] = useState(0);        // [number, Dispatch<SetStateAction<number>>]
const [name, setName] = useState('');         // [string, ...]
const [user, setUser] = useState<User | null>(null); // explicit type required — can't infer from null
```

**Rules:**
- Provide the type parameter when the initial value is `null`, `undefined`, or an empty array/object.
- `useState<User | null>(null)` — always include `null` in the type when the initial state is null.
- Never use `useState<any>` — if you don't know the type, use `useState<unknown>` and narrow.

```tsx
// Loading state — use discriminated union, not separate booleans
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

const [state, setState] = useState<FetchState<User>>({ status: 'idle' });
```

---

## 2. `useRef`

`useRef` has two distinct use cases with different typing:

### DOM Refs (ref attached to an element)
```tsx
// Initial value must be null for DOM refs
const inputRef = useRef<HTMLInputElement>(null);

// ref.current is HTMLInputElement | null
// Must check before accessing
function focusInput() {
  inputRef.current?.focus();
}

return <input ref={inputRef} />;
```

### Mutable Value Refs (not a DOM element)
```tsx
// Initial value provided — ref.current is mutable
const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
const countRef = useRef(0); // inferred as MutableRefObject<number>

// Use non-null initial value so ref.current is not null
const prevValueRef = useRef<string>('');
```

**Rules:**
- DOM refs: `useRef<HTMLElement>(null)` — initial value is `null`, `ref.current` will be `HTMLElement | null`.
- Mutable refs: `useRef<T>(initialValue)` — initial value provided, `ref.current` is `T` (not null).
- Never initialize a DOM ref with a non-null value — it bypasses React's ref assignment.

---

## 3. `useReducer`

Always use discriminated union action types:

```tsx
type CounterAction =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'reset'; payload: number };

type CounterState = {
  count: number;
  lastAction: CounterAction['type'] | null;
};

function counterReducer(state: CounterState, action: CounterAction): CounterState {
  switch (action.type) {
    case 'increment':
      return { ...state, count: state.count + 1, lastAction: 'increment' };
    case 'decrement':
      return { ...state, count: state.count - 1, lastAction: 'decrement' };
    case 'reset':
      return { count: action.payload, lastAction: 'reset' }; // payload available ✓
    default:
      return assertNever(action); // exhaustive check ✓
  }
}

const [state, dispatch] = useReducer(counterReducer, { count: 0, lastAction: null });
dispatch({ type: 'reset', payload: 10 });
```

**Rule:** Prefer `useReducer` over multiple `useState` calls when state transitions are interrelated or require complex logic.

---

## 4. `useEffect`

```tsx
useEffect(() => {
  // Effect body — return value must be void | (() => void)
  const subscription = subscribe(userId);

  return () => {
    subscription.unsubscribe(); // cleanup
  };
}, [userId]); // dependency array

// Async effects — DO NOT make the effect callback async
// BAD:
useEffect(async () => { ... }, []); // async returns Promise, not cleanup fn

// GOOD:
useEffect(() => {
  let cancelled = false;

  async function load() {
    const data = await fetchUser(userId);
    if (!cancelled) setState(data);
  }

  load();
  return () => { cancelled = true; };
}, [userId]);
```

**Rules:**
- Dependency arrays must include all values used inside the effect. ESLint rule `react-hooks/exhaustive-deps` enforces this.
- For async effects, define an inner async function and call it. Never make the effect callback `async`.
- Use a `cancelled` flag or `AbortController` to prevent state updates after unmount.

---

## 5. Custom Hooks

```tsx
// Return type: object for 3+ values, tuple for 2 values
function useToggle(initialValue = false): [boolean, () => void] {
  const [value, setValue] = useState(initialValue);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle];
}

// Object return for complex hooks
type UseUserResult = {
  user: User | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
};

function useUser(id: string): UseUserResult {
  const [state, setState] = useState<{
    user: User | null;
    loading: boolean;
    error: Error | null;
  }>({ user: null, loading: true, error: null });

  const refetch = useCallback(async () => {
    setState(s => ({ ...s, loading: true, error: null }));
    try {
      const user = await fetchUser(id);
      setState({ user, loading: false, error: null });
    } catch (e) {
      setState({ user: null, loading: false, error: e instanceof Error ? e : new Error(String(e)) });
    }
  }, [id]);

  useEffect(() => { refetch(); }, [refetch]);

  return { ...state, refetch };
}
```

**Rules:**
- Custom hooks must start with `use` — this is enforced by the `react-hooks/rules-of-hooks` ESLint rule.
- Always export a named return type (`UseXxxResult`) for hooks with complex return shapes.
- Tuple returns: use `as const` or explicit tuple return type to prevent widening to `(A | B)[]`.
  ```tsx
  function useBool(): [boolean, () => void] {
    const [v, setV] = useState(false);
    return [v, () => setV(x => !x)]; // typed as tuple ✓
  }
  ```

---

## 6. `useCallback` and `useMemo` Typing

TypeScript infers types from the callback:

```tsx
// useCallback — type inferred from the function body
const handleSubmit = useCallback((e: React.FormEvent) => {
  e.preventDefault();
  submitForm(values);
}, [values, submitForm]);
// type: (e: React.FormEvent) => void

// useMemo — type inferred from the return value
const sortedItems = useMemo(
  () => [...items].sort((a, b) => a.name.localeCompare(b.name)),
  [items]
);
// type: typeof items[number][]
```

**Rule:** Never pass a type parameter to `useCallback` or `useMemo` explicitly — let TypeScript infer. If inference fails, fix the types of the values inside, not the hook call.

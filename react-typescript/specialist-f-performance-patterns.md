# Specialist F: Performance Patterns

## Scope
`React.memo`, `useMemo`, `useCallback` — when to use them, how to type them, and when NOT to use them.

---

## 1. `React.memo`

`memo` prevents re-renders when props haven't changed (shallow equality by default):

```tsx
type UserCardProps = {
  user: { id: string; name: string; avatar: string };
  onSelect: (id: string) => void;
};

const UserCard = memo(function UserCard({ user, onSelect }: UserCardProps) {
  return (
    <div onClick={() => onSelect(user.id)}>
      <img src={user.avatar} alt={user.name} />
      <span>{user.name}</span>
    </div>
  );
});

UserCard.displayName = 'UserCard';
```

**Rules:**
- Always set `displayName` on memoized components — React DevTools shows "Memo(UserCard)" otherwise.
- `memo` compares props shallowly. If you pass a new object/array/function on every render, memo is useless.
- For custom comparison, use the second argument:
  ```tsx
  const UserCard = memo(
    function UserCard({ user, onSelect }: UserCardProps) { ... },
    (prevProps, nextProps) => prevProps.user.id === nextProps.user.id
  );
  ```
- The comparator returns `true` to SKIP re-render (props are equal), `false` to re-render.

---

## 2. `useCallback`

Stabilize function identity across renders (prevents passing new function references to memoized children):

```tsx
function UserList({ users }: { users: User[] }) {
  const [filter, setFilter] = useState('');

  // Without useCallback — new function on every render, breaks memo
  // const handleSelect = (id: string) => navigate(`/users/${id}`);

  // With useCallback — stable reference as long as navigate doesn't change
  const handleSelect = useCallback(
    (id: string) => navigate(`/users/${id}`),
    [navigate] // navigate from useNavigate is stable
  );

  const filtered = useMemo(
    () => users.filter(u => u.name.toLowerCase().includes(filter)),
    [users, filter]
  );

  return (
    <>
      <input value={filter} onChange={e => setFilter(e.target.value)} />
      {filtered.map(u => (
        <UserCard key={u.id} user={u} onSelect={handleSelect} />
      ))}
    </>
  );
}
```

**Rules:**
- `useCallback` is only useful when:
  1. The function is passed to a `memo`-wrapped component, OR
  2. The function is a dependency in another `useEffect`/`useMemo`/`useCallback`
- Do NOT add `useCallback` to every function "just in case." It has overhead and adds noise.
- The ESLint rule `react-hooks/exhaustive-deps` will catch missing dependencies.

---

## 3. `useMemo`

Memoize expensive computed values:

```tsx
function DataGrid({ rows, sortKey, sortDir }: DataGridProps) {
  const sortedRows = useMemo(() => {
    return [...rows].sort((a, b) => {
      const cmp = String(a[sortKey]).localeCompare(String(b[sortKey]));
      return sortDir === 'asc' ? cmp : -cmp;
    });
  }, [rows, sortKey, sortDir]);

  return (
    <table>
      {sortedRows.map(row => <Row key={row.id} data={row} />)}
    </table>
  );
}
```

**Rules:**
- `useMemo` is only worth it when the computation is measurably expensive OR when you need a stable object/array reference as a prop to a memoized child.
- Do NOT use `useMemo` for simple transformations (filtering a 10-item array, formatting a string).
- Never use `useMemo` as a substitute for proper state management.

---

## 4. Common Memo Trap — Unstable References

```tsx
// BAD — new object on every render breaks memo
function Parent() {
  return <Child config={{ timeout: 5000 }} />; // new object reference each render
}

// GOOD — stable reference
const CONFIG = { timeout: 5000 } as const; // module-level constant

function Parent() {
  return <Child config={CONFIG} />;
}

// Or with useMemo if config depends on props/state
function Parent({ timeoutMs }: { timeoutMs: number }) {
  const config = useMemo(() => ({ timeout: timeoutMs }), [timeoutMs]);
  return <Child config={config} />;
}
```

---

## 5. When NOT to Optimize

Before adding memo/useCallback/useMemo, ask:
1. Have I measured a performance problem? (React DevTools Profiler)
2. Is this component actually re-rendering unnecessarily?
3. Is the computation actually expensive?

**Do NOT add memo to:**
- Components that always re-render when their parent does (because props change every time anyway)
- Tiny components that render in < 1ms
- Components with many props — comparisons themselves have cost

**Do NOT add useCallback to:**
- Functions not passed to memoized children
- Functions not listed as effect dependencies
- Async functions that create new closures anyway

---

## 6. `lazy` and `Suspense` Typing

```tsx
import { lazy, Suspense } from 'react';

// lazy accepts a function returning Promise<{ default: Component }>
const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));

function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Router>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Router>
    </Suspense>
  );
}
```

**Rules:**
- `lazy` only works with default exports.
- The `fallback` prop of `Suspense` is `React.ReactNode` — pass any renderable value.
- Wrap lazy routes at the router level, not individually, to avoid nested Suspense boundaries for navigations.

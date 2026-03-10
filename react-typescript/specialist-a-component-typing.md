# Specialist A: Component Typing

## Scope
Function component declarations, props typing, children, `displayName`, `forwardRef`, extending native elements.

---

## 1. The Correct Way to Write a Component

```tsx
// GOOD — plain function declaration, explicit props type
type ButtonProps = {
  label: string;
  onClick?: () => void;
  disabled?: boolean;
};

function Button({ label, onClick, disabled = false }: ButtonProps) {
  return (
    <button onClick={onClick} disabled={disabled}>
      {label}
    </button>
  );
}

export { Button };
export type { ButtonProps };
```

**Rules:**
- Never use `React.FC` or `React.FunctionComponent`. It was deprecated in React 18 typedefs and adds no value.
- Function declaration (`function Foo`) is preferred over arrow function (`const Foo = () =>`) for components — it reads as a component, not a variable.
- Always export the props type alongside the component.
- Default exports for components are discouraged in large codebases — named exports enable better tree-shaking and refactoring.

---

## 2. Typing `children`

```tsx
import type { ReactNode, ReactElement } from 'react';

// Most common — accepts anything React can render
type LayoutProps = {
  children: ReactNode;
};

// Strict — only accepts React elements (not strings, numbers, null)
type SlotProps = {
  children: ReactElement;
};

// Render prop pattern
type ListProps<T> = {
  items: T[];
  renderItem: (item: T) => ReactElement;
};
```

**Rules:**
- Use `ReactNode` for `children` in 99% of cases. It accepts elements, strings, numbers, fragments, portals, null, undefined.
- Use `ReactElement` only when you need to clone or inspect the child.
- Never use `JSX.Element` — it's a global and doesn't survive strict JSX transforms.
- Do NOT use `React.FC` solely because it infers children — just type `children` explicitly.

---

## 3. Extending Native HTML Elements

```tsx
import type { ComponentPropsWithoutRef } from 'react';

// Extend <button> with additional props
type ButtonProps = ComponentPropsWithoutRef<'button'> & {
  variant?: 'primary' | 'secondary' | 'ghost';
  loading?: boolean;
};

function Button({ variant = 'primary', loading, children, ...props }: ButtonProps) {
  return (
    <button
      {...props}
      disabled={loading || props.disabled}
      className={`btn btn-${variant}`}
    >
      {loading ? <Spinner /> : children}
    </button>
  );
}
```

**Rules:**
- `ComponentPropsWithoutRef<'element'>` — use when the component does NOT forward refs.
- `ComponentPropsWithRef<'element'>` — use when the component DOES forward refs (rare; use `forwardRef` instead).
- Always spread `...props` last to let consumers override props you set internally.
- Override `disabled` carefully — don't silently ignore the consumer's `disabled` value.

---

## 4. `forwardRef`

```tsx
import { forwardRef, type Ref, type ComponentPropsWithoutRef } from 'react';

type InputProps = ComponentPropsWithoutRef<'input'> & {
  label: string;
  error?: string;
};

const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ label, error, ...props }, ref) => {
    return (
      <div className="field">
        <label>{label}</label>
        <input ref={ref} {...props} />
        {error && <span className="error">{error}</span>}
      </div>
    );
  }
);

Input.displayName = 'Input';

export { Input };
export type { InputProps };
```

**Rules:**
- `forwardRef<ElementType, PropsType>` — always specify both type parameters explicitly.
- Always set `displayName` on `forwardRef` components — React DevTools shows the component name, not "ForwardRef".
- The ref type is the DOM element type (`HTMLInputElement`, `HTMLButtonElement`, etc.), not the component.
- Never use `useRef` inside a `forwardRef` component for the forwarded ref — the ref comes from the parent.

---

## 5. Component Composition Patterns

### Compound Components
```tsx
type CardContextValue = {
  elevated: boolean;
};

const CardContext = createContext<CardContextValue | undefined>(undefined);

function useCardContext() {
  const ctx = useContext(CardContext);
  if (!ctx) throw new Error('Must be used within <Card>');
  return ctx;
}

function Card({ elevated = false, children }: { elevated?: boolean; children: ReactNode }) {
  return (
    <CardContext.Provider value={{ elevated }}>
      <div className={`card ${elevated ? 'card-elevated' : ''}`}>{children}</div>
    </CardContext.Provider>
  );
}

function CardHeader({ children }: { children: ReactNode }) {
  const { elevated } = useCardContext();
  return <div className={`card-header ${elevated ? 'elevated' : ''}`}>{children}</div>;
}

Card.Header = CardHeader;
```

### Render Props
```tsx
type DataFetcherProps<T> = {
  url: string;
  render: (data: T, loading: boolean) => ReactElement;
};

function DataFetcher<T>({ url, render }: DataFetcherProps<T>) {
  const { data, loading } = useFetch<T>(url);
  return render(data as T, loading);
}
```

---

## 6. Component File Conventions

```
components/
  Button/
    Button.tsx          # component + types
    Button.test.tsx     # tests
    index.ts            # re-exports: export { Button } from './Button'
```

- One component per file (except tiny sub-components used only by the parent).
- Types in the same file as the component unless shared across multiple files.
- Shared types go in `types/` or a `components/shared-types.ts`.

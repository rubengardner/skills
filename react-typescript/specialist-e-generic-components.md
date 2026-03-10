# Specialist E: Generic Components

## Scope
Generic function components, the `<T,>` comma, `forwardRef` with generics, polymorphic `as` prop.

---

## 1. Basic Generic Component

```tsx
// The trailing comma after T is required in .tsx files
// to disambiguate from JSX tag syntax
type ListProps<T> = {
  items: T[];
  keyExtractor: (item: T) => string;
  renderItem: (item: T) => React.ReactNode;
  emptyMessage?: string;
};

function List<T>({ items, keyExtractor, renderItem, emptyMessage = 'No items' }: ListProps<T>) {
  if (items.length === 0) return <p>{emptyMessage}</p>;

  return (
    <ul>
      {items.map(item => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Usage — T is inferred as User
<List
  items={users}
  keyExtractor={u => u.id}
  renderItem={u => <UserCard user={u} />}
/>
```

---

## 2. Generic Component with Constraints

```tsx
// T must have an id property
type TableProps<T extends { id: string }> = {
  rows: T[];
  columns: Array<{
    key: keyof T;
    label: string;
    render?: (value: T[keyof T], row: T) => React.ReactNode;
  }>;
};

function Table<T extends { id: string }>({ rows, columns }: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map(col => (
            <th key={String(col.key)}>{col.label}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {rows.map(row => (
          <tr key={row.id}>
            {columns.map(col => (
              <td key={String(col.key)}>
                {col.render
                  ? col.render(row[col.key], row)
                  : String(row[col.key])}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

---

## 3. Generic `forwardRef` Component

`forwardRef` is not naturally generic. The solution is a wrapper function with a type assertion:

```tsx
import { forwardRef, type Ref } from 'react';

type SelectProps<T> = {
  options: T[];
  value: T | null;
  onChange: (value: T) => void;
  getLabel: (option: T) => string;
  getValue: (option: T) => string;
};

// Inner component typed correctly
function SelectInner<T>(
  { options, value, onChange, getLabel, getValue }: SelectProps<T>,
  ref: Ref<HTMLSelectElement>
) {
  const currentValue = value ? getValue(value) : '';

  const handleChange = (e: React.ChangeEvent<HTMLSelectElement>) => {
    const selected = options.find(opt => getValue(opt) === e.target.value);
    if (selected !== undefined) onChange(selected);
  };

  return (
    <select ref={ref} value={currentValue} onChange={handleChange}>
      {options.map(opt => (
        <option key={getValue(opt)} value={getValue(opt)}>
          {getLabel(opt)}
        </option>
      ))}
    </select>
  );
}

// Cast to recover the generic signature
const Select = forwardRef(SelectInner) as <T>(
  props: SelectProps<T> & { ref?: Ref<HTMLSelectElement> }
) => React.ReactElement;

export { Select };
```

---

## 4. Polymorphic `as` Prop

A polymorphic component renders as different HTML elements or components:

```tsx
import type {
  ComponentPropsWithoutRef,
  ElementType,
  ReactElement,
} from 'react';

type PolymorphicProps<E extends ElementType> = {
  as?: E;
  children?: React.ReactNode;
} & Omit<ComponentPropsWithoutRef<E>, 'as' | 'children'>;

function Text<E extends ElementType = 'p'>({
  as,
  children,
  ...props
}: PolymorphicProps<E>): ReactElement {
  const Component = as ?? 'p';
  return <Component {...props}>{children}</Component>;
}

// Usage
<Text>Paragraph (default)</Text>
<Text as="h1" className="heading">Heading</Text>
<Text as="span" aria-label="label">Span</Text>
<Text as={Link} href="/about">Link component</Text>
```

**Complexity warning:** Polymorphic components are type-system-heavy. Only implement this pattern if your design system genuinely needs it. For most cases, separate components (`<Heading>`, `<Paragraph>`) are cleaner.

---

## 5. Generic Hook Paired with Generic Component

```tsx
// Generic hook
function useSelection<T>(items: T[], getKey: (item: T) => string) {
  const [selected, setSelected] = useState<Set<string>>(new Set());

  const toggle = useCallback((item: T) => {
    setSelected(prev => {
      const next = new Set(prev);
      const key = getKey(item);
      if (next.has(key)) next.delete(key);
      else next.add(key);
      return next;
    });
  }, [getKey]);

  const isSelected = useCallback(
    (item: T) => selected.has(getKey(item)),
    [selected, getKey]
  );

  return { selected, toggle, isSelected };
}

// Generic component using the hook
function CheckList<T extends { id: string; label: string }>({
  items,
}: {
  items: T[];
}) {
  const { toggle, isSelected } = useSelection(items, item => item.id);

  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>
          <input
            type="checkbox"
            checked={isSelected(item)}
            onChange={() => toggle(item)}
          />
          {item.label}
        </li>
      ))}
    </ul>
  );
}
```

---

## 6. TSX Disambiguation Rules

In `.tsx` files, `<T>` is ambiguous with JSX. Always add a comma or constraint:

```tsx
// Ambiguous — TypeScript may parse <T> as JSX
const identity = <T>(x: T) => x;   // might error in .tsx

// Fix 1: trailing comma
const identity = <T,>(x: T) => x;

// Fix 2: constraint
const identity = <T extends unknown>(x: T) => x;

// Fix 3: use function declaration (no ambiguity)
function identity<T>(x: T): T { return x; }
```

**Rule:** Always use a trailing comma `<T,>` in generic arrow functions inside `.tsx` files.

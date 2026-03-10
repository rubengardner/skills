# Specialist C: Event Handling

## Scope
Synthetic event types, `event.target`, `event.currentTarget`, common event handlers, inline vs extracted handlers.

---

## 1. Common React Event Types

All React event types live in the `React` namespace from `@types/react`:

| Event | Type |
|-------|------|
| Click | `React.MouseEvent<HTMLButtonElement>` |
| Change (input) | `React.ChangeEvent<HTMLInputElement>` |
| Change (select) | `React.ChangeEvent<HTMLSelectElement>` |
| Change (textarea) | `React.ChangeEvent<HTMLTextAreaElement>` |
| Submit | `React.FormEvent<HTMLFormElement>` |
| Key down/up/press | `React.KeyboardEvent<HTMLInputElement>` |
| Focus / Blur | `React.FocusEvent<HTMLInputElement>` |
| Drag | `React.DragEvent<HTMLDivElement>` |
| Clipboard | `React.ClipboardEvent<HTMLInputElement>` |
| Pointer | `React.PointerEvent<HTMLDivElement>` |
| Touch | `React.TouchEvent<HTMLDivElement>` |
| Wheel | `React.WheelEvent<HTMLDivElement>` |
| Animation | `React.AnimationEvent<HTMLDivElement>` |

The generic parameter is the **element type** the event is attached to.

---

## 2. Input Change Handler

```tsx
function SearchInput() {
  const [query, setQuery] = useState('');

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value);
  };

  return <input type="text" value={query} onChange={handleChange} />;
}
```

**Rule:** `e.target` is always the element that triggered the event. `e.currentTarget` is the element the handler is attached to. For most input scenarios, `e.target.value` is what you want.

---

## 3. Form Submit Handler

```tsx
function LoginForm() {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const form = e.currentTarget;
    const data = new FormData(form);
    // process...
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" />
      <button type="submit">Login</button>
    </form>
  );
}
```

---

## 4. Button Click Handler

```tsx
// Simple void handler
function DeleteButton({ id }: { id: string }) {
  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    e.stopPropagation(); // stop event bubbling
    deleteItem(id);
  };

  return <button onClick={handleClick}>Delete</button>;
}

// Handler that passes data — use a closure, not event.target tricks
function ItemList({ items }: { items: Item[] }) {
  const handleDelete = (id: string) => () => deleteItem(id);
  // Or: const handleDelete = (id: string) => (_e: React.MouseEvent) => deleteItem(id);

  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>
          {item.name}
          <button onClick={handleDelete(item.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}
```

---

## 5. Keyboard Events

```tsx
function CommandInput() {
  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Enter') {
      e.preventDefault();
      submitCommand(e.currentTarget.value);
    }
    if (e.key === 'Escape') {
      clearInput();
    }
  };

  return <input onKeyDown={handleKeyDown} />;
}
```

**Rule:** Use `e.key` (the logical key string) instead of deprecated `e.keyCode` or `e.which`.

---

## 6. Prop Typing for Event Callbacks

When a parent passes a handler as a prop:

```tsx
// Prefer specific types over React.MouseEventHandler
type ButtonProps = {
  onClick: (e: React.MouseEvent<HTMLButtonElement>) => void;
  onFocus?: (e: React.FocusEvent<HTMLButtonElement>) => void;
};

// Or use the shorthand type aliases from @types/react
type ButtonProps = {
  onClick: React.MouseEventHandler<HTMLButtonElement>;
  onChange?: React.ChangeEventHandler<HTMLInputElement>;
};
```

The shorthand aliases (`MouseEventHandler<T>`) are `(event: MouseEvent<T>) => void` — convenient for simple cases.

---

## 7. `event.target` vs `event.currentTarget`

```tsx
// event.target — the actual element clicked (could be a child)
// event.currentTarget — the element with the handler

function Container() {
  const handleClick = (e: React.MouseEvent<HTMLDivElement>) => {
    // e.currentTarget is always HTMLDivElement (the div)
    // e.target is EventTarget — could be any child element

    if (e.target === e.currentTarget) {
      // clicked directly on the div, not a child
    }
  };

  return (
    <div onClick={handleClick}>
      <button>Child button</button>
    </div>
  );
}
```

**Rule:** Use `e.currentTarget` when you need the element the handler is attached to. Use `e.target` only when you need to identify which child triggered the event, and always cast or check it:

```tsx
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  // e.target is typed as HTMLInputElement because of the generic ✓
  const value = e.target.value;
};
```

---

## 8. Custom Event Emitter Pattern (component to component)

```tsx
// Define the callback type in the props
type SelectProps = {
  options: Array<{ value: string; label: string }>;
  onSelect: (value: string) => void; // not a DOM event — just the value
};

function Select({ options, onSelect }: SelectProps) {
  const handleChange = (e: React.ChangeEvent<HTMLSelectElement>) => {
    onSelect(e.target.value); // extract value, pass up
  };

  return (
    <select onChange={handleChange}>
      {options.map(opt => (
        <option key={opt.value} value={opt.value}>{opt.label}</option>
      ))}
    </select>
  );
}
```

**Rule:** Don't pass raw DOM events up to parent components. Extract the meaningful value and pass that. This decouples the parent from implementation details.

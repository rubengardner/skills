# Specialist D: Context API Typing

## Scope
`createContext`, `useContext`, Provider components, avoiding `null` defaults, multi-context patterns.

---

## 1. The Correct Pattern — No Unsafe Defaults

**Never use `null` as the context default and then non-null assert everywhere.**

```tsx
// BAD — forces every consumer to handle or assert null
const AuthContext = createContext<AuthUser | null>(null);
// Consumer: const user = useContext(AuthContext)! // unsafe assertion
```

**The correct pattern — throw in the hook:**

```tsx
import { createContext, useContext, useState, type ReactNode } from 'react';

type AuthUser = {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
};

type AuthContextValue = {
  user: AuthUser | null;
  login: (credentials: LoginInput) => Promise<void>;
  logout: () => void;
  isAuthenticated: boolean;
};

// Use undefined as default — signals "not provided"
const AuthContext = createContext<AuthContextValue | undefined>(undefined);

// Type-safe hook that throws if used outside provider
function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (ctx === undefined) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return ctx;
}

// Provider component
type AuthProviderProps = {
  children: ReactNode;
};

function AuthProvider({ children }: AuthProviderProps) {
  const [user, setUser] = useState<AuthUser | null>(null);

  const login = async (credentials: LoginInput) => {
    const authUser = await authService.login(credentials);
    setUser(authUser);
  };

  const logout = () => {
    setUser(null);
    authService.logout();
  };

  const value: AuthContextValue = {
    user,
    login,
    logout,
    isAuthenticated: user !== null,
  };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export { AuthProvider, useAuth };
export type { AuthUser, AuthContextValue };
```

---

## 2. Context with Updater Functions

```tsx
type ThemeContextValue = {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
};

const ThemeContext = createContext<ThemeContextValue | undefined>(undefined);

function useTheme(): ThemeContextValue {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used within ThemeProvider');
  return ctx;
}

function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  const toggleTheme = useCallback(() => {
    setTheme(t => (t === 'light' ? 'dark' : 'light'));
  }, []);

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}
```

---

## 3. Context with Reducer

For complex state, combine context with `useReducer`:

```tsx
type CartItem = { id: string; quantity: number; price: number };
type CartState = { items: CartItem[]; total: number };
type CartAction =
  | { type: 'add'; item: CartItem }
  | { type: 'remove'; id: string }
  | { type: 'clear' };

type CartContextValue = {
  state: CartState;
  dispatch: React.Dispatch<CartAction>;
};

const CartContext = createContext<CartContextValue | undefined>(undefined);

function useCart(): CartContextValue {
  const ctx = useContext(CartContext);
  if (!ctx) throw new Error('useCart must be used within CartProvider');
  return ctx;
}

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'add': {
      const items = [...state.items, action.item];
      return { items, total: items.reduce((sum, i) => sum + i.price * i.quantity, 0) };
    }
    case 'remove': {
      const items = state.items.filter(i => i.id !== action.id);
      return { items, total: items.reduce((sum, i) => sum + i.price * i.quantity, 0) };
    }
    case 'clear':
      return { items: [], total: 0 };
    default:
      return assertNever(action);
  }
}

function CartProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, { items: [], total: 0 });
  return (
    <CartContext.Provider value={{ state, dispatch }}>
      {children}
    </CartContext.Provider>
  );
}
```

---

## 4. Splitting Context — Separating State and Dispatch

For performance: consumers that only read state don't re-render when dispatch changes, and vice versa.

```tsx
const CountStateContext = createContext<number | undefined>(undefined);
const CountDispatchContext = createContext<React.Dispatch<CounterAction> | undefined>(undefined);

function useCountState(): number {
  const ctx = useContext(CountStateContext);
  if (ctx === undefined) throw new Error('Must be within CountProvider');
  return ctx;
}

function useCountDispatch(): React.Dispatch<CounterAction> {
  const ctx = useContext(CountDispatchContext);
  if (!ctx) throw new Error('Must be within CountProvider');
  return ctx;
}

function CountProvider({ children }: { children: ReactNode }) {
  const [count, dispatch] = useReducer(counterReducer, 0);
  return (
    <CountStateContext.Provider value={count}>
      <CountDispatchContext.Provider value={dispatch}>
        {children}
      </CountDispatchContext.Provider>
    </CountStateContext.Provider>
  );
}
```

---

## 5. Context Defaults — When Non-Undefined Default Is OK

The `undefined` default is the right choice for **required** providers (throw if missing).

A non-undefined default is OK for **optional** providers with sensible defaults:

```tsx
type LocaleContextValue = {
  locale: string;
  setLocale: (locale: string) => void;
};

// This default makes sense — the context works without a provider
const LocaleContext = createContext<LocaleContextValue>({
  locale: 'en-US',
  setLocale: () => {}, // no-op — acceptable for a default
});

function useLocale(): LocaleContextValue {
  return useContext(LocaleContext);
}
```

**Rule:** Use a real default (not `undefined`) only when the context has sensible fallback behavior without a provider. Otherwise, use `undefined` + throwing hook.

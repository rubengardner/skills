# Specialist H: Testing React + TypeScript

## Scope
React Testing Library with TypeScript, typed queries, mocking hooks and modules, user-event, accessibility-first assertions.

---

## 1. Setup

```json
// package.json devDependencies
{
  "@testing-library/react": "^14",
  "@testing-library/user-event": "^14",
  "@testing-library/jest-dom": "^6",
  "vitest": "^1",
  "@vitejs/plugin-react": "^4"
}
```

```typescript
// vitest.setup.ts
import '@testing-library/jest-dom';
```

```typescript
// vite.config.ts
export default defineConfig({
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./vitest.setup.ts'],
  },
});
```

---

## 2. Basic Component Test

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';

describe('Button', () => {
  it('renders the label', () => {
    render(<Button label="Click me" />);
    expect(screen.getByRole('button', { name: 'Click me' })).toBeInTheDocument();
  });

  it('calls onClick when clicked', async () => {
    const handleClick = vi.fn();
    const user = userEvent.setup();

    render(<Button label="Submit" onClick={handleClick} />);
    await user.click(screen.getByRole('button', { name: 'Submit' }));

    expect(handleClick).toHaveBeenCalledOnce();
  });

  it('is disabled when disabled prop is true', () => {
    render(<Button label="Disabled" disabled />);
    expect(screen.getByRole('button', { name: 'Disabled' })).toBeDisabled();
  });
});
```

**Rules:**
- Always use `userEvent.setup()` (not the legacy `userEvent` direct calls) — it properly simulates events.
- Query by role first (`getByRole`), then by label text, then by test id. Never query by CSS class or implementation details.
- Use `await` with all `userEvent` interactions — they're async in v14.

---

## 3. Typing Custom Render Wrappers

When components need providers, create a typed render wrapper:

```tsx
import type { RenderOptions } from '@testing-library/react';
import { render } from '@testing-library/react';
import type { ReactElement } from 'react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { MemoryRouter } from 'react-router-dom';

type CustomRenderOptions = Omit<RenderOptions, 'wrapper'> & {
  initialRoute?: string;
};

function customRender(ui: ReactElement, options: CustomRenderOptions = {}) {
  const { initialRoute = '/', ...renderOptions } = options;

  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });

  function Wrapper({ children }: { children: ReactElement }) {
    return (
      <QueryClientProvider client={queryClient}>
        <MemoryRouter initialEntries={[initialRoute]}>
          {children}
        </MemoryRouter>
      </QueryClientProvider>
    );
  }

  return render(ui, { wrapper: Wrapper, ...renderOptions });
}

export { customRender as render };
export { screen, waitFor, within } from '@testing-library/react';
```

---

## 4. Testing Async State

```tsx
import { render, screen, waitFor } from '../test-utils';
import { UserProfile } from './UserProfile';

// Mock the module
vi.mock('../api/users', () => ({
  fetchUser: vi.fn(),
}));

import { fetchUser } from '../api/users';
const mockFetchUser = vi.mocked(fetchUser);

describe('UserProfile', () => {
  it('shows loading state initially', () => {
    mockFetchUser.mockResolvedValue(makeUser());
    render(<UserProfile userId="1" />);
    expect(screen.getByRole('status', { name: /loading/i })).toBeInTheDocument();
  });

  it('shows user data after loading', async () => {
    const user = makeUser({ name: 'Alice' });
    mockFetchUser.mockResolvedValue(user);

    render(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByText('Alice')).toBeInTheDocument();
    });
  });

  it('shows error state on failure', async () => {
    mockFetchUser.mockRejectedValue(new Error('Network error'));

    render(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByRole('alert')).toHaveTextContent('Network error');
    });
  });
});
```

---

## 5. Test Factories (Typed)

```tsx
// test/factories/user.ts
import type { User } from '../types';

function makeUser(overrides: Partial<User> = {}): User {
  return {
    id: 'user-1',
    name: 'Test User',
    email: 'test@example.com',
    role: 'user',
    createdAt: new Date('2024-01-01'),
    ...overrides,
  };
}

export { makeUser };
```

**Rule:** Always use factory functions for test data. Never inline raw objects — they're brittle when types change.

---

## 6. Testing Forms

```tsx
import { render, screen } from '../test-utils';
import userEvent from '@testing-library/user-event';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('submits with valid data', async () => {
    const onSubmit = vi.fn();
    const user = userEvent.setup();

    render(<LoginForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText('Email'), 'alice@example.com');
    await user.type(screen.getByLabelText('Password'), 'securepassword');
    await user.click(screen.getByRole('button', { name: 'Log in' }));

    expect(onSubmit).toHaveBeenCalledWith({
      email: 'alice@example.com',
      password: 'securepassword',
      rememberMe: false,
    });
  });

  it('shows validation errors for invalid email', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={vi.fn()} />);

    await user.type(screen.getByLabelText('Email'), 'not-an-email');
    await user.click(screen.getByRole('button', { name: 'Log in' }));

    expect(await screen.findByText('Invalid email address')).toBeInTheDocument();
  });
});
```

---

## 7. Mocking Hooks

```tsx
// Mock a custom hook
vi.mock('../hooks/useUser', () => ({
  useUser: vi.fn(),
}));

import { useUser } from '../hooks/useUser';
const mockUseUser = vi.mocked(useUser);

describe('UserGreeting', () => {
  it('shows user name when authenticated', () => {
    mockUseUser.mockReturnValue({
      user: makeUser({ name: 'Bob' }),
      loading: false,
      error: null,
      refetch: vi.fn(),
    });

    render(<UserGreeting />);
    expect(screen.getByText('Hello, Bob')).toBeInTheDocument();
  });

  it('shows loading indicator', () => {
    mockUseUser.mockReturnValue({
      user: null,
      loading: true,
      error: null,
      refetch: vi.fn(),
    });

    render(<UserGreeting />);
    expect(screen.getByRole('status')).toBeInTheDocument();
  });
});
```

---

## 8. Accessibility-First Query Priority

Use queries in this order (most to least preferred):

1. `getByRole` — semantic HTML roles (button, textbox, heading, etc.)
2. `getByLabelText` — for form fields
3. `getByPlaceholderText` — fallback for unlabeled inputs
4. `getByText` — for non-interactive text
5. `getByDisplayValue` — for select/input current value
6. `getByAltText` — for images
7. `getByTitle` — last resort
8. `getByTestId` — only when nothing else works, use `data-testid`

**Rule:** If you can't find an element by role or label, the component may have an accessibility problem. Fix the component, not the query.

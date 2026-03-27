# React + TypeScript — Cursor Rules

You are an expert React and TypeScript developer building production web applications.

## Project Structure

```
src/
  components/       # Reusable UI components (Button, Modal, DataTable)
  features/         # Feature modules (auth/, dashboard/, billing/)
    auth/
      components/   # Feature-specific components
      hooks/        # Feature-specific hooks
      api.ts        # API calls for this feature
      types.ts      # Types for this feature
  hooks/            # Shared custom hooks
  lib/              # Utilities, constants, helpers
  types/            # Shared type definitions
  api/              # API client and request helpers
```

Each feature is self-contained. Never import from one feature into another — extract to shared first.

## Component Patterns

- Use function components exclusively. Never use class components.
- Export components as named exports, never default exports: `export function UserCard() {}`.
- Props interfaces are named `{Component}Props`: `interface UserCardProps {}`.
- Colocate props interface directly above the component, not in a separate file.
- Destructure props in the function signature: `export function UserCard({ name, email }: UserCardProps) {}`.
- Children must be explicitly typed: `children: React.ReactNode`, never `React.FC`.
- Never use `React.FC` or `React.FunctionComponent` — they add implicit children and complicate generics.

## Hooks

- Custom hooks always start with `use` and return a clearly typed object or tuple.
- Prefer returning objects for hooks with 3+ return values: `return { data, error, isLoading }`.
- Never call hooks conditionally. Extract conditional logic inside the hook body.
- `useMemo`: only when computation genuinely costs >1ms or when stabilizing a reference passed to a memoized child. Do not wrap every derived value.
- `useCallback`: only when passing callbacks to memoized children (`React.memo`) or as dependencies of other hooks. Wrapping every handler is noise.
- `useEffect` cleanup: every subscription, timer, or listener in useEffect MUST return a cleanup function.
- Never use `useEffect` to sync derived state. Compute it inline or with `useMemo`.

## Anti-Patterns to Avoid

BAD: `useEffect` to derive state from props
```tsx
// WRONG
const [fullName, setFullName] = useState('');
useEffect(() => { setFullName(`${first} ${last}`); }, [first, last]);
```
GOOD: Compute inline
```tsx
const fullName = `${first} ${last}`;
```

BAD: Boolean state pairs
```tsx
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError] = useState(false);
const [data, setData] = useState(null);
```
GOOD: Discriminated union state
```tsx
type State<T> = { status: 'idle' } | { status: 'loading' } | { status: 'error'; error: Error } | { status: 'success'; data: T };
```

BAD: Prop drilling through 3+ levels
GOOD: Use React Context for cross-cutting concerns, or pass components as children/slots.

## TypeScript Rules

- Enable `strict: true` in tsconfig. No exceptions.
- Never use `any`. Use `unknown` and narrow with type guards. If you must bypass, use `as` with a comment explaining why.
- Prefer `interface` for object shapes that may be extended. Use `type` for unions, intersections, and mapped types.
- Discriminated unions over optional fields for state that has mutually exclusive shapes.
- Generic components: `function List<T extends { id: string }>({ items }: { items: T[] })`.
- API responses get their own types in `types.ts`, separate from UI component props.
- Use `satisfies` for type-checking object literals while preserving narrow types: `const config = { ... } satisfies Config`.
- Avoid enums. Use `as const` objects: `const Status = { Active: 'active', Inactive: 'inactive' } as const`.

## Import Order

Enforce this order with a blank line between groups:

```tsx
// 1. React and framework imports
import { useState, useCallback } from 'react';
import { useRouter } from 'next/navigation';

// 2. Third-party libraries
import { z } from 'zod';
import { clsx } from 'clsx';

// 3. Internal aliases (@/ paths) — shared modules
import { Button } from '@/components/Button';
import { useAuth } from '@/hooks/useAuth';

// 4. Relative imports — feature-local modules
import { UserAvatar } from './UserAvatar';
import type { UserCardProps } from './types';

// 5. Type-only imports last (when not colocated)
import type { User } from '@/types';
```

## Error Handling

- API errors: wrap in a typed `Result<T, E>` pattern or use React Query / SWR error states. Never swallow errors silently.
- Render errors: every route-level component must have an Error Boundary. Use `react-error-boundary` package.
- Form validation: use `zod` schemas for both client and server validation. Define the schema once, derive the TypeScript type with `z.infer<typeof schema>`.
- Never use `try/catch` around synchronous React rendering logic — it does not catch render errors.

## Testing

- Framework: Vitest + React Testing Library.
- Test file location: colocated `ComponentName.test.tsx` next to the component.
- Test behavior, not implementation. Query by role, label, or text — never by CSS class or test ID unless no semantic alternative exists.
- Mock API calls at the network level with `msw` (Mock Service Worker), not by mocking fetch/axios.
- Never test that `useState` was called or assert on internal state. Assert on what the user sees.
- Each test must be independent. No shared mutable state between tests. Use `beforeEach` for setup.
- Snapshot tests: avoid for components. They break on every style change and teach nothing.

## Performance

- Lazy-load routes with `React.lazy()` and wrap in `Suspense` with a meaningful fallback.
- Images: always specify width and height, use `loading="lazy"` for below-fold images.
- Lists with 100+ items: use virtualization (`@tanstack/react-virtual`).
- Never create new object/array literals in JSX props: `style={{ color: 'red' }}` in a render loop kills performance. Hoist to a constant.
- Bundle analysis: aim for <200KB initial JS (gzipped). Split vendor chunks.

## Security

- Sanitize all user-generated content rendered as HTML. Prefer React's default JSX escaping.
- Never use `dangerouslySetInnerHTML` without sanitizing with DOMPurify first.
- Auth tokens: store in httpOnly cookies, never localStorage.
- Validate all URL parameters and query strings with `zod` before using them.
- CSP headers: configure in the server/middleware, not in meta tags.

## Styling

- Use Tailwind CSS utility classes as the primary styling method.
- Component variants: use `cva` (class-variance-authority) for type-safe variant props.
- Combine classes with `clsx` or `cn` (tailwind-merge + clsx).
- No inline `style` props except for truly dynamic values (e.g., positioning from calculations).
- Design tokens: define in `tailwind.config.ts` under `theme.extend`, never as CSS variables unless needed for runtime theming.

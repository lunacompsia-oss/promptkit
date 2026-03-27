# [Project Name] — Frontend Application

## Project Overview

<!-- Replace this block with your project description -->
[One paragraph describing what this app does, who uses it, and the core user experience goal.]

## Tech Stack

- **Framework:** [React 18+ / Vue 3 / Svelte 5] with TypeScript
- **Build:** Vite
- **Styling:** Tailwind CSS 4 + [shadcn/ui / Radix / Headless UI]
- **State:** Zustand (client) + TanStack Query (server state)
- **Routing:** [React Router v7 / TanStack Router / Vue Router / SvelteKit]
- **Forms:** React Hook Form + Zod validation
- **Testing:** Vitest + Testing Library + Playwright (E2E)
- **Hosting:** [Vercel / Cloudflare Pages / Netlify]

## File Structure

```
src/
  app/                    # App shell, providers, router config
  pages/                  # Route-level components (one per route)
    dashboard/
      DashboardPage.tsx
      DashboardPage.test.tsx
    settings/
      SettingsPage.tsx
  components/
    ui/                   # Design system primitives (Button, Input, Badge, Modal)
    features/             # Domain-specific (InvoiceTable, UserAvatar, PlanCard)
    layouts/              # Page shells (AppLayout, AuthLayout, OnboardingLayout)
  hooks/
    useDebounce.ts        # One hook per file
    useMediaQuery.ts
    useAuth.ts
  lib/
    api.ts                # API client — typed fetch wrapper
    cn.ts                 # Class name merge utility (clsx + twMerge)
    format.ts             # Date, currency, relative time formatters
    constants.ts          # App-wide constants
    storage.ts            # localStorage wrapper with JSON parse/stringify
  stores/
    auth.ts               # Zustand stores — one per domain
    ui.ts                 # Sidebar state, theme, modals
  types/
    api.ts                # API response types (mirror backend schemas)
    models.ts             # Domain model types
  assets/
    icons/                # SVG components or sprite
    images/               # Static images
```

**Rules:**
- One component per file. File name matches the export: `UserCard.tsx` exports `UserCard`.
- Co-locate tests: `UserCard.tsx` and `UserCard.test.tsx` in the same directory.
- No barrel files (`index.ts` re-exports). Import from the actual file path.
- `pages/` components are thin orchestrators. Business logic lives in hooks and services.
- `ui/` components are generic and reusable. They never import from `features/` or `stores/`.

## Code Style and Conventions

### TypeScript
- No `any`. Use `unknown` and narrow with type guards when the type is truly dynamic.
- Prefer `interface` for component props. Use `type` for unions and computed types.
- Export types from the file that owns them. Co-locate types with usage.
- Use `as const` for string literal arrays (route paths, status enums).
- Discriminated unions for state: `{ status: 'loading' } | { status: 'error'; error: Error } | { status: 'success'; data: T }`.

### Component Patterns
```tsx
// Good: clear, scannable, early returns for edge cases
interface UserCardProps {
  user: User;
  onEdit: (id: string) => void;
  variant?: 'compact' | 'full';
}

function UserCard({ user, onEdit, variant = 'full' }: UserCardProps) {
  if (!user.isActive) return <InactiveUserBanner userId={user.id} />;

  return (
    <div className="rounded-lg border p-4">
      <h3 className="text-sm font-medium">{user.name}</h3>
      {variant === 'full' && <p className="text-muted-foreground">{user.email}</p>}
      <Button size="sm" onClick={() => onEdit(user.id)}>Edit</Button>
    </div>
  );
}
```

- Props interface defined directly above the component. Not in a separate types file.
- Destructure props in the function signature with defaults.
- No `React.FC`. Use plain functions with explicit return types only when needed.
- Render logic reads top-to-bottom. Early returns for empty/error/loading states, then the main UI.

### State Management
- **Server state** (data from API): TanStack Query. Never store API data in Zustand.
- **Client state** (UI state): Zustand for global (sidebar, theme), `useState` for local.
- **Derived state**: compute it. If you can calculate it from existing state, do not store it.
- **URL state**: search params for filters, sorts, pagination. Use the URL as state.

```tsx
// Good: URL as state for shareable views
const [searchParams, setSearchParams] = useSearchParams();
const status = searchParams.get('status') ?? 'all';
const page = Number(searchParams.get('page') ?? '1');

// Bad: duplicating URL state in React state
const [status, setStatus] = useState('all');
const [page, setPage] = useState(1);
```

### Data Fetching
```tsx
// Standard pattern: query hook + loading/error states
function InvoicesPage() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['invoices', { status, page }],
    queryFn: () => api.invoices.list({ status, page }),
  });

  if (isLoading) return <InvoicesSkeleton />;
  if (error) return <ErrorState error={error} retry={() => refetch()} />;
  if (data.length === 0) return <EmptyState action={<CreateInvoiceButton />} />;

  return <InvoiceTable invoices={data} />;
}
```

- Every query has loading, error, empty, and success states handled.
- Use skeleton loaders, not spinners. Skeletons match the layout of the loaded content.
- Mutations invalidate related queries. `onSuccess: () => queryClient.invalidateQueries(['invoices'])`.
- Optimistic updates for instant-feel actions (toggling, reordering).

### Hooks Rules
- Custom hooks start with `use`. One hook per file.
- Hooks encapsulate logic, not just state. A hook should do one useful thing.
- No hooks that just wrap a single `useState`. That is not a useful abstraction.
- Hooks at the top of the component body. Never inside conditionals or loops.
- `useEffect` is a last resort. You probably want: a query, an event handler, or derived state.

## Styling with Tailwind

- Use Tailwind utility classes directly. No custom CSS unless Tailwind genuinely cannot do it.
- Use `cn()` helper (clsx + tailwind-merge) for conditional classes:
  ```tsx
  <div className={cn('rounded-lg p-4', isActive && 'ring-2 ring-primary', className)} />
  ```
- Component variants: use `cva` (class-variance-authority) for multi-variant components.
- Responsive: mobile-first. `sm:` for tablet, `lg:` for desktop. No breakpoint above `xl`.
- Dark mode via `dark:` prefix. Never hardcode colors. Use semantic tokens: `bg-background`, `text-foreground`, `text-muted-foreground`.
- No `style` prop. No inline styles. If you need a dynamic value, use a CSS variable.

## Accessibility (Non-Negotiable)

- Every interactive element is keyboard accessible. Tab order makes sense.
- Every image has `alt` text. Decorative images use `alt=""`.
- Form inputs have visible labels. Placeholder is not a label.
- Error messages are associated with inputs via `aria-describedby`.
- Focus management: after modal open, focus moves to modal. After close, focus returns to trigger.
- Color contrast: 4.5:1 for text, 3:1 for large text. Check with browser devtools.
- Use semantic HTML first: `button` not `div onClick`. `nav`, `main`, `section`, `aside`.
- Screen reader testing: visually hidden elements use `sr-only` class, not `display: none`.
- Interactive patterns follow WAI-ARIA patterns: dialog, tabs, combobox, menu.

```tsx
// Good: accessible dialog
<Dialog open={open} onOpenChange={setOpen}>
  <DialogTrigger asChild>
    <Button>Open Settings</Button>
  </DialogTrigger>
  <DialogContent>
    <DialogTitle>Settings</DialogTitle>
    <DialogDescription>Manage your account preferences.</DialogDescription>
    {/* form fields */}
  </DialogContent>
</Dialog>

// Bad: inaccessible modal
<div className={open ? 'block' : 'hidden'} onClick={close}>
  <h2>Settings</h2>
  {/* no focus trap, no aria, no escape key */}
</div>
```

## Error Handling

- **API errors**: catch in query `onError`. Show toast for recoverable, error page for fatal.
- **Render errors**: wrap route-level components in `ErrorBoundary`. Show a retry button.
- **Form errors**: display inline below the field. Scroll to first error on submit.
- **Network offline**: detect with `navigator.onLine` and show a persistent banner.
- Never show raw error messages from the API. Map error codes to user-friendly strings.
- Log errors to a service (Sentry) with component stack and user context.

## Testing

### Component Tests (Primary)
```tsx
describe('InvoiceTable', () => {
  it('renders invoice rows with correct amounts', () => {
    render(<InvoiceTable invoices={mockInvoices} />);

    expect(screen.getByText('INV-001')).toBeInTheDocument();
    expect(screen.getByText('$1,200.00')).toBeInTheDocument();
  });

  it('calls onSort when column header is clicked', async () => {
    const onSort = vi.fn();
    render(<InvoiceTable invoices={mockInvoices} onSort={onSort} />);

    await userEvent.click(screen.getByRole('columnheader', { name: /amount/i }));
    expect(onSort).toHaveBeenCalledWith('amount', 'asc');
  });

  it('shows empty state when no invoices', () => {
    render(<InvoiceTable invoices={[]} />);
    expect(screen.getByText(/no invoices/i)).toBeInTheDocument();
  });
});
```

- Test user behavior, not implementation. Click buttons, fill inputs, check text.
- Use `screen.getByRole` as the primary query. It enforces accessible markup.
- Mock API calls with MSW (Mock Service Worker), not by mocking fetch.
- Test all states: loading, success, empty, error, disabled.
- E2E with Playwright for critical user flows: sign up, create resource, payment.

### What Not to Test
- Styling (Tailwind classes). Visual regression tools do this better.
- Third-party component internals. Trust the library.
- Pure type-level code. TypeScript compiler tests this.

## Performance Guidelines

- **Bundle size**: lazy-load routes with `React.lazy`. Keep initial bundle under 200KB gzipped.
- **Images**: use `srcset` and `sizes` for responsive images. WebP/AVIF with fallback.
- **Lists**: virtualize lists over 50 items with `@tanstack/react-virtual`.
- **Rerenders**: split components at data boundaries. A `UserCard` with a `LastSeen` timer should not rerender the whole card.
- **Debounce**: search inputs, resize handlers, scroll handlers. 300ms default.
- **Prefetch**: prefetch data on hover/focus for links the user is likely to click.
- **Fonts**: use `font-display: swap`. Limit to 2 font families. Preload the primary font.
- **Third-party scripts**: load analytics/chat widgets after `onLoad`. Never block initial render.
- `useMemo`/`useCallback` only after measuring a problem. They have a cost too.

## Common Pitfalls

1. **Do not sync server state to local state.** Use TanStack Query. `useState(apiData)` is a bug factory.
2. **Do not use `useEffect` for derived values.** Compute during render: `const fullName = first + ' ' + last`.
3. **Do not pass data through 4+ levels of props.** Use context or Zustand. Prop drilling kills readability.
4. **Do not wrap every component in `React.memo`.** Measure first. Most rerenders are cheap.
5. **Do not create a `/utils` dumping ground.** Name files by what they do: `format.ts`, `validation.ts`, `storage.ts`.
6. **Do not use `useEffect` to respond to events.** Use event handlers. Effects are for synchronization, not reactions.
7. **Do not ignore TypeScript errors with `@ts-ignore`.** Fix the type. If you truly cannot, use `@ts-expect-error` with a comment explaining why.
8. **Do not import the entire icon library.** Import individual icons: `import { Check } from 'lucide-react'`.

## Git and PR Workflow

### Commit Messages
```
feat(dashboard): add revenue chart with date range picker
fix(auth): preserve redirect path after login
perf(table): virtualize invoice list for 1000+ rows
a11y(forms): add aria-describedby to all error messages
refactor(hooks): extract pagination logic to usePagination
```
Format: `type(scope): lowercase imperative description`
Types: `feat`, `fix`, `perf`, `a11y`, `refactor`, `test`, `chore`, `style`

### PR Checklist
- [ ] All interactive elements are keyboard accessible
- [ ] Loading, empty, and error states are handled
- [ ] No `any` types, no `@ts-ignore`
- [ ] New components have tests covering user interactions
- [ ] Responsive design works on 375px (mobile) through 1440px (desktop)
- [ ] Dark mode works (no hardcoded colors)
- [ ] No `console.log` in production code
- [ ] Bundle size impact checked (no accidental large dependency)
- [ ] Forms validate on submit and show inline errors
- [ ] Images have alt text, icons in buttons have aria-label

---
name: typescript-react
description: >
  TypeScript + React patterns for building typed, maintainable UI components: strict
  typing, component prop patterns, custom hooks, discriminated unions for state,
  avoiding 'any', and data fetching conventions. Based on react.dev TypeScript guide
  and Vanderkam's "Effective TypeScript" 2nd edition. Use for any task creating or
  modifying React components, hooks, or frontend state. Required by Dev Agents on
  TypeScript/React UI tasks.
applies-to: [dev-agent]
stack: typescript
layer: ui
version: 1.0
---

# TypeScript + React

<context>
This skill applies when writing React components and hooks in TypeScript. It covers
how to type props, state, events, refs, context, and custom hooks correctly. The
central principle: use the type system to make illegal states unrepresentable, reducing
runtime errors. Sources: react.dev/learn/typescript and Vanderkam — "Effective TypeScript" 2nd ed.
</context>

<rules>
## TypeScript Strictness
- `strict: true` MUST be enabled in `tsconfig.json`; partial strict options MUST NOT be used as a substitute.
- `any` MUST NOT be used; use `unknown` for values of unknown type and narrow with type guards.
- Type assertions (`as T`) MUST NOT be used to silence type errors; fix the type mismatch at the source.
- Non-null assertion (`!`) MUST NOT be used unless the non-null invariant is proven by context; use optional chaining or explicit guards.

## Component Props
- All component props MUST be typed with an explicit interface or type alias named `{ComponentName}Props`.
- Props interfaces MUST NOT use `React.FC<Props>` — declare the function directly with typed parameters.
- Children props MUST be typed as `React.ReactNode` (not `JSX.Element` or `any`).
- Event handler props MUST use the correct React event type: `React.MouseEvent<HTMLButtonElement>`, `React.ChangeEvent<HTMLInputElement>`.
- Optional props MUST use `?` suffix, not `prop: T | undefined` without a default.

## State
- State that has multiple mutually exclusive cases MUST be modeled as a discriminated union, not as multiple boolean flags.
- `useState` MUST be given an explicit type parameter when TypeScript cannot infer it: `useState<Order | null>(null)`.
- Derived values MUST NOT be stored in state — compute them during render.

## Hooks
- Custom hooks MUST be named with the `use` prefix.
- Custom hooks MUST return a typed object or tuple — not `any`.
- `useEffect` dependencies MUST be exhaustive; ESLint's `react-hooks/exhaustive-deps` rule MUST NOT be suppressed.
- `useCallback` and `useMemo` MUST be applied only when profiling proves a performance benefit — do not add preemptively.

## Data Fetching
- API responses MUST be typed; use `zod` or a type guard to validate the shape at runtime.
- Loading, error, and success states MUST be modeled as a discriminated union (not three separate booleans).
- Fetch calls MUST handle errors explicitly — never assume a fetch succeeded.

## Exports
- Components MUST be exported as named exports — not default exports — to support tree-shaking and consistent import naming.
- Types/interfaces MUST be exported with `export type` to enable `isolatedModules` compatibility.
</rules>

<patterns>
## Typed Component Props

```tsx
// Props interface — explicit, not React.FC
interface UserCardProps {
  name: string;
  email: string;
  role: "admin" | "member" | "guest";
  onEdit?: () => void;           // optional handler
  children?: React.ReactNode;    // typed children
}

export function UserCard({ name, email, role, onEdit, children }: UserCardProps) {
  return (
    <div>
      <h2>{name}</h2>
      <p>{email} · {role}</p>
      {onEdit && <button onClick={onEdit}>Edit</button>}
      {children}
    </div>
  );
}
```

## Discriminated Union for State (instead of boolean flags)

```tsx
// WRONG — three booleans; illegal states are representable (isLoading=true AND isError=true)
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError]     = useState(false);
const [data, setData]           = useState<Order | null>(null);

// CORRECT — discriminated union; illegal states are unrepresentable
type FetchState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: string };

const [state, setState] = useState<FetchState<Order>>({ status: "idle" });

// Usage
if (state.status === "success") {
  console.log(state.data.id); // TypeScript knows data exists here
}
```

## Custom Hook with Typed Return

```tsx
function useOrder(orderId: string): FetchState<Order> {
  const [state, setState] = useState<FetchState<Order>>({ status: "loading" });

  useEffect(() => {
    let cancelled = false;
    setState({ status: "loading" });

    fetchOrder(orderId)
      .then((order) => { if (!cancelled) setState({ status: "success", data: order }); })
      .catch((err) => { if (!cancelled) setState({ status: "error", error: err.message }); });

    return () => { cancelled = true; };
  }, [orderId]); // exhaustive deps

  return state;
}
```

## Typed Event Handlers

```tsx
interface SearchBarProps {
  onSearch: (query: string) => void;
}

export function SearchBar({ onSearch }: SearchBarProps) {
  const [value, setValue] = useState("");

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setValue(e.target.value);
  };

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    onSearch(value);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input value={value} onChange={handleChange} placeholder="Search..." />
      <button type="submit">Search</button>
    </form>
  );
}
```

## Runtime Validation with Zod

```tsx
import { z } from "zod";

const OrderSchema = z.object({
  id: z.string().uuid(),
  status: z.enum(["pending", "confirmed", "shipped", "cancelled"]),
  total: z.number().positive(),
  createdAt: z.string().datetime(),
});

type Order = z.infer<typeof OrderSchema>; // derive TS type from schema

async function fetchOrder(id: string): Promise<Order> {
  const res = await fetch(`/api/orders/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  const raw = await res.json();
  return OrderSchema.parse(raw); // throws ZodError if shape is wrong
}
```

## `unknown` instead of `any` for error handling

```tsx
// WRONG — any loses type safety
} catch (e: any) {
  setError(e.message); // compiles even if e has no .message
}

// CORRECT — unknown forces explicit narrowing
} catch (e: unknown) {
  const message = e instanceof Error ? e.message : "An unexpected error occurred.";
  setError(message);
}
```
</patterns>

<anti-patterns>
- **`React.FC<Props>`**: implicitly adds `children?: ReactNode` to every component, hides return type inference, and adds ceremony — declare functions directly.
- **Three boolean flags for async state**: `isLoading`, `isError`, `isSuccess` can all be `true` simultaneously — discriminated union makes this impossible.
- **`as T` to silence errors**: `const user = data as User` bypasses type checking — validate at runtime with Zod or add a proper type guard.
- **`any` for API responses**: `const data: any = await res.json()` — type errors in API contract changes are silently swallowed at compile time.
- **Storing derived values in state**: `const [total, setTotal] = useState(0)` when `total = lines.reduce(...)` — computed on each render from `lines`; storing it creates stale-state bugs.
- **Default exports for components**: `export default MyComponent` — import name can drift from component name; tree-shaking is less reliable; use named exports.
- **Missing `useEffect` cleanup**: fetch inside `useEffect` without cancellation — component unmounts during in-flight request; `setState` called on unmounted component.
- **`useMemo` and `useCallback` everywhere**: premature optimization; adds cognitive overhead for no measured benefit; profile before memoizing.
</anti-patterns>

<checklist>
- [ ] `"strict": true` in `tsconfig.json`.
- [ ] No `any` — use `unknown` and narrow with type guards.
- [ ] No `as T` type assertions to silence type errors.
- [ ] All component props typed with `{ComponentName}Props` interface.
- [ ] No `React.FC<Props>` — components declared as plain functions.
- [ ] Async UI state modeled as discriminated union (not boolean flags).
- [ ] `useEffect` dependencies are exhaustive — ESLint rule not suppressed.
- [ ] API responses validated at runtime with Zod or equivalent.
- [ ] Components exported as named exports.
- [ ] `catch (e: unknown)` with explicit narrowing — not `catch (e: any)`.
</checklist>

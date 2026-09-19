# rts — library conventions

Conventions for specific libraries. Apply a section only when the project uses
that library. These sections extend the core rules in [SKILL.md](./SKILL.md).
The core rules still apply (function style, `Result` errors, and so on).

Some sections tell you to **ask the user** first. Ask one time, when you first
add the library to the project. Then apply the answer everywhere, and do not ask
again.

## TanStack Query

### Manage query keys centrally

Do not scatter raw array keys (`["user", id]`) across the codebase. Raw keys
drift, and then cache reads stop matching cache writes.

When you first add TanStack Query to a project, **ask the user** how they want
to manage keys. Offer two options:

1. Install a key-management library (for example `@lukemorales/query-key-factory`).
2. Write a small typed key factory in the repo.

Then use the chosen factory everywhere.

```ts
// hand-written factory (or the equivalent from a library)
const userKeys = {
  all: ["users"] as const,
  lists: () => [...userKeys.all, "list"] as const,
  detail: (id: string) => [...userKeys.all, "detail", id] as const,
};
```

### Let the key and signal flow through `queryFn`

Do not close over a hoisted key variable inside `queryFn`. Read the key from the
context argument that TanStack Query gives to `queryFn`. Then the value that
drives the cache is the same value that the fetch uses. There is only one source
of truth.

Always forward `ctx.signal` to the network request. Then TanStack Query can
cancel in-flight requests (on unmount, refetch, or key change).

Wrap the options in `queryOptions()`. It types `ctx.queryKey` from the key
factory, and `useQuery`, `prefetchQuery`, and `getQueryData` all accept the
result.

```ts
import { queryOptions } from "@tanstack/react-query";

function userQuery(id: string) {
  return queryOptions({
    queryKey: userKeys.detail(id),
    queryFn: async (ctx) => {
      // the key comes from the context, not from a hoisted variable
      const [, , userId] = ctx.queryKey;
      const res = await fetch(`/users/${userId}`, { signal: ctx.signal });
      // queryFn is a throw boundary — see SKILL.md rule 7
      if (!res.ok) throw new Error("User not found");
      return res.json() as Promise<User>;
    },
  });
}

// the same options object serves every call site
const query = useQuery(userQuery(id));
await queryClient.prefetchQuery(userQuery(id));
```

### Invalidate through the key factory

After a mutation, invalidate with keys from the factory. Do not type the key by
hand. A broad key (`userKeys.all`) invalidates every query below it.

```ts
function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: updateUser,
    onSuccess: () =>
      queryClient.invalidateQueries({ queryKey: userKeys.all }),
  });
}
```

To render the query states (`pending`, `error`, `success`), see
[Match on query state](#match-on-query-state).

## Zustand

### Always use the `immer` middleware

Write updates as draft mutations through `immer`. This keeps the update
functions readable, and you do not have to write nested spreads by hand.
Prerequisite: install `immer`.

### Use the `persist` middleware for persistence

When state must survive a page reload, use the built-in `persist` middleware. Do
not write your own persistence code.

```ts
import { create } from "zustand";
import { immer } from "zustand/middleware/immer";
import { persist } from "zustand/middleware";

interface BearState {
  bears: number;
  addBear: () => void;
}

const useBearStore = create<BearState>()(
  persist(
    immer((set) => ({
      bears: 0,
      addBear: () =>
        set((state) => {
          state.bears += 1; // immer draft mutation
        }),
    })),
    { name: "bear-store" },
  ),
);
```

Before you add `persist`, **ask the user**:

- **Which keys must persist?** Persist only those keys. Use `partialize` to
  select them. Do not persist functions or temporary UI state (open menus,
  loading flags).
- **Which storage?** The default is `localStorage`. Use `sessionStorage` when
  the state must end with the browser tab.

When the shape of persisted state changes, increase `version` and add a
`migrate` function. Otherwise, users with old saved state get broken data.

```ts
{
  name: "bear-store",
  version: 2,
  partialize: (state) => ({ bears: state.bears }),
  migrate: (persisted, version) =>
    version < 2 ? { bears: 0 } : (persisted as BearState),
}
```

For server-side rendering (for example Next.js), the saved state is not
available on the server. Use `skipHydration: true`, and call
`useBearStore.persist.rehydrate()` on the client. This prevents a hydration
mismatch.

### Auto-selectors — ask first

When you set up Zustand in a project for the first time, **ask the user** if
they want auto-generated selectors. If yes, generate a `.use` accessor. Then the
code reads the store as `useBearStore.use.useSortedBears()`.

Keep the `use` prefix on every selector name. Each selector is a hook, and React
19 requires hook-style names. So you read a `bears` value with
`useBearStore.use.useBears()`, and a derived `sortedBears` value with
`useBearStore.use.useSortedBears()`.

```ts
// createSelectors, adapted to keep the `use` prefix on each selector
function createSelectors<S extends { getState: () => object }>(store: S) {
  const withUse = store as S & { use: Record<string, unknown> };
  withUse.use = {};
  for (const key of Object.keys(store.getState())) {
    const name = `use${key[0].toUpperCase()}${key.slice(1)}`;
    withUse.use[name] = () =>
      store((state: Record<string, unknown>) => state[key]);
  }
  return withUse;
}
```

The Zustand "Auto Generating Selectors" guide shows the base pattern that this
code adapts: https://zustand.docs.pmnd.rs/guides/auto-generating-selectors

## React Hook Form

Use the React Hook Form (RHF) API. Do not manage form state by hand.

### Bind fields through the UI library

When the form uses a UI library (shadcn/ui, Base UI, or similar), bind each
field through the field component of that library, together with the `control`
object from RHF. Do not spread `...register()` onto the styled inputs of the
library. The field component connects the value, `onChange`, the error state,
and accessibility for you. `register()` bypasses that connection and does not
follow the contract of the library.

Component names are different in each library (`<Field />`, `<FormField />`).
Check the form documentation of the library for the correct API.

```tsx
// ✗ spreading register onto a UI-library input
<Input {...register("email")} />

// ✓ bind through the library's field component + RHF control
<Field
  control={form.control}
  name="email"
  render={({ field }) => <Input {...field} />}
/>
```

The shadcn/ui form docs show a full integration:
https://ui.shadcn.com/docs/components/form

### Validate with a schema — ask first

When you add the first form, **ask the user** which schema library to use. Zod
is the common choice, and it connects to RHF through `@hookform/resolvers`.
Define the schema one time, and get the form type from it. Do not write the type
a second time by hand.

Always set `defaultValues`. Without them, inputs start as uncontrolled and React
shows a warning when they become controlled.

```tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const signUpSchema = z.object({
  email: z.string().email(),
  password: z.string().min(12),
});

type SignUpValues = z.infer<typeof signUpSchema>;

function SignUpForm() {
  const form = useForm<SignUpValues>({
    resolver: zodResolver(signUpSchema),
    defaultValues: { email: "", password: "" },
  });
  // ...
}
```

In a full-stack project, use the same schema on the server to validate the
request body.

## Dates

When a project needs date handling for the first time, **ask the user** if they
want to add a date library (`date-fns`) or use the built-in `Date`.

If the application must support time zones, recommend `date-fns`. Do not write
time-zone math with `Date` by hand. Manual time-zone code often causes
off-by-one and daylight-saving (DST) bugs. For time zones:

- **date-fns v4:** use `@date-fns/tz`.
- **date-fns v3:** use `date-fns-tz`.

Put parsing and formatting in one module. Do not scatter them across
components.

```ts
// lib/date.ts — the only place that parses or formats dates
import { format, parseISO } from "date-fns";

export function formatDisplayDate(iso: string) {
  return format(parseISO(iso), "d MMM yyyy");
}
```

Store and send dates as UTC ISO strings (`2026-09-19T08:30:00Z`). Convert to the
local time zone only when you display the date.

Follow-up question: when the app shows times, **ask the user** which time zone
to use — the time zone of the browser, or a time zone that each user saves in
their profile.

## Data handling: ts-belt and ts-pattern

rts prefers two libraries for data handling:

- **`@mobily/ts-belt`** — immutable, typed functions for arrays (`A`), objects
  (`D`), optional values (`O`), results (`R`), and composition (`pipe`).
- **`ts-pattern`** — pattern matching (`match`) for code that branches on many
  cases. An example is rendering a component from the state of a TanStack Query
  request.

These rules apply to frontend code and to backend code.

### Ask first

When the project first needs data transformation or branching logic, **ask the
user** if they want to install ts-belt and ts-pattern.

Pin the exact ts-belt version. Version 4 is a release candidate, so the API can
change between releases:

```bash
pnpm add @mobily/ts-belt@4.0.0-rc.5 ts-pattern
```

Because the version is a release candidate, the function names in this file can
be different from the installed version. Before you use a function, check the
type definitions in `node_modules/@mobily/ts-belt`.

**If the user says yes:**

- Use ts-belt for data transformation (arrays, objects, optional values,
  results).
- Use ts-pattern for branching on more than two cases.
- Do not add a second utility library (lodash, ramda, remeda) for the same work.

Then ask these follow-up questions:

1. **Existing code:** migrate the existing code, or apply the rules only to new
   code and to code that you change? The default is new code and changed code.
   Do not rewrite files that the task does not touch.
2. **Result type:** use the `R` (Result) module of ts-belt, or keep the
   hand-written `Result` union from SKILL.md rule 7? Use one of them everywhere.
   Do not mix the two.

**If the user says no:** the rule about ternaries (below) still applies. Use
early returns, a lookup object, or a `switch` statement that has an exhaustive
`never` check.

### Do not nest ternaries

Nested ternaries are hard to read. Never nest them. A single, flat ternary that
selects between two simple values is acceptable (see SKILL.md rule 18).

- For a value that can be missing, use the `O` (Option) module of ts-belt.
- For more than two cases, use `match` from ts-pattern.

```ts
// ✗ nested ternary
const city = user ? (user.address ? user.address.city : "Unknown") : "Unknown";

// ✓ ts-belt Option
const city = pipe(
  O.fromNullable(user),
  O.flatMap((value) => O.fromNullable(value.address)),
  O.map((address) => address.city),
  O.getWithDefault("Unknown"),
);
```

```ts
// ✗ nested ternary on a union
const color =
  status === "active" ? "green" : status === "pending" ? "yellow" : "red";

// ✓ ts-pattern — .exhaustive() fails to compile if you miss a case
const color = match(status)
  .with("active", () => "green" as const)
  .with("pending", () => "yellow" as const)
  .with("banned", () => "red" as const)
  .exhaustive();
// color: "green" | "yellow" | "red"
```

### Keep literal types with `as const`

ts-pattern and ts-belt infer return types from the callbacks. TypeScript widens
a literal that a callback returns: `() => "green"` returns `string`, not
`"green"`. Then the result of `match` is `string`, and the union is lost.

When the literal type is important, add `as const` to the returned value. The
same applies to ts-belt callbacks and default values, for example in `O.map`,
`O.getWithDefault`, and `A.map`.

```ts
// ✗ color: string
const color = match(status)
  .with("active", () => "green")
  .with("pending", () => "yellow")
  .exhaustive();

// ✓ color: "green" | "yellow"
const color = match(status)
  .with("active", () => "green" as const)
  .with("pending", () => "yellow" as const)
  .exhaustive();
```

In ts-pattern, `.returnType<Color>()` is an alternative. It sets the return type
one time, before the first `.with()`. A type annotation on the variable does not
help: the chain still returns `string`, and the assignment fails to compile.

### Match on query state

Use `match` to render each state of a TanStack Query request. Each branch gets
the narrowed type, so `data` exists only in the `success` branch.

```tsx
import { match } from "ts-pattern";

function UserProfile(props: UserProfileProps) {
  const query = useQuery(userQuery(props.id));

  return match(query)
    .with({ status: "pending" }, () => <UserProfileSkeleton />)
    .with({ status: "error" }, (result) => <ErrorMessage error={result.error} />)
    .with({ status: "success" }, (result) => <UserCard user={result.data} />)
    .exhaustive();
}
```

The same pattern works in the backend. For example, map a domain error to an
HTTP response at the protocol boundary (SKILL.md rule 7):

```ts
function toHttpError(error: AppError) {
  return match(error)
    .with({ kind: "not-found" }, () => ({ status: 404, message: "Not found" }))
    .with({ kind: "unauthorized" }, () => ({ status: 401, message: "Sign in" }))
    .with({ kind: "validation" }, (e) => ({ status: 422, message: e.message }))
    .exhaustive();
}
```

### Keep data immutable

ts-belt functions do not change their input. They return a new value. Apply the
same rule to the code of the project:

- Do not change function arguments. Do not use `push`, `splice`, or an in-place
  `sort` on data that you received.
- Mark array and object types as `readonly` where you can.
- Use `as const` for fixed data.

```ts
// ✗ changes the input
function addTag(post: Post, tag: string) {
  post.tags.push(tag);
  return post;
}

// ✓ returns a new value
function addTag(post: Post, tag: string) {
  return D.set(post, "tags", A.append(post.tags, tag));
}
```

Two exceptions:

- **immer drafts in Zustand.** A draft mutation is acceptable, because immer
  creates a new immutable state from it.
- **Hot paths.** For very large data, SKILL.md rule 4 permits in-place work on
  data that you own. Write a comment that tells why.

### Compose with `pipe`

When you apply more than one ts-belt function to a value, put all the steps in
one `pipe()`. Do not store each step in a separate variable. Inside `pipe`, use
the data-last form: give only the callback, not the data.

```ts
// ✗ one variable for each step
const active = A.filter(users, (user) => user.active);
const sorted = A.sortBy(active, (user) => user.name);
const names = A.map(sorted, (user) => user.name);

// ✓ one pipe, read from top to bottom
const names = pipe(
  users,
  A.filter((user) => user.active),
  A.sortBy((user) => user.name),
  A.map((user) => user.name),
);
```

For a single call, `pipe` is not necessary: `A.map(users, (user) => user.id)` is
correct.

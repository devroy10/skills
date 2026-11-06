---
name: react-query-patterns
description: Best practices and patterns for TanStack Query (React Query) v5 - custom query hooks, queryOptions, select, parameterized queries, pagination, prefetching, infinite queries, query key factories, mutations, automatic invalidation via mutation meta, global error handling, optimistic updates, and Suspense. Make sure to use this skill whenever the user writes, reviews, refactors, or debugs code using useQuery, useMutation, useInfiniteQuery, useSuspenseQuery, queryOptions, QueryClient, queryKey, invalidateQueries, or prefetchQuery, or asks how to structure data fetching, caching, pagination, or optimistic UI in React, even if they never mention "React Query" by name.
---

# React Query Patterns

Patterns for structuring data fetching with TanStack Query (React Query), based on the "Advanced React Query Patterns for Modern Applications" video by Youssef Benlemlih. Code targets **v5** (`isPending`, `gcTime`, `initialPageParam`, `useSuspenseQuery`). If the project is on v4, check the migration notes at the bottom before applying.

## How to use this skill

1. Look at what the user is doing (writing a new query, refactoring, adding pagination, mutations, etc.).
2. Apply the matching pattern below. Don't introduce every pattern at once; add structure only when the app needs it (e.g. key factories matter once there are several related queries).
3. When reviewing existing code, check against the **Review checklist** at the end.

---

## 1. Basic query

`useQuery` needs a **query key** (cache identity) and a **query function** (returns a promise). It returns the states needed to render the UI.

```tsx
const { data, isPending, isError, error, refetch } = useQuery({
  queryKey: ['contacts'],
  queryFn: fetchContacts,
});

if (isPending) return <Spinner />;
if (isError) return <ErrorAlert message={error.message} onRetry={refetch} />;
return <ContactsTable contacts={data.contacts} />;
```

- Render loading, error (with a retry button wired to `refetch`), then data.
- The query function must throw or reject on failure. With `fetch`, check `response.ok` and throw, otherwise HTTP errors look like successes.

## 2. Extract queries into custom hooks

Keep components readable and make the same query reusable.

```tsx
export function useContacts() {
  return useQuery({ queryKey: ['contacts'], queryFn: fetchContacts });
}

// component
const { data, isPending, isError } = useContacts();
```

## 3. `select` to transform data

Use `select` to derive what a component needs from the cached data. The selected value is memoized, so the component only re-renders when the **selected result** changes, not whenever unrelated parts of the data change.

```tsx
const { data: count } = useQuery({
  queryKey: ['contacts'],
  queryFn: fetchContacts,
  select: (data) => data.contacts.length,
});
```

Good for counts, filtered lists, and picking one field out of a large response. It matters most in large apps where re-renders add up.

## 4. `queryOptions` for flexible, shareable config

A custom hook locks callers into one shape (and makes custom `select` awkward to type). Prefer exporting **query options** and letting each component spread and extend them.

```ts
import { queryOptions } from '@tanstack/react-query';

export const contactsQueryOptions = () =>
  queryOptions({
    queryKey: ['contacts'],
    queryFn: fetchContacts,
  });
```

```tsx
// Top bar only needs the count
const { data: count } = useQuery({
  ...contactsQueryOptions(),
  select: (data) => data.contacts.length,
});
```

Benefits: type-safe overrides, one definition used by `useQuery`, `useSuspenseQuery`, `prefetchQuery`, `ensureQueryData`, and `setQueryData`.

## 5. Parameterized queries

When the query depends on input (an ID, filters, a page), pass the input to **both** the query function and the **query key**. The key is the dependency list: if a variable affects the data, it belongs in the key, so each value gets its own cache entry and the right data is fetched.

```ts
export const contactQueryOptions = (contactId: string) =>
  queryOptions({
    queryKey: ['contact', contactId],
    queryFn: () => fetchContact(contactId),
  });
```

Never leave an input out of the key. That causes stale or wrong data being served from the cache.

## 6. Pagination

Keep the current page in React state and pass it into a parameterized query. Changing the page changes the key, which triggers a fetch automatically.

```tsx
const [page, setPage] = useState(1);
const pageSize = 10;

const { data, isPending } = useQuery({
  queryKey: ['contacts', { page, pageSize }],
  queryFn: () => fetchContacts({ page, pageSize }),
  placeholderData: keepPreviousData, // keep showing the old page while the next loads
});

<button onClick={() => setPage((p) => p + 1)}>Next page</button>
```

`placeholderData: keepPreviousData` (import `keepPreviousData` from `@tanstack/react-query`) avoids the table flashing empty between pages.

## 7. Prefetching

Fetch likely-next data in the background so navigation is instant.

```tsx
const queryClient = useQueryClient();

useEffect(() => {
  queryClient.prefetchQuery({
    queryKey: ['contacts', { page: page + 1, pageSize }],
    queryFn: () => fetchContacts({ page: page + 1, pageSize }),
  });
}, [page, pageSize, queryClient]);
```

Can prefetch several pages ahead if the UX justifies it. Reuse `queryOptions` here so the prefetch key matches the real query key exactly.

## 8. Infinite queries

For infinite scroll or "Load more". React Query tracks the cursor internally.

```tsx
const {
  data,
  fetchNextPage,
  hasNextPage,
  isFetchingNextPage,
} = useInfiniteQuery({
  queryKey: ['contacts', 'infinite'],
  queryFn: ({ pageParam }) => fetchContacts({ cursor: pageParam }),
  initialPageParam: 0, // required in v5
  getNextPageParam: (lastPage) => lastPage.nextCursor, // return undefined when done
});

const contacts = data?.pages.flatMap((p) => p.contacts) ?? [];

<button
  onClick={() => fetchNextPage()}
  disabled={!hasNextPage || isFetchingNextPage}
>
  {isFetchingNextPage ? 'Loading...' : 'Load more'}
</button>
```

Use `isFetchingNextPage` and `fetchNextPage` to drive the button or an intersection observer.

## 9. Query key factories

Centralize keys in one object so queries, mutations, and invalidations all reference the same source and can't drift apart through typos.

```ts
export const contactKeys = {
  all: ['contacts'] as const,
  lists: () => [...contactKeys.all, 'list'] as const,
  list: (params: { page: number; pageSize: number }) =>
    [...contactKeys.lists(), params] as const,
  details: () => [...contactKeys.all, 'detail'] as const,
  detail: (id: string) => [...contactKeys.details(), id] as const,
};
```

Keys are hierarchical, so `invalidateQueries({ queryKey: contactKeys.all })` invalidates every contacts query, while `contactKeys.detail(id)` targets one.

---

## 10. Mutations

`useMutation` handles writes (create, update, delete). It exposes `mutate`, a pending state, and success/error callbacks, which suit notifications.

```tsx
const queryClient = useQueryClient();

const deleteContact = useMutation({
  mutationFn: (id: string) => api.deleteContact(id),
  onSuccess: () => {
    toast.success('Contact deleted');
    queryClient.invalidateQueries({ queryKey: contactKeys.all });
  },
  onError: (error) => toast.error(error.message),
});

<button onClick={() => deleteContact.mutate(id)} disabled={deleteContact.isPending}>
  Delete
</button>
```

After a successful mutation, **invalidate** the affected queries so they refetch fresh data.

## 11. Automatic invalidation via mutation `meta`

Repeating `invalidateQueries` in every mutation is noisy. Instead, tag mutations with `meta` and handle invalidation once, globally.

```ts
const queryClient = new QueryClient({
  mutationCache: new MutationCache({
    onSettled: (_data, _error, _variables, _context, mutation) => {
      const keys = mutation.meta?.invalidatesQuery as readonly unknown[] | undefined;
      if (keys) queryClient.invalidateQueries({ queryKey: keys });
    },
  }),
});
```

```tsx
useMutation({
  mutationFn: (id: string) => api.deleteContact(id),
  meta: { invalidatesQuery: contactKeys.all },
});
```

Note: the exact callback argument list on `MutationCache` has shifted slightly across v5 minor versions. Check the installed version's types and adjust where `mutation` appears.

## 12. Global error handling

Handle cross-cutting errors (for example 401 Unauthorized, so the user is logged out) once on the cache instead of in each mutation or query.

```ts
const queryClient = new QueryClient({
  mutationCache: new MutationCache({
    onError: (error) => {
      if (isHttpError(error) && error.status === 401) logout();
    },
  }),
  queryCache: new QueryCache({
    onError: (error) => {
      if (isHttpError(error) && error.status === 401) logout();
    },
  }),
});
```

Keep per-mutation `onError` for UI specific to that action (like a toast message).

## 13. Optimistic updates

Update the UI immediately, before the server confirms. Two approaches:

**UI-level**: render from the mutation's pending state (for example, filter out an item whose delete mutation is pending). Simple, local to one component.

**Cache-level**: edit the query cache directly, consistent across the whole app. Cancel in-flight queries, snapshot, update, roll back on error, then re-sync.

```tsx
useMutation({
  mutationFn: (id: string) => api.deleteContact(id),
  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: contactKeys.all });
    const previous = queryClient.getQueryData(contactsQueryOptions().queryKey);

    queryClient.setQueryData(contactsQueryOptions().queryKey, (old) =>
      old && { ...old, contacts: old.contacts.filter((c) => c.id !== id) },
    );
    return { previous };
  },
  onError: (_err, _id, context) => {
    queryClient.setQueryData(contactsQueryOptions().queryKey, context?.previous);
  },
  onSettled: () => queryClient.invalidateQueries({ queryKey: contactKeys.all }),
});
```

Always: cancel, snapshot, update, rollback on error, invalidate on settle.

## 14. Suspense for loading states

In query-heavy apps, per-component spinners cause flicker. Use `useSuspenseQuery` with React `Suspense` to show one coordinated fallback (such as a page skeleton).

```tsx
function Contacts() {
  const { data } = useSuspenseQuery(contactsQueryOptions()); // data is never undefined
  return <ContactsTable contacts={data.contacts} />;
}

<Suspense fallback={<PageSkeleton />}>
  <Contacts />
</Suspense>
```

- Loading is handled by the nearest `Suspense` boundary; errors by an error boundary.
- Place boundaries deliberately (app-level for a global skeleton, finer-grained where parts of a page should load independently).

---

## Review checklist

When reviewing or writing React Query code, check:

- [ ] Every variable that affects the data is in the `queryKey`
- [ ] Query function throws on HTTP errors (`response.ok` checked)
- [ ] Reusable queries are `queryOptions` (or a hook), not copy-pasted inline objects
- [ ] Keys come from a factory once there are several related queries
- [ ] Components that need a slice of data use `select`
- [ ] Mutations invalidate (or optimistically update) the affected queries
- [ ] Repeated invalidation and 401 handling are centralized, not duplicated per mutation
- [ ] Optimistic updates cancel queries first and roll back on error
- [ ] Pagination keeps previous data and prefetches the next page where it helps
- [ ] Infinite queries set `initialPageParam` and return `undefined` from `getNextPageParam` at the end

## v4 vs v5 quick notes

| v4 | v5 |
|---|---|
| `isLoading` for first load | `isPending` (and `isLoading` = pending and fetching) |
| `cacheTime` | `gcTime` |
| `keepPreviousData: true` | `placeholderData: keepPreviousData` |
| `getNextPageParam` only | also requires `initialPageParam` |
| `onSuccess` / `onError` on `useQuery` | removed; use `QueryCache` callbacks or effects |
| `useQuery({ suspense: true })` | `useSuspenseQuery` |

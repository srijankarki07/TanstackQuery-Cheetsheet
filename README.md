# THE COMPLETE TANSTACK QUERY CHEATSHEET  

### Why You Should Use TanStack Query (React Query)
| Old way (useEffect + fetch) | TanStack Query | Why it matters |
|-----------------------------|----------------|----------------|
| You manually track `loading`, `error`, `data` with useState | Automatic states | No more boilerplate |
| You write your own caching logic | Built-in smart cache | Same data → no extra network calls |
| Data becomes stale when tab is in background | Auto-refetches on window focus / reconnect | Users always see fresh data |
| Same API called 10 times from 10 components | Deduplicates requests | Saves battery & bandwidth |
| Background updates, offline support, pagination | All built-in | Feels instantly fast |

**Bottom line:** TanStack Query removes 90 % of the data-fetching code you used to write and makes your app feel magically fast and reliable.

### 1. Setup (Do this only once)
```bash
npm install @tanstack/react-query @tanstack/react-query-devtools
```

```tsx
// app/QueryProvider.tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";

const queryClient = new QueryClient();    // ← the brain that manages everything

export default function QueryProvider({ children }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />  {/* optional devtools */}
    </QueryClientProvider>
  );
}
```

**What’s happening?**  
We create a single `QueryClient` (the cache + rules engine) and wrap the whole app with `QueryClientProvider` so every component can talk to it.

### 2. useQuery – Reading Data (GET requests)

```tsx
const { data, isLoading, isError, isFetching, refetch } = useQuery({
  queryKey: ["todos", userId, page],           // ← unique identifier for this data
  queryFn: () => fetch(`/api/todos?user=${userId}&page=${page}`).then(r => r.json()),
  
  staleTime: 1000 * 60,          // data is "fresh" for 1 minute
  gcTime: 1000 * 60 * 30,        // keep in cache for 30 minutes even if unused
  refetchOnWindowFocus: true,    // auto-refresh when user returns to tab
});
```

**Key states explained**
| State          | When it’s true                                   | Typical UI use                |
|----------------|---------------------------------------------------|-------------------------------|
| `isLoading`    | First fetch ever (cache is empty)                | Show big skeleton / spinner   |
| `isFetching`   | Any fetch happening (including background ones) | Show small spinner in corner  |
| `isError`      | Request failed and retries exhausted             | Show error message            |
| `data`         | We have successful data                          | Render the actual list        |

**Why do we need `queryKey`?**  
It’s the DNA of your data. Same key → TanStack Query returns cached data instantly instead of hitting the server again.

### 3. useMutation – Changing Data (POST/PUT/DELETE)

```tsx
const mutation = useMutation({
  mutationFn: (newTodoTitle) => 
    axios.post("/api/todos", { title: newTodoTitle }),

  onSuccess: () => {
    // Tell all "todos" queries that data is now stale → they will refetch
    queryClient.invalidateQueries({ queryKey: ["todos"] });
  },
});
```

**Why do we need `invalidateQueries`?**  
After you create/update/delete something on the server, the old cached data is wrong. Invalidating forces a fresh fetch so the UI stays correct.

### 4. The 6 Most Important QueryClient Methods (you’ll use these daily)

| Method                              | What it actually does                                 | Typical situation                     |
|-------------------------------------|-------------------------------------------------------|---------------------------------------|
| `invalidateQueries(key)`            | Marks cache as stale → auto-refetch next time        | After mutation succeeds               |
| `setQueryData(key, newData)`        | Instantly updates cache without network               | Optimistic updates                    |
| `refetchQueries(key)`               | Forces immediate refetch right now                    | "Refresh" button                      |
| `cancelQueries(key)`                | Stops ongoing fetches                                 | In optimistic update rollback        |
| `getQueryData(key)`                 | Reads current cache (without triggering fetch)        | To rollback on error                  |
| `removeQueries(key)`                | Completely deletes from cache                         | Logout / clear sensitive data         |

### 5. Optimistic Updates – UI feels instant

```tsx
const mutation = useMutation({
  mutationFn: toggleTodo,
  onMutate: async (todoId) => {                // runs BEFORE request
    await queryClient.cancelQueries({ queryKey: ["todos"] });
    const previous = queryClient.getQueryData(["todos"]);

    // Immediately update UI
    queryClient.setQueryData(["todos"], old =>
      old.map(t => t.id === todoId ? { ...t, done: !t.done } : t)
    );
    return { previous };                       // for rollback if needed
  },
  onError: (err, variables, context) => {
    // Revert UI if server says no
    queryClient.setQueryData(["todos"], context.previous);
  },
  onSettled: () => queryClient.invalidateQueries({ queryKey: ["todos"] }),
});
```

**Why?** User clicks → checkbox flips instantly → server confirms later. Feels like desktop-app speed.

### 6. Pagination – No flash of old data

```tsx
useQuery({
  queryKey: ["users", page],
  queryFn: () => fetchUsers(page),
  placeholderData: keepPreviousData,   // ← keeps showing page 3 while loading page 4
});
```

**Why?** Prevents janky “jump” when changing pages.

### 7. Infinite Scroll – One hook does everything

```tsx
const {
  data,
  fetchNextPage,
  hasNextPage,
  isFetchingNextPage,
} = useInfiniteQuery({
  queryKey: ["posts"],
  queryFn: ({ pageParam = 1 }) => fetchPosts(pageParam),
  getNextPageParam: (lastPage) => lastPage.nextCursor,
});
```

**Why?** Handles loading state, deduplication, and pagination logic automatically.

### 8. Survive Page Refresh (optional but feels premium)

```ts
persistQueryClient({
  queryClient,
  persister: createSyncStoragePersister({ storage: window.localStorage }),
});
```

**Why?** User refreshes → data instantly appears from localStorage instead of showing loading skeletons.

### 9. Most Useful Options Cheat-Copy

```ts
useQuery({
  queryKey: ["key"],
  queryFn: fetchSomething,
  enabled: !!userId,              // don't run until we have userId
  staleTime: 1000 * 60 * 5,       // 5 min fresh
  gcTime: 1000 * 60 * 60 * 24,    // keep 24h
  refetchOnWindowFocus: true,
  placeholderData: keepPreviousData,
  select: data => data.results,   // transform before returning
});
```

### 10. One-Line Summary Table

| You want to…                | Just use…                                 |
|-----------------------------|-------------------------------------------|
| Show data                   | `useQuery`                                |
| Change data                 | `useMutation`                             |
| Instant UI (optimistic)     | `onMutate → setQueryData`                 |
| Safe but correct UI         | `invalidateQueries` on success            |
| Loading spinner first time  | `isLoading`                               |
| Background loading          | `isFetching`                              |
| Smooth pagination           | `placeholderData: keepPreviousData`       |
| Infinite scroll             | `useInfiniteQuery`                        |
| Survive refresh             | `persistQueryClient` + localStorage       |


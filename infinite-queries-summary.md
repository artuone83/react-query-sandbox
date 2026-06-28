# Infinite Queries in React Query

- [infinite queries docs](https://tanstack.com/query/latest/docs/framework/react/guides/infinite-queries)

## Background
- Infinite scroll was invented ~20 years ago by Aza Raskin, who later regretted it
- React Query's `useInfiniteQuery` hook makes it straightforward to implement

---

## `useInfiniteQuery` vs `useQuery`
- `useQuery` creates a separate cache entry per `queryKey`
- `useInfiniteQuery` uses **a single cache entry**, appending new data as it arrives
- Data is returned as a **multidimensional array** (`data.pages[]`) rather than a flat list — use `.flat()` if needed

---

## Key Configuration Options
- **`initialPageParam`** — the starting page value passed to `queryFn` on first call
- **`getNextPageParam(lastPage, allPages, lastPageParam)`** — derives the next page; return `undefined` to signal no more pages
- **`getPreviousPageParam(firstPage, allPages, firstPageParam)`** — for bidirectional fetching (e.g. deep-linked messaging apps)
- **Cursor-based APIs** — return the cursor from `getNextPageParam` instead of incrementing a number

---

## Fetching & UI Controls
- **`fetchNextPage()`** — triggers the next page fetch automatically
- **`isFetchingNextPage`** — `true` while the next page is loading
- **`hasNextPage`** — `true` if `getNextPageParam` returns a non-`undefined` value
- For **automatic** infinite scroll, trigger `fetchNextPage` via an intersection observer when the user reaches the bottom

---

## Refetching Behaviour
- On refetch, React Query **re-fetches all cached pages sequentially** (not just the current one)
- This guarantees **consistency** — partial refetches could cause duplicates or missing entries
- With many pages cached, this can be costly — mitigated with the **`maxPages`** option

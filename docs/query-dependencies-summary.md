# Managing Query Dependencies in React Query

## The Problem
- Real-world APIs often require dynamic parameters (e.g. GitHub's `sort` param: `created`, `updated`, `pushed`, `full_name`)
- Naively passing a parameter into `queryFn` while keeping a static `queryKey` **won't trigger a refetch** when the parameter changes — React Query does not re-run `queryFn` on every re-render

## The Wrong Fix
- Manually calling `refetch()` on change works but is **imperative** — React prefers declarative solutions

## The Correct Solution: Include Dependencies in `queryKey`
- **Whenever a value in `queryKey` changes, React Query automatically re-runs `queryFn`**
- Any value used inside `queryFn` must also appear in `queryKey`:
  ```js
  queryKey: ['repos', { sort }]
  ```

## How It Works Under the Hood
- `queryKey` maps directly to cache entries
- When `queryKey` changes, the observer **switches** to watching a different cache entry:
  - If the entry is **new** → starts in `pending`, fetches fresh data
  - If the entry **already exists** → instantly returns cached data (`success` state)

## Key Benefits
- **No race conditions** — handled automatically
- **Independent caching** per parameter set — different keys never overwrite each other
- **Instant switching** between previously fetched results
- Unlike `useEffect`'s dependency array, no need to worry about referential stability — arrays and objects are handled deterministically

## Avoiding Human Error
- Use **`@tanstack/eslint-plugin-query`** to catch missing `queryKey` dependencies automatically, with fix suggestions

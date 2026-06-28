# `useQuery` Hook – TanStack Query

## Signature
```ts
const result = useQuery({ ...options }, queryClient?)
```
Accepts an **options object** and an optional `queryClient` instance (for custom client injection).

---

## Returned State

### Data & Timing
- `data` – resolved query result
- `dataUpdatedAt` / `errorUpdatedAt` – timestamps of last successful/failed update

### Primary State Enums

These are the raw string values from which boolean flags are derived:

- `status` – describes whether data exists: `"pending"` | `"success"` | `"error"`
- `fetchStatus` – describes network activity independently of `status`: `"fetching"` | `"paused"` | `"idle"`

> **Why two separate states?** A query can be `status: "success"` (data exists) while simultaneously `fetchStatus: "fetching"` (silently refetching in the background). Separating them lets you handle UI concerns independently — e.g. show stale data while a refresh happens.

---

### Boolean Flags Derived from `status`

These are conveniences for `status === "..."` checks.

| Flag | Equivalent to | What it means in practice |
|---|---|---|
| `isPending` | `status === "pending"` | No data in cache yet. The query has never successfully resolved. Show a skeleton/spinner. |
| `isSuccess` | `status === "success"` | Data was fetched at least once and is available in `data`. Safe to render. |
| `isError` | `status === "error"` | The query failed and has no usable cached data. Show an error state. |

#### Specialised `status`-derived flags

| Flag | What it means in practice |
|---|---|
| `isLoading` | `isPending && isFetching` — query is pending *and* actively fetching right now. Narrower than `isPending` alone (which is also true when `enabled: false`). Use this to show a loading spinner. |
| `isInitialLoading` | Alias for `isLoading`. Kept for backwards compatibility. |
| `isLoadingError` | Failed on the very first fetch attempt — no data was ever retrieved. |
| `isRefetchError` | Failed during a *subsequent* fetch — stale data may still be available in `data`. Useful for showing a subtle error without wiping the screen. |

---

### Boolean Flags Derived from `fetchStatus`

These reflect real-time network activity, independent of whether data exists.

| Flag | Equivalent to | What it means in practice |
|---|---|---|
| `isFetching` | `fetchStatus === "fetching"` | A fetch is actively in-flight right now — including background refetches. Use for a subtle loading indicator when data is already shown. |
| `isPaused` | `fetchStatus === "paused"` | A fetch was attempted but is waiting (typically because the device is offline). The request will resume automatically when connectivity is restored. |

#### Specialised `fetchStatus`-derived flags

| Flag | What it means in practice |
|---|---|
| `isRefetching` | `isFetching && !isPending` — actively fetching but data already exists. Use to show a background refresh indicator without hiding existing content. |

---

### Other State Flags

| Flag | What it means in practice |
|---|---|
| `isFetched` | The query has completed at least once (success or error) in any session. |
| `isFetchedAfterMount` | The query has completed at least once since the component mounted. Useful for avoiding stale-data flickers on mount. |
| `isStale` | Cached data has exceeded `staleTime` and will be refetched on the next trigger (e.g. window focus). Does not trigger a fetch on its own. |
| `isPlaceholderData` | The value in `data` is coming from `placeholderData`, not a real fetch. Treat as a loading state — real data is still pending. |
| `isEnabled` | Whether the query is allowed to run. `false` when `enabled: false` is set. A disabled query never fetches automatically. |

### Errors & Retries
- `error` – error object if query failed
- `failureCount` / `failureReason` – retry diagnostics

### Other
- `promise` – the underlying fetch promise
- `refetch` – function to manually trigger a refetch

---

## Flags in Terms of the QueryClient Cache

### How the cache works (brief mental model)

The `QueryClient` holds a `QueryCache` — a flat map of `queryKey hash → Query instance`. Each `Query` instance is a small state machine that stores a `QueryState` object with these raw fields:

```
QueryState {
  data,              // the cached value, or undefined
  error,             // last error, or null
  status,            // "pending" | "success" | "error"
  fetchStatus,       // "idle" | "fetching" | "paused"
  dataUpdatedAt,     // timestamp of last successful fetch
  errorUpdatedAt,    // timestamp of last failure
  dataUpdateCount,   // how many times data was written
  errorUpdateCount,  // how many times an error was stored
  fetchFailureCount, // consecutive failures in the current retry chain
  isInvalidated,     // whether queryClient.invalidateQueries() has been called
}
```

`useQuery` subscribes a `QueryObserver` to that `Query` instance. Every flag you receive is a derived read of the above raw state. The component re-renders when the observer detects a change.

---

### What each flag tells you about the cache entry

| Flag | Cache condition it reflects |
|---|---|
| `isPending` | `QueryState.data === undefined` AND `status === "pending"`. The cache entry either does not exist yet, or exists but has never had a successful fetch write to it. |
| `isSuccess` | `status === "success"`. `data` has been written to the cache at least once and has not been evicted. |
| `isError` | `status === "error"` AND `data === undefined`. The cache entry exists but only holds an error — no usable data was ever written. |
| `isLoading` | `isPending && fetchStatus === "fetching"`. Cache has no data *and* the `queryFn` is actively running right now. |
| `isLoadingError` | `status === "error"` AND `dataUpdateCount === 0`. The cache write for data has never succeeded — the very first fetch attempt failed. |
| `isRefetchError` | `status === "error"` AND `dataUpdateCount > 0`. The cache has previously held valid data (written at least once), but the most recent fetch attempt failed and wrote an error instead. The stale `data` may still be readable. |
| `isFetching` | `fetchStatus === "fetching"`. The `queryFn` promise is currently in-flight. The cache entry is being actively written to (or attempted). |
| `isRefetching` | `isFetching && !isPending`. The cache already has data (`dataUpdateCount > 0`) and a new fetch is running to overwrite it. |
| `isPaused` | `fetchStatus === "paused"`. A fetch was triggered (e.g. by `networkMode: "online"` detecting offline), but execution is blocked. The cache entry is unchanged; the request is queued. |
| `isStale` | Either `dataUpdatedAt + staleTime < now` OR `isInvalidated === true` (set by `queryClient.invalidateQueries()`). The data in the cache is considered outdated. Does **not** trigger a fetch on its own — only the next refetch trigger will. |
| `isFetched` | `dataUpdateCount > 0 OR errorUpdateCount > 0`. The cache has been written to at least once — a fetch has fully completed (successfully or not). |
| `isFetchedAfterMount` | Same as `isFetched` but scoped to the current `QueryObserver` subscription lifetime. Resets to `false` each time the component mounts a new observer. |
| `isPlaceholderData` | The value in `data` did **not** come from a cache write by `queryFn`. It was injected via the `placeholderData` option and is ephemeral — invisible to other observers and never persisted in `QueryState.data`. |
| `isEnabled` | `enabled !== false` in query options. When `false`, the observer never calls `query.fetch()` and the cache entry is never populated automatically. `isPending` will remain `true` indefinitely for a disabled query with no prior cache entry. |

---

### Key cache lifecycle transitions

```
queryFn called
  → fetchStatus: "fetching"           (isFetching = true)

queryFn resolves
  → status: "success"
  → data written to QueryState
  → dataUpdateCount++                 (isFetched = true, isSuccess = true)
  → isInvalidated reset to false      (isStale determined by staleTime)

queryFn rejects (first time)
  → fetchFailureCount++               (retry chain begins)
  → after retries exhausted:
    → status: "error"                 (isError = true)
    → errorUpdateCount++              (isLoadingError = true if dataUpdateCount === 0)

queryClient.invalidateQueries(key)
  → isInvalidated = true              (isStale = true immediately)
  → triggers refetch if observer is mounted

gcTime exceeded with no observers
  → Query instance removed from QueryCache entirely
  → next useQuery call starts fresh (isPending = true again)
```

---

## Key Options

| Option | Purpose |
|---|---|
| `queryKey` | Cache key for the query |
| `queryFn` | Function that fetches data |
| `enabled` | Conditionally enable/disable query |
| `staleTime` / `gcTime` | Cache freshness & garbage collection timing |
| `refetchInterval` / `refetchOnMount` etc. | Auto-refetch behaviour |
| `retry` / `retryDelay` / `retryOnMount` | Retry configuration |
| `select` | Transform/select a subset of data |
| `placeholderData` | Show temporary data while loading |
| `initialData` / `initialDataUpdatedAt` | Pre-seed the cache |
| `networkMode` | Control fetch behaviour based on connectivity |
| `throwOnError` | Propagate errors to an Error Boundary |
| `notifyOnChangeProps` | Optimise re-renders by watching specific props |
| `structuralSharing` / `subscribed` | Advanced memoisation/subscription control |

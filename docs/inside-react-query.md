# Inside React Query – Architecture Summary

**Source:** [Inside React Query – TkDodo's Blog](https://tkdodo.eu/blog/inside-react-query)

## The QueryClient
- Created once at app startup; distributed via `QueryClientProvider` using React Context
- Doesn't cause re-renders — just provides access via `useQueryClient`
- Acts as a **container** for the `QueryCache` and `MutationCache`, plus global defaults

## QueryCache
- An **in-memory key-value store**: keys are serialized `queryKey` hashes, values are `Query` instances
- Data is not persisted — a page reload clears it (use persisters for external storage like `localStorage`)

## Query
- Where **most logic lives**: holds data, status, meta, retry, cancellation, and de-duplication
- Uses an **internal state machine** to prevent impossible states (e.g. de-duplicates concurrent fetches)
- Tracks which Observers are interested and notifies them of changes

## QueryObserver
- Created when `useQuery` is called; always subscribed to exactly **one query**
- Handles key **optimizations**: only notifies a component if properties it actually uses have changed (e.g. no re-render if only `isFetching` changes and the component only reads `data`)
- Hosts the `select` option, `staleTime`, and interval-fetching timers

## Active vs. Inactive Queries
- A query with no Observers is **inactive** — still cached but unused (shown greyed out in DevTools)

## Framework Agnosticism
- `QueryClient`, `QueryCache`, `Query`, and `QueryObserver` all live in the **framework-agnostic query core**
- Framework adapters (React, Solid, etc.) only need ~100 lines to create an Observer, subscribe to it, and trigger re-renders

## Component-Level Flow
1. Component mounts → `useQuery` creates an **Observer**
2. Observer subscribes to a **Query** in the `QueryCache` (creating it if needed)
3. Subscription may trigger a **background refetch** if data is stale
4. Fetch changes Query state → Observer is notified
5. Observer runs optimizations → notifies the **component** if relevant
6. Query completion → Observer informed again

> The key takeaway: almost all logic runs **outside React**, with the Observer acting as the bridge between the Query and the component.

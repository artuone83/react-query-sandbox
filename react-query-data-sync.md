# React Query: Data Synchronization & Cache Invalidation

## The Core Problem
- Server state changes constantly — cached client data quickly becomes outdated
- React Query acts as a **data synchronization tool**, keeping UI data as fresh as possible

---

## `staleTime` — The Key Concept
- **Fresh** = data served only from cache, no refetch triggered
- **Stale** = React Query will update the cache in the background when a trigger occurs
- `staleTime` defines how long (in ms) before a query is considered stale
- **Default is `0`** — every query is instantly stale ("aggressive but sane")

### Why 0?
- Fetching too often is the lesser evil vs. showing outdated data
- The right `staleTime` always **depends on the resource** — how often it changes, how accurate it needs to be, how collaborative the environment is

---

## Stale-While-Revalidate Strategy
- Stale data is **always delivered from cache instantly** — no waiting
- If stale, React Query **silently refetches in the background** and updates the cache
- Result: instant UI updates + eventual consistency with the server

---

## The 4 Refetch Triggers
1. **`queryKey` changes** — e.g. sort order changes
2. **New observer mounts** — a component using `useQuery` appears on screen
3. **Window regains focus** — user switches back to the app tab
4. **Device comes back online** — after a network interruption

---

## Customisation Options

| Goal | Approach |
|---|---|
| Reduce refetches | Increase `staleTime` |
| Disable specific triggers | Set `refetchOnMount`, `refetchOnWindowFocus`, `refetchOnReconnect` to `false` |
| Data that never changes | Set `staleTime: Infinity` |

---

## Key Takeaways
- Cached data is **always returned instantly**, regardless of freshness
- `staleTime` only controls **when background updates happen**, not whether cache is used
- Tuning `staleTime` per resource is preferable to disabling triggers entirely

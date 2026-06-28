# Managing Query Keys in React Query

## The Problem
- As apps grow, query keys get duplicated across hooks (e.g., in `useQuery` and `invalidateQueries`)
- Manual recreation of key arrays risks **typos and mismatches**

---

## Solution 1: Query Key Factories
- Define all keys in one object, imported wherever needed
- Eliminates typos and centralises key management
- Can use **composition** to build specific keys from generic ones:
  - `all → allLists → list(sort)`
- **Tradeoff:** composed keys are less readable
- **TypeScript tip:** add `as const` to each key for the narrowest inferred type

**Best practice:** one factory per feature, all keys sharing the same prefix

### Query Key Factories Example

```ts
const todoQueriesKeys = {
  all: () => ({ queryKey: ['todos'] }),
  allLists: () => ({
    queryKey: [...todoQueries.all().queryKey, 'list']
  }),
  list: (sort) => ({
    queryKey: [...todoQueries.allLists().queryKey, sort],
  }),
  allDetails: () => ({
    queryKey: [...todoQueries.all().queryKey, 'detail']
  }),
  detail: (id) => ({
    queryKey: [...todoQueries.allDetails().queryKey, id],
  }),
}
```

---

## Solution 2: Query Factories (next level)
- Combines Query Key Factories with the **options object pattern**
- Each entry returns both `queryKey` *and* `queryFn` (plus options like `staleTime`) together
- Keeps the inseparable `queryKey`/`queryFn` pair co-located
- Still allows per-call overrides by spreading: `{ ...todoQueries.list(sort), refetchInterval: … }`

**Gotcha:** entries have **mixed shapes** — some return arrays (hierarchy helpers like `allLists`), others return full query objects. This means:
- `invalidateQueries(todoQueries.allLists())` ❌ — needs wrapping in `{ queryKey: … }`
- `invalidateQueries(todoQueries.detail(id))` ✅ — extra properties are ignored by React Query

**TypeScript tip:** use the exported `queryOptions()` function for full type safety.

**Non-TypeScript alternative:** always return an object from every method to keep shapes consistent — more verbose, but avoids the mixed-shape pitfall.


### Query Factories Example

```ts
const todoQueries = {
  all: () => ({ queryKey: ['todos'] }),
  allLists: () => ({
    queryKey: [...todoQueries.all().queryKey, 'list']
  }),
  list: (sort) => ({
    queryKey: [...todoQueries.allLists().queryKey, sort],
    queryFn: () => fetchTodos(sort),
    staleTime: 5 * 1000,
  }),
  allDetails: () => ({
    queryKey: [...todoQueries.all().queryKey, 'detail']
  }),
  detail: (id) => ({
    queryKey: [...todoQueries.allDetails().queryKey, id],
    queryFn: () => fetchTodo(id),
    staleTime: 5 * 1000,
  }),
}
```

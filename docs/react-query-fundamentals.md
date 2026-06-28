# React Query – Fundamentals

  - [useQuery - docs](https://tanstack.com/query/latest/docs/framework/react/reference/useQuery)
---

## What is TanStack Query?


- TanStack Query is an `async state manager` that is acutely aware of the needs of server state. The way it works is, via useQuery, it fetches and caches data in the QueryCache and creates an Observer that listens to and notifies React of changes in that cache
- Created by **Tanner Linsley**, built on an **framework-agnostic core** (pure JavaScript, not coupled to React)
- Works with Vue, Solid, Svelte, and Angular — hence the `@tanstack` namespace
- Import from `@tanstack/react-query`, but commonly still referred to as "React Query"
- Knowledge is **easily portable** across frameworks

```ts
import { ... } from '@tanstack/react-query'
```

---

## QueryClient & QueryCache

- **`QueryClient`** is the heart of React Query — it contains and manages the **`QueryCache`**
- The `QueryCache` is the single location where **all fetched data lives**
- Under the hood, `QueryCache` is an **in-memory JavaScript `Map`**
- The `QueryClient` must be created **outside** of your root React component to keep the cache stable across re-renders

```ts
import { QueryClient } from '@tanstack/react-query'

const queryClient = new QueryClient(options)

export default function App() {
  // ...
}
```

---

## QueryClientProvider

- Since `QueryClient` lives outside of React, it needs to be **distributed** through the component tree
- Wrap your root component with `<QueryClientProvider>` and pass the `queryClient` as a `client` prop

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

const queryClient = new QueryClient()

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      {/* your app */}
    </QueryClientProvider>
  )
}
```

- Uses **React Context** under the hood — but only for **dependency injection**, not state management
- The `QueryClient` passed down is a **static object** that never changes → no unnecessary re-renders

---

## The `useQuery` Hook

The primary way to interact with the cache. It subscribes to the `QueryCache` and re-renders when the relevant data changes.

### Two required parameters

| Parameter | Purpose |
|-----------|---------|
| `queryKey` | Uniquely identifies the data in the cache |
| `queryFn`  | A function that returns a Promise resolving with the data to cache |

```ts
const { data } = useQuery({
  queryKey: ['luckyNumber'],
  queryFn: () => Promise.resolve(7),
})
```

### How it works

```
Is data already in cache at queryKey?
  ├── YES → return cached data immediately
  └── NO  → call queryFn → store result in cache at queryKey → return data
```

### Real-world example

```ts
const { data: pokemon, isLoading, error } = useQuery({
  queryKey: ['pokemon', id],
  queryFn: () =>
    fetch(`https://pokeapi.co/api/v2/pokemon/${id}`)
      .then(res => res.json()),
})
```

---

## Key Rules to Remember

1. **`queryKey` must be globally unique** — it acts as the key in the underlying `Map`
2. **`queryFn` must return a Promise** — in practice this is usually a `fetch` call, which returns a Promise by default
3. **Create `QueryClient` outside your root component** — ensures cache stability across re-renders
4. **Wrap your app in `QueryClientProvider`** — makes the cache accessible anywhere in the component tree

---

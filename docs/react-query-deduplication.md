# React Query: Deduplication & Observers

## What is Deduplication?

> the process of removing unnecessary duplicate information in computer systems

- Deduplication is react query context the process of **reusing cached values** when multiple components share the same `queryKey`.
- In general, when a `queryKey` is used, React Query first checks the global cache for a value, and if one exists, returns it immediately without running `queryFn`.

## The Core Concept: Deduplication
- When `useQuery` is called with a `queryKey`, React Query checks its **global cache** first
- If a value exists at that key → returns it **immediately**, skipping `queryFn` entirely
- If no value exists → runs `queryFn`, stores the result, then returns it
- This means multiple components using the **same `queryKey`** always get the same cached value, regardless of what `queryFn` would return on subsequent calls

## Key Implications
- The cache is **global** — deduplication works across different components, not just repeated calls in the same one
- Components don't need to be the same; they just need to share a `queryKey` under the same `QueryClientProvider`

## Custom Hooks
- Query logic can be abstracted into a **custom hook**, making async code feel synchronous and reusable
- Works identically in TypeScript and JavaScript — the return type (`UseQueryResult<number>`) is **inferred** from `queryFn`, no annotation needed

## How It Works Internally: Query Observers
- The cache lives **outside React**, so observers bridge the gap between cache and components
- Each `useQuery` call creates an **observer** that watches a specific `queryKey`
- When the cache value changes, the observer triggers a **re-render** to keep the UI in sync

## Benefits
- **Predictability** — every component reflects exactly what's in the cache
- **Performance** — `queryFn` only runs when necessary
- **Consistency** — all components under the same `QueryClientProvider` share one cache

## Code Example

```tsx
import * as React from "react"
import { QueryClient, QueryClientProvider, useQuery } from '@tanstack/react-query'

const queryClient = new QueryClient()

function useLuckyNumber() {
  return useQuery({
    queryKey: ['luckyNumber'],
    queryFn: () => {
      return Promise.resolve(Math.random())
    }
  })
}

function FortuneCookie() {
  const { data } = useLuckyNumber()

  if (data > 0.5) {
    return <div>Today's your lucky day</div>
  }

  return <div>Better stay home today</div>

}

function LuckyNumber() {
  const { data } = useLuckyNumber()

  return (
    <div>Lucky Number is: {data}</div>
  )
}

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <LuckyNumber />
      <FortuneCookie />
    </QueryClientProvider>
  )
}
```

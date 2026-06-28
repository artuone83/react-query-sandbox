# React Query: Query Lifecycle & Status Handling

## The Core Problem
- React Query makes async code *feel* synchronous, but data is still unavailable until the request resolves
- Accessing `data.map()` before the query completes throws: `Cannot read properties of undefined (reading 'map')`

## Three Query States

| State | Meaning |
|---|---|
| `pending` | Query not yet complete, no data available |
| `success` | Query finished, data is ready |
| `error` | Query failed |

These mirror native Promise states: pending → fulfilled → rejected.

## Two Ways to Access Status

### 1. `status` string property
```js
if (status === 'pending') { /* show loader */ }
if (status === 'error') { /* show error */ }
// safe to use data here
```

### 2. Derived boolean flags
```js
if (isPending === true) { /* show loader */ }
if (isError === true) { /* show error */ }
// safe to use data here
```

Both approaches are equivalent — choose one and stay consistent.

## TypeScript Bonus
- `useQuery` returns a **Discriminated Union Type** narrowed by `status` or boolean flags
- `data` is typed as `T | undefined` initially, but narrows to `T` once `pending` and `error` states are ruled out

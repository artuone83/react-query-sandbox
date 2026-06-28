# Fetching on Demand with React Query

## The Problem: Conditional Fetching
- By default, `useQuery` fetches immediately on component mount
- Sometimes you need to delay fetching (e.g. waiting for user input)
- Using `useQuery` inside an `if` statement **violates React's rules of hooks**

## Solution: The `enabled` Property
- Pass a boolean to `enabled` to control whether the query function runs
- Example: `enabled: search !== ''` — only fetches when a search term exists

---

## Pitfall 1: Misleading Loading State
- A `pending` status means *no data in cache* — **not** that the query is actively fetching
- Showing a loading spinner based solely on `pending` causes it to appear before the user types anything
- **Fix:** Also check `fetchStatus === 'fetching'` (or use the derived `isFetching`)
- React Query provides `isLoading` as shorthand for `status === 'pending' && fetchStatus === 'fetching'`

---

## Pitfall 2: Assuming Data Exists
- If `isLoading` is false and there's no error, you might assume data is available — **this is wrong**
- When `enabled` is false, the query is `pending` but **idle** (not fetching), so `isLoading` is false and `data` is `undefined`
- **Fix:** Always explicitly check `status === 'success'` before accessing data

## Recommended Component Structure
```jsx
if (isLoading)            → show spinner
if (status === 'error')   → show error
if (status === 'success') → render data
return                    → show prompt (e.g. "Enter a search term")
```

---

## TypeScript Note
- `enabled` doesn't narrow types — use `skipToken` from React Query instead of the query function when the parameter is `undefined`
- React Query will internally set `enabled: false` and TypeScript will correctly narrow the type

---

## Alternative Pattern: Conditional Component Rendering
- Instead of `enabled`, move query logic into a child component and **conditionally render that component**
- Pure React pattern — no React Query-specific flags needed
- Eliminates the need for `enabled`, `isLoading`, and `success` checks in the parent

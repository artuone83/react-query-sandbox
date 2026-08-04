# Queries, Caching, and Observers

## Observer Pattern

- References
  - [Observer Pattern](https://www.patterns.dev/vanilla/observer-pattern/)

### Core Idea

- A subject maintains a list of observers and broadcasts changes to them.
- Observers can attach/detach freely; the subject has no knowledge of specific consumers (decoupling)

### Minimal Implementation - the core shape

> [!IMPORTANT]
> At minimum, a subject needs three things
> - somewhere to keep observers
> - a way to add and remove them
> - and a way to push updates

```ts
class Subject {
  #observers = new Set();

  subscribe(observer) {
    this.#observers.add(observer);
    return () => this.#observers.delete(observer);
  }

  notify(payload) {
    for (const observer of this.#observers) {
      observer(payload);
    }
  }
}
```

### Example Use Case - Netflix Example

### Scenario

You subscribe to a show. When a new episode drops, Netflix notifies everyone
who's subscribed — it doesn't care who you are or what you do with the
notification.

### The Code

```typescript
type EpisodeHandler = (title: string) => void;

class Show {
  #subscribers = new Set<EpisodeHandler>();

  subscribe(fn: EpisodeHandler): () => void {
    this.#subscribers.add(fn);
    return () => this.#subscribers.delete(fn); // unsubscribe
  }

  releaseEpisode(title: string): void {
    for (const fn of this.#subscribers) {
      fn(title);
    }
  }
}
```

### Usage

```typescript
const strangerThings = new Show();

// Different "observers" reacting in their own way
const sendPushNotification: EpisodeHandler = (title) =>
  console.log(`📱 Push: New episode "${title}" is out!`);

const updateDownloadQueue: EpisodeHandler = (title) =>
  console.log(`⬇️ Auto-downloading "${title}" for offline viewing`);

const sendEmail: EpisodeHandler = (title) =>
  console.log(`📧 Email: "${title}" just dropped, come watch!`);

// Subscribe to the show
const unsub1 = strangerThings.subscribe(sendPushNotification);
const unsub2 = strangerThings.subscribe(updateDownloadQueue);
const unsub3 = strangerThings.subscribe(sendEmail);

// Netflix releases a new episode
strangerThings.releaseEpisode("Chapter 9: The Piggyback");

// Output:
// 📱 Push: New episode "Chapter 9: The Piggyback" is out!
// ⬇️ Auto-downloading "Chapter 9: The Piggyback" for offline viewing
// 📧 Email: "Chapter 9: The Piggyback" just dropped, come watch!

// unsubscribe example
unsub2(); // stop auto-downloading — e.g. user turned off downloads in settings
```

| Role              | In Netflix terms                                  |
| ----------------- | ------------------------------------------------- |
| **Subject**       | The `Show` (e.g., Stranger Things)                |
| **Observers**     | Push notifications, download queue, email service |
| **`subscribe()`** | You hitting the bell icon / following a show      |
| **`notify()`**    | Netflix publishing a new episode                  |

- The `Show` has **zero knowledge** of push notifications, emails, or downloads
  — it just knows "someone is listening."
- If you **unsubscribe** (stop following the show), you stop getting notified
  — but Netflix keeps releasing episodes regardless.
- Adding a new reaction (say, "notify your smart TV") doesn't require touching
  the `Show` class at all — just subscribe another function.

**The whole pattern in one sentence:** one thing happening, many independent
reactions, zero coupling between them.


## TanStack Query `useQuery` Observers Explained

One of the most important concepts in TanStack Query is the **Query Observer**. Once you understand observers, you'll understand why TanStack Query is so efficient.

Let's use a practical example.

---

### Scenario

We have three React components:

```tsx
<App>
  <UserProfile />
  <UserPosts />
  <UserSidebar />
</App>
```

All three components call exactly the same query:

```tsx
const userQuery = useQuery({
  queryKey: ["user", 1],
  queryFn: () =>
    fetch("https://jsonplaceholder.typicode.com/users/1")
      .then(res => res.json()),
})
```

---

#### What actually happens?

Many beginners think:

> Three components → three fetches

This is **not** what happens.

Instead:

```text
QueryClient
    │
    │
QueryCache
    │
    ▼
Query ["user", 1]
        │
        ├── Observer (UserProfile)
        ├── Observer (UserPosts)
        └── Observer (UserSidebar)
```

There is:

- **1 Query**
- **3 QueryObservers**
- **1 network request**

---

#### Step 1 — First component mounts

`UserProfile`

```tsx
function UserProfile() {
  const query = useQuery({
    queryKey: ["user", 1],
    queryFn: fetchUser,
  })

  return <div>{query.data?.name}</div>
}
```

Since the cache is empty:

```text
QueryCache

(empty)
```

TanStack Query creates:

```text
Query
key = ["user", 1]

Observers:
    UserProfile
```

The query immediately starts fetching.

```text
GET /users/1
```

---

#### Step 2 — Second component mounts

```tsx
function UserPosts() {
  const query = useQuery({
    queryKey: ["user", 1],
    queryFn: fetchUser,
  })
}
```

TanStack Query checks:

```text
Does query ["user", 1] already exist?

YES
```

Instead of creating another query, it simply adds another observer.

```text
Query

Observers

✓ UserProfile
✓ UserPosts
```

No new fetch occurs because one is already in progress.

---

#### Step 3 — Third component mounts

Same thing.

```text
Observers

✓ UserProfile
✓ UserPosts
✓ UserSidebar
```

Still only one fetch.

---

#### Request finishes

Suppose JSONPlaceholder returns:

```json
{
  "id": 1,
  "name": "Leanne Graham"
}
```

The Query stores:

```text
Query State

status: success

data:
{
  id: 1,
  name: "Leanne Graham"
}
```

Now the important part:

The Query notifies **every observer**.

```text
Query

      │
      ├────────► Observer 1
      ├────────► Observer 2
      └────────► Observer 3
```

Each observer causes its own component to re-render.

```text
UserProfile

renders

Leanne Graham

-------------------

UserPosts

renders

Leanne Graham

-------------------

UserSidebar

renders

Leanne Graham
```

Notice:

There was still only **one fetch**.

---

### Why observers exist

Each component can have different options.

Example:

```tsx
useQuery({
  queryKey: ["user", 1],
  queryFn: fetchUser,
  select: data => data.name,
})
```

Another component:

```tsx
useQuery({
  queryKey: ["user", 1],
  queryFn: fetchUser,
  select: data => data.email,
})
```

The cache stores:

```ts
{
  id: 1,
  name: "Leanne",
  email: "leanne@example.com",
}
```

Observer A receives:

```text
"Leanne"
```

Observer B receives:

```text
"leanne@example.com"
```

The Query stores the full object **once**, while each observer derives its own view.

---

### Different loading states

Suppose one component uses `placeholderData`:

```tsx
const query = useQuery({
  queryKey: ["user", 1],
  queryFn: fetchUser,
  placeholderData: {
    name: "Loading...",
  },
})
```

Another component:

```tsx
const query = useQuery({
  queryKey: ["user", 1],
  queryFn: fetchUser,
})
```

These are still observing the same Query.

But each observer exposes its own result.

Observer A:

```text
data:
Loading...
```

Observer B:

```text
data:
undefined
```

Same Query.

Different observer results.

---

### Refetch

Later:

```tsx
queryClient.invalidateQueries({
  queryKey: ["user", 1],
})
```

The Query becomes stale.

```text
Observers

Profile
Posts
Sidebar
```

The Query performs **one** refetch.

```text
GET /users/1
```

When it finishes:

```text
notify()

↓

Observer A

Observer B

Observer C
```

All three components update.

---

### Unmounting

Suppose:

```text
UserSidebar
```

is removed.

```text
Observers

✓ Profile
✓ Posts
```

Nothing happens to the Query because it still has active observers.

---

Now:

```text
UserPosts
```

unmounts.

```text
Observers

✓ Profile
```

Still active.

---

Finally:

```text
UserProfile
```

unmounts.

```text
Observers

(none)
```

The Query remains in the cache (for the configured `gcTime`, 5 minutes by default).

```text
Cache

Query
["user", 1]

Observers: 0
```

No component is using it anymore, but the cached data is retained.

---

### If another component mounts 30 seconds later

```tsx
useQuery({
  queryKey: ["user", 1],
  queryFn: fetchUser,
})
```

TanStack Query finds:

```text
Query exists

Observers = 0

Data exists
```

A new observer is attached:

```text
Observer

✓ NewComponent
```

If the data is still **fresh** (`staleTime` has not expired), it is returned immediately without a network request.

If the data is **stale**, the cached data is returned immediately while a background refetch is started.

---

### Mental model

Think of the architecture like this:

```text
               QueryClient
                    │
             QueryCache
                    │
        ┌────────────────────┐
        │ Query ["user", 1]  │
        │                    │
        │ data               │
        │ status             │
        │ fetch state        │
        └────────────────────┘
           ▲      ▲      ▲
           │      │      │
     Observer Observer Observer
        │         │         │
        ▼         ▼         ▼
   UserProfile UserPosts UserSidebar
```

---

### Key Takeaways

- A **Query** represents a unique `queryKey`.
- Every `useQuery()` call creates a **QueryObserver**.
- Multiple observers can subscribe to the same Query.
- Only **one network request** is made for a given Query, even if many observers are watching it.
- When the Query updates, it notifies all observers.
- Each observer can expose different derived data (`select`, `placeholderData`, etc.).
- When the last observer unmounts, the Query stays in the cache until `gcTime` expires.
- If a new observer subscribes before garbage collection, it reuses the existing cached Query.


## Cache - What exactly is `Cache`?

- `Cache` in the most basic form, is a piece of software that **STORES** data
- `Cache` enables **quicker** data access

## Mini React Query

- React Query is an async state manager
- Aware of the needs of the server state.
- It works is, via `useQuery`
- It fetches and caches data in the `QueryCache`
- Creates an `Observer` that listens to and notifies React of changes in that cache

### Building out the Query Client

- `QueryClient` contains and manages the `Cache`

#### Tanstack Query Example

```ts
import React from 'react'
import ReactDOM from 'react-dom/client'
import {
  QueryClient,
  QueryClientProvider,
  useQuery,
} from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'

const queryClient = new QueryClient()

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <ReactQueryDevtools />
      <Example />
    </QueryClientProvider>
  )
}

function Example() {
  const { isPending, error, data, isFetching } = useQuery({
    queryKey: ['repoData'],
    queryFn: async () => {
      const response = await fetch(
        'https://api.github.com/repos/TanStack/query',
      )
      return await response.json()
    },
  })

  if (isPending) return 'Loading...'

  if (error) return 'An error has occurred: ' + error.message

  return (
    <div>
      <h1>{data.full_name}</h1>
      <p>{data.description}</p>
      <strong>👀 {data.subscribers_count}</strong>{' '}
      <strong>✨ {data.stargazers_count}</strong>{' '}
      <strong>🍴 {data.forks_count}</strong>
      <div>{isFetching ? 'Updating...' : ''}</div>
    </div>
  )
}

const rootElement = document.getElementById('root')
if (!rootElement) throw new Error('Missing #root element')
ReactDOM.createRoot(rootElement).render(<App />)
```

#### Naive `QueryClient` Implementation

```ts
class QueryClient {
  constructor() {
    this.cache = new Map();
  }

  get(queryKey) {
    return this.cache.get(queryKey);
  }

  set(queryKey, data) {
    this.cache.set(queryKey, data);
  }
}

const queryClient = new QueryClient();

queryClient.get("mediaDevices"); // -> undefined

queryClient.set("mediaDevices", [{ deviceId: "id1", label: "label1" }]);
queryClient.get("mediaDevices"); // -> [{ deviceId: "id1", label: "label1" }]
```

> [!NOTE]
> This is a great start
> Now we need to figure out how to get actual data into our cache.
> Specifically, data that comes from an asynchronous request.

##### The request

- Our generic cache implementation shouldn't know how to create that Promise
- so let's let it accept a function that makes one, and store the result in our Map

###### `obtain` method

- With a queryKey and a queryFn parameters
- Stores the result of that async function in the cache.

```ts
class QueryClient {
  constructor() {
    this.cache = new Map();
  }

  get(queryKey) {
    return this.cache.get(queryKey);
  }

  set(queryKey, data) {
    this.cache.set(queryKey, data);
  }

  async obtain({ queryKey, queryFn }) {
    const data = await queryFn(queryKey);
    this.set(queryKey, data);
  }
}

// Usage

const queryClient = new QueryClient();

await queryClient.obtain({
  queryKey: "mediaDevices",
  queryFn: () => navigator.mediaDevices.enumerateDevices(),
});

queryClient.get("mediaDevices"); // -> [{ deviceId: "1", label: "label1"
```

- `obtain` looks a lot like the API for `useQuery`
- one exception, the `queryKey` is a string and not an array.
- Let's fix that.

##### The Hash of the queryKey

- To support `keys as arrays`, we need to create a `hash` of the `queryKey`,
- which we can do pretty easily by passing it to `JSON.stringify`

```ts
function hashKey(queryKey) {
  return JSON.stringify(queryKey);
}

class QueryClient {
  constructor() {
    this.cache = new Map();
  }

  get(queryKey) {
    const hash = hashKey(queryKey);
    return this.cache.get(hash);
  }

  set(queryKey, data) {
    const hash = hashKey(queryKey);
    this.cache.set(hash, data);
  }

  async obtain({ queryKey, queryFn }) {
    const data = await queryFn(queryKey);
    this.set(queryKey, data);
  }
}

const queryClient = new QueryClient();

await queryClient.obtain({
  queryKey: ["mediaDevices"],
  queryFn: () => navigator.mediaDevices.enumerateDevices(),
});
```

##### The Meta information about the query - status

We're making good progress, but there's still one big difference between how useQuery behaves and how we've implemented obtain so far.

When we invoke useQuery, we don't just get back the data that's in the cache, what we get back is an object that contains the data, but also contains other meta information about the query – the most important being the status of the query.

```ts
const { data, status, error, ...rest } = useQuery({
  queryKey,
  queryFn,
});
```

As you know, the `status` can be one of three values:

- `pending` - the Query has not yet completed, so you don't have data yet.
- `success` - the Query has finished successfully, and data is available.
- `error` - the Query has failed, and you have an error.

###### `pending` status

- We know that when the status is `pending`,
- That means there's `no data` in the cache – and vice versa,
- When there's `no data` in the cache, the status should be `pending`.
- Let's update our get method to reflect this logic.

```ts
get(queryKey) {
  const hash = hashKey(queryKey)

  if (!this.cache.has(hash)) {
    this.set(queryKey, {
      status: "pending"
    })
  }

  return this.cache.get(hash)
}
```

> [!NOTE]
> Now when we call get with a queryKey that doesn't exist in the cache, instead of undefined,
> we'll get back an object with the status set to pending.

###### `success` and `error` statuses

- If our queryFn resolves successfully,
- We'll store that data in the cache with the status set to success.
- If our queryFn rejects successfully `:p`,
- We'll set the status to `error` and store the error in the cache.

```ts
// FROM
async obtain({ queryKey, queryFn }) {
  const data = await queryFn(queryKey);
  this.set(queryKey, data);
}
// TO
async obtain({ queryKey, queryFn }) {
  try {
    const data = await queryFn(queryKey)

    this.set(queryKey, { status: "success", data })
  } catch (error) {
    this.set(queryKey, { status: "error", error })
  }
}
```

###### Update set method - Naive QueryClient

```ts
// FROM - always replacing the value in the cache
set(queryKey, data) {
  const hash = hashKey(queryKey)
  this.cache.set(hash, data)
}
// TO - merge the new data with the existing value
// update the status of the query, keeping the data intact
set(queryKey, query) {
  const hash = hashKey(queryKey)
  this.cache.set(hash, { ...this.cache.get(hash), ...query })
}
```

##### The React Integration

> [!CAUTION]
> How do we get data from our cache into a React component?
> How do we re-render the React component whenever the data changes?

###### The Observer

- The Observer needs a way to listen to updates that happen in the QueryCache
- To accomplish this, the QueryClient needs to maintain a list of listeners and notify them whenever the set function is called

```ts
function hashKey(queryKey) {
  return JSON.stringify(queryKey);
}

class QueryClient {
  constructor() {
    this.cache = new Map();
    this.listeners = new Set();
  }

  subscribe(listener) {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  get(queryKey) {
    const hash = hashKey(queryKey);

    if (!this.cache.has(hash)) {
      this.set(queryKey, {
        status: "pending",
      });
    }

    return this.cache.get(hash);
  }

  set(queryKey, query) {
    const hash = hashKey(queryKey);
    this.cache.set(hash, { ...this.cache.get(hash), ...query });
    this.listeners.forEach((listener) => listener(queryKey));
  }

  async obtain({ queryKey, queryFn }) {
    try {
      const data = await queryFn(queryKey);
      this.set(queryKey, { status: "success", data });
    } catch (error) {
      this.set(queryKey, { status: "error", error });
    }
  }
}
```

## Server Cache

We can see this in action whenever we use the API by looking at the response cache-control header

Response Headers

```bash
cache-control: public, max-age=60
```

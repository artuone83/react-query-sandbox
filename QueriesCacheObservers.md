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

# TanStack Query v5 + Next.js 16+ Server Components: The Complete Guide

## Table of Contents
1. [The Mental Model — Why Both Exist](#1-the-mental-model--why-both-exist)
2. [Project Setup](#2-project-setup)
3. [useQuery()](#3-usequery)
4. [useQueries()](#4-usequeries)
5. [useSuspenseQuery()](#5-usesuspensequery)
6. [usePrefetchQuery()](#6-useprefetchquery)
7. [queryClient.prefetchQuery()](#7-queryclientprefetchquery)
8. [useMutation / mutation.mutate / mutation.mutateAsync](#8-usemutation--mutationmutate--mutationmutateasync)
9. [queryClient.invalidateQueries()](#9-queryclientinvalidatequeries)
10. [HydrationBoundary](#10-hydrationboundary)
11. [useInfiniteQuery()](#11-useinfinitequery)
12. [Pagination with TanStack Query](#12-pagination-with-tanstack-query)
13. [Next.js Counterparts Comparison](#13-nextjs-counterparts-comparison)
14. [Scalable Architecture — Putting It All Together](#14-scalable-architecture--putting-it-all-together)
15. [Gotchas & Caveats](#15-gotchas--caveats)
16. [React Suspense — Server & Client Side in Next.js 16+](#16-react-suspense--server--client-side-in-nextjs-16)
17. [Deep Dive Q&A](#17-deep-dive-qa)
18. [Official Doc Links](#18-official-doc-links)

---

## 1. The Mental Model — Why Both Exist
### The Core Problem

Next.js 16+ Server Components can fetch data on the server with zero client JS. So why use TanStack Query at all?

**Next.js Server Components excel at:**
- Initial page load (SSR/SSG)
- SEO-critical content
- Data that doesn't change during a user session
- Reducing client bundle size

**TanStack Query excels at:**
- **Client-side interactivity** — data that changes based on user actions (filters, search, sorting)
- **Optimistic updates** — UI updates before the server confirms
- **Background refetching** — stale-while-revalidate pattern
- **Cache sharing** — multiple components reading the same cached data
- **Infinite scroll / pagination** — complex cursor-based data flows
- **Polling / real-time** — `refetchInterval` for live dashboards
- **Offline support** — cached data available without network

### The Hybrid Approach (What Production Apps Do)

```
Server Component (page.tsx)
  └── Fetches initial data on server
  └── Prefetches into QueryClient (dehydrates)
  └── Passes dehydrated state via HydrationBoundary
      └── Client Component
          └── useQuery() picks up prefetched data instantly (no loading spinner)
          └── Handles mutations, refetching, polling client-side
```

**The key insight:** Use Server Components to **seed** TanStack Query's cache so the user sees data instantly, then let TanStack Query manage the data lifecycle client-side.

---

## 2. Project Setup
### Installation

```bash
npm install @tanstack/react-query @tanstack/react-query-devtools
```

### The Provider Pattern (Critical for Next.js App Router)

**Why:** In App Router, layouts are Server Components by default. `QueryClientProvider` needs React Context (client-only). You need a Client Component wrapper.

**Gotcha #1 — DO NOT create QueryClient at module scope:**

```tsx
// ❌ WRONG — shared across requests on the server, causes data leaks between users
const queryClient = new QueryClient()

export function Providers({ children }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}
```

**Correct approach — create per-component instance with useState:**

```tsx
// src/providers/query-provider.tsx
'use client'

import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import { useState } from 'react'

export function QueryProvider({ children }: { children: React.ReactNode }) {
  // useState ensures ONE QueryClient per component lifecycle
  // It's lazy-initialized — the factory runs once on mount
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            // With SSR, we usually want to set some default staleTime
            // above 0 to avoid refetching immediately on the client
            staleTime: 60 * 1000, // 1 minute
          },
        },
      })
  )

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

```tsx
// src/app/layout.tsx (Server Component)
import { QueryProvider } from '@/providers/query-provider'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <QueryProvider>{children}</QueryProvider>
      </body>
    </html>
  )
}
```

> **TanStack Docs quote:** *"In Next.js specifically, the root layout is a Server Component, which means the QueryClientProvider can't be placed directly there. You need to create a separate client component for the provider."*
> — https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr

### Server-Side QueryClient (For Prefetching)

For prefetching in Server Components, you need a **separate** QueryClient that lives on the server:

```tsx
// src/lib/get-query-client.ts
import { QueryClient, defaultShouldDehydrateQuery, isServer } from '@tanstack/react-query'

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,
      },
      dehydrate: {
        // include pending queries in dehydration so the client
        // can show loading states that were started on the server
        shouldDehydrateQuery: (query) =>
          defaultShouldDehydrateQuery(query) || query.state.status === 'pending',
      },
    },
  })
}

let browserQueryClient: QueryClient | undefined = undefined

export function getQueryClient() {
  if (isServer) {
    // Server: always make a new query client
    // because each request should have its own client
    return makeQueryClient()
  } else {
    // Browser: reuse the same query client
    // so we don't lose our cache on re-renders
    if (!browserQueryClient) browserQueryClient = makeQueryClient()
    return browserQueryClient
  }
}
```

> **TanStack Docs quote:** *"On the server, we create a new QueryClient for every request to avoid sharing data between users. On the client, we reuse the same QueryClient across re-renders."*
> — https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr

---

## 3. useQuery()
### Why
The foundational hook. Declaratively fetch, cache, and synchronize server state with your UI. Eliminates manual `useState` + `useEffect` + `loading` + `error` boilerplate.

### What
Subscribes a component to a query. Returns `{ data, error, isLoading, isPending, isError, isFetching, refetch, ... }`.

### How

```tsx
'use client'

import { useQuery } from '@tanstack/react-query'

// API function — kept separate for reuse and testing
async function fetchTodos(userId: string): Promise<Todo[]> {
  const res = await fetch(`/api/todos?userId=${userId}`)
  if (!res.ok) throw new Error('Failed to fetch todos')
  return res.json()
}

export function TodoList({ userId }: { userId: string }) {
  const {
    data: todos,       // The resolved data (undefined until first fetch)
    isPending,         // true when there's no cached data and no query attempt has finished yet
    isLoading,         // isPending AND isFetching — true only on FIRST load
    isFetching,        // true whenever a request is in-flight (including background refetches)
    isError,           // true if the query encountered an error
    error,             // the Error object (null if no error)
    isSuccess,         // true if the query has data
    isStale,           // true if data is older than staleTime
    refetch,           // manually trigger a refetch
    isPlaceholderData, // true when showing placeholder data
  } = useQuery({
    queryKey: ['todos', userId],   // unique cache key — MUST include all variables the queryFn depends on
    queryFn: () => fetchTodos(userId),
    staleTime: 5 * 60 * 1000,     // data is "fresh" for 5 minutes (won't refetch)
    gcTime: 30 * 60 * 1000,       // garbage collect unused cache after 30 min (was "cacheTime" in v4)
    refetchOnWindowFocus: true,    // default: true — refetch when user tabs back
    refetchOnMount: true,          // default: true — refetch when component mounts (if stale)
    retry: 3,                      // retry failed requests 3 times (default)
    retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000), // exponential backoff
    enabled: !!userId,             // only fetch when userId exists (conditional fetching)
    placeholderData: [],           // show this while loading (keeps previous data on key change with `keepPreviousData`)
    select: (data) => data.filter(t => !t.completed), // transform/filter data (runs on cached data too)
  })

  if (isPending) return <div>Loading...</div>
  if (isError) return <div>Error: {error.message}</div>

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>{todo.title}</li>
      ))}
    </ul>
  )
}
```

### When
- Any time you need to read server data in a Client Component
- When data needs to stay synchronized (refetch on focus, on reconnect)
- When multiple components need the same data (shared cache key)

### Key Behaviors

| Option | Default | What It Does |
|--------|---------|-------------|
| `staleTime` | `0` | How long data is considered fresh. `0` = always stale (refetches on mount/focus) |
| `gcTime` | `5 min` | How long unused/inactive cache entries stay in memory |
| `refetchOnWindowFocus` | `true` | Refetch when browser tab regains focus |
| `refetchOnReconnect` | `true` | Refetch when network reconnects |
| `refetchOnMount` | `true` | Refetch when component mounts (if data is stale) |
| `retry` | `3` | Number of retries on failure |
| `enabled` | `true` | Set to `false` to disable automatic fetching |

### Next.js Counterpart

In a Server Component, you'd just `await` the data:

```tsx
// Server Component — no useQuery needed
export default async function TodoPage({ params }: { params: Promise<{ userId: string }> }) {
  const { userId } = await params
  const todos = await fetchTodos(userId) // Direct server fetch

  return <TodoList todos={todos} />
}
```

**But this has no client-side refetching, no cache sharing, no optimistic updates.** That's where the hybrid approach comes in (see Section 10 on HydrationBoundary).

> **Note — Do you always need `if(!data)` checks?**
>
> With `useQuery`, **yes**. `data` is typed as `T | undefined` — it's `undefined` until the first successful fetch. You must always guard: `if (isPending) return <Skeleton />` or `if (!data) return null`. There is no way around this with `useQuery`.
>
> If your component **can't render anything meaningful without the data** (e.g., a user profile, a data table, a product card), that's a sign you should use `useSuspenseQuery` instead. It guarantees `data: T` (never `undefined`) because the component simply doesn't render until data is ready — React Suspense shows the fallback instead. No `if(!data)` checks, no `isPending` guards.
>
> **Keep `useQuery`** when your component has a useful partial UI without data (e.g., a form shell, a layout with placeholders), or when you need `enabled: false` for conditional fetching. See the full comparison in Section 5.

---

## 4. useQueries()
### Why
When you need to run a **dynamic number** of queries in parallel. You can't call `useQuery()` in a loop (hooks rules).

### What
Accepts an array of query options and runs them all concurrently. Returns an array of query results.

### How

```tsx
'use client'

import { useQueries } from '@tanstack/react-query'

export function MultiUserDashboard({ userIds }: { userIds: string[] }) {
  const userQueries = useQueries({
    queries: userIds.map((id) => ({
      queryKey: ['user', id],
      queryFn: () => fetchUser(id),
      staleTime: 5 * 60 * 1000,
    })),
    // Optional: combine all results into a single value
    combine: (results) => {
      return {
        data: results.map((result) => result.data),
        pending: results.some((result) => result.isPending),
        errors: results.filter((result) => result.isError),
      }
    },
  })

  if (userQueries.pending) return <div>Loading users...</div>

  return (
    <div>
      {userQueries.data.map((user, i) => (
        <UserCard key={userIds[i]} user={user} />
      ))}
    </div>
  )
}
```

### When
- Dashboard showing data for N items (N is dynamic)
- Aggregating data from multiple endpoints
- When the number of queries depends on another query's result

### Gotcha — The `combine` callback
The `combine` function is **not** memoized by default. If you return a new object reference every time, it triggers re-renders. Use `useMemo` patterns or keep `combine` stable:

```tsx
// ❌ Creates new array reference every render
combine: (results) => ({
  data: results.map(r => r.data) // new array every time
})

// ✅ Only changes reference when data actually changes (TanStack handles this internally)
// The combine callback IS tracked for reference equality by TanStack Query v5
// so this is actually fine — TanStack memoizes the result
```

> **TanStack Docs quote:** *"The combine function will be called on every render, even if the query results haven't changed. The result will be structurally shared to avoid unnecessary re-renders."*

### Next.js Counterpart
In Server Components, just use `Promise.all`:

```tsx
export default async function Dashboard({ userIds }: { userIds: string[] }) {
  const users = await Promise.all(userIds.map(id => fetchUser(id)))
  return <div>{users.map(u => <UserCard key={u.id} user={u} />)}</div>
}
```

---

## 5. useSuspenseQuery()
### Why
Integrates with React Suspense so you can use `<Suspense>` boundaries for loading states instead of checking `isPending` manually. The data is **guaranteed to be defined** when the component renders.

### What
Same as `useQuery()` but:
- Throws a promise while loading (React Suspense catches it)
- Throws the error if the query fails (needs an Error Boundary)
- `data` is **never** `undefined` — TypeScript narrows this for you
- `enabled` option is **not available** (would break Suspense contract)
- `placeholderData` is **not available**

### How

```tsx
'use client'

import { useSuspenseQuery } from '@tanstack/react-query'
import { Suspense } from 'react'
import { ErrorBoundary } from 'react-error-boundary'

// The data-fetching component — notice no loading/error checks needed
function UserProfile({ userId }: { userId: string }) {
  const { data: user } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  })

  // `user` is guaranteed to be defined here — no undefined check needed!
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  )
}

// The parent wraps with Suspense + ErrorBoundary
export function UserProfilePage({ userId }: { userId: string }) {
  return (
    <ErrorBoundary fallback={<div>Something went wrong</div>}>
      <Suspense fallback={<UserProfileSkeleton />}>
        <UserProfile userId={userId} />
      </Suspense>
    </ErrorBoundary>
  )
}
```

### When
- When you want loading/error UI to be handled by boundaries (React's recommended pattern)
- When you need type-safe guaranteed data (no `| undefined`)
- When you're already using Suspense boundaries in your layout

### Gotcha — Suspense Waterfall

```tsx
// ❌ WATERFALL — second query waits for first
function UserAndPosts({ userId }: { userId: string }) {
  const { data: user } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  })
  // This doesn't start until the first suspense resolves!
  const { data: posts } = useSuspenseQuery({
    queryKey: ['posts', userId],
    queryFn: () => fetchPosts(userId),
  })

  return <div>{user.name}: {posts.length} posts</div>
}
```

> **TanStack Docs quote:** *"When using suspense, it's important to understand that each useSuspenseQuery will 'throw' a promise independently. This means if you have two useSuspenseQuery calls in the same component, they will waterfall — the second won't start until the first resolves."*

**Fix — use `useSuspenseQueries()` for parallel suspense:**

```tsx
import { useSuspenseQueries } from '@tanstack/react-query'

function UserAndPosts({ userId }: { userId: string }) {
  const [{ data: user }, { data: posts }] = useSuspenseQueries({
    queries: [
      { queryKey: ['user', userId], queryFn: () => fetchUser(userId) },
      { queryKey: ['posts', userId], queryFn: () => fetchPosts(userId) },
    ],
  })

  return <div>{user.name}: {posts.length} posts</div>
}
```

### Gotcha — What If You Don't Define a `<Suspense>` Boundary?

`useSuspenseQuery` throws a Promise. That Promise **bubbles up** the component tree looking for the nearest `<Suspense>` — exactly like errors bubble up looking for the nearest Error Boundary.

```
Your component tree:

<RootLayout>                     ← Next.js has an implicit Suspense here
  <Suspense fallback={...}>      ← from loading.tsx (if you have one)
    <DashboardLayout>
      <SomeParent>
        <YourComponent />        ← useSuspenseQuery throws Promise
                                   ↑ bubbles UP looking for <Suspense>
```

**What happens depends on what's above:**

| What's above your component | What the user sees during loading |
|---|---|
| Explicit `<Suspense fallback={<Skeleton />}>` you wrote | Your skeleton (intended behavior) |
| A `loading.tsx` several route segments up | That loading.tsx skeleton (probably wrong UI — e.g., a full-page spinner for a tiny widget) |
| Nothing — no `<Suspense>` anywhere | Next.js root implicit boundary → **blank white page** while loading |
| Plain React app (no Next.js, no Suspense) | **React throws an error** in development |

**Your app won't crash in Next.js** (the framework provides a root-level Suspense), but you lose control over what the user sees. Instead of a targeted skeleton for your component, the user might see:
- A `loading.tsx` from a parent route segment (unrelated skeleton)
- A blank page (root boundary has no meaningful fallback)

```tsx
// ❌ No Suspense boundary — Promise bubbles to who-knows-where
function Page() {
  return (
    <div>
      <h1>Dashboard</h1>
      <UserProfile />  {/* useSuspenseQuery inside — where does the Promise go? */}
    </div>
  )
}

// ✅ Explicit boundary — you control exactly what shows during loading
function Page() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<UserSkeleton />}>
        <UserProfile />
      </Suspense>
    </div>
  )
}
```

**Rule: Always pair `useSuspenseQuery` with an explicit `<Suspense>` boundary.** Don't rely on implicit/inherited boundaries — you'll get loading UI you didn't intend.

### Gotcha — Don't Put `<Suspense>` Inside the Suspending Component

```tsx
// ❌ WRONG — Suspense inside the component that calls useSuspenseQuery
function TicketBoard() {
  const { data } = useSuspenseQuery({
    queryKey: ['tickets'],
    queryFn: fetchTickets,
  })
  // ↑ The throw happens HERE, during render, BEFORE the return statement.
  //   React never reaches the JSX below.

  return (
    <Suspense fallback={<TicketSkeleton />}>   {/* ← never reached */}
      <TicketList tickets={data} />
    </Suspense>
  )
}
```

**Why it fails:** `useSuspenseQuery` throws a Promise synchronously during render. JavaScript stops executing the function at the throw — the `return` statement (and the `<Suspense>` inside it) is never reached. React walks **up** the tree to find a `<Suspense>` boundary, not down into unreturned JSX.

```tsx
// ✅ CORRECT — Suspense is in the PARENT component
function TicketPage() {
  return (
    <Suspense fallback={<TicketSkeleton />}>
      <TicketBoard />   {/* ← this is where the throw happens */}
    </Suspense>
  )
}

function TicketBoard() {
  const { data } = useSuspenseQuery({
    queryKey: ['tickets'],
    queryFn: fetchTickets,
  })

  // data is guaranteed defined — Suspense in the parent caught the Promise
  return <TicketList tickets={data} />
}
```

**Rule: `<Suspense>` must always be in a component _above_ the one that suspends — never inside it.**

### Gotcha — Should You Always Wrap Suspense with an ErrorBoundary?

**With `useSuspenseQuery`: yes.** It throws two things:

```
useSuspenseQuery
  ├── Data not ready → throws a Promise   → caught by <Suspense>
  └── Fetch failed   → throws an Error    → caught by <ErrorBoundary>
```

Without an ErrorBoundary, a failed query error **bubbles up** and crashes everything above it — same bubbling behavior as the missing Suspense problem.

**With `useQuery`: no.** Errors are returned as values, not thrown:

```tsx
// useQuery — errors are VALUES you check
const { data, isError, error } = useQuery({ queryKey: ['user'], queryFn: fetchUser })
if (isError) return <div>Error: {error.message}</div>  // you handle it inline

// useSuspenseQuery — errors are THROWN
const { data } = useSuspenseQuery({ queryKey: ['user'], queryFn: fetchUser })
// if fetchUser fails → Error is thrown → crashes component
// → ErrorBoundary above catches it
```

#### The Complete Pattern — Suspense + ErrorBoundary

```tsx
import { Suspense } from 'react'
import { ErrorBoundary } from 'react-error-boundary'  // npm i react-error-boundary

function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>

      <ErrorBoundary fallback={<div>Failed to load user profile</div>}>
        <Suspense fallback={<UserSkeleton />}>
          <UserProfile />   {/* useSuspenseQuery inside */}
        </Suspense>
      </ErrorBoundary>

      <ErrorBoundary fallback={<div>Failed to load stats</div>}>
        <Suspense fallback={<StatsSkeleton />}>
          <StatsPanel />    {/* useSuspenseQuery inside */}
        </Suspense>
      </ErrorBoundary>
    </div>
  )
}
```

**Order matters:** `ErrorBoundary` goes **outside** `Suspense`. Why?
- While loading: `useSuspenseQuery` throws a Promise → `Suspense` catches it → shows skeleton
- On error: `useSuspenseQuery` throws an Error → `Suspense` doesn't catch errors → bubbles up → `ErrorBoundary` catches it → shows error UI

If you put `ErrorBoundary` inside `Suspense`, errors thrown during the suspended render would bypass it.

#### ErrorBoundary with Reset (Retry Button)

```tsx
import { ErrorBoundary } from 'react-error-boundary'
import { useQueryErrorResetBoundary } from '@tanstack/react-query'

function DashboardPage() {
  // TanStack Query provides this hook to reset failed queries
  const { reset } = useQueryErrorResetBoundary()

  return (
    <ErrorBoundary
      onReset={reset}  // resets the failed query state when user clicks retry
      fallbackRender={({ resetErrorBoundary, error }) => (
        <div>
          <p>Something went wrong: {error.message}</p>
          <button onClick={resetErrorBoundary}>Try again</button>
        </div>
      )}
    >
      <Suspense fallback={<UserSkeleton />}>
        <UserProfile />
      </Suspense>
    </ErrorBoundary>
  )
}
```

`useQueryErrorResetBoundary` is from TanStack Query — when the user clicks "Try again":
1. `resetErrorBoundary()` resets the ErrorBoundary (re-renders children)
2. `onReset={reset}` resets TanStack Query's internal error state for the failed query
3. `useSuspenseQuery` retries the fetch
4. `Suspense` shows the skeleton while retrying

#### Next.js `error.tsx` — The Automatic ErrorBoundary

Just like `loading.tsx` auto-wraps pages in `<Suspense>`, **`error.tsx` auto-wraps pages in an Error Boundary**:

```
app/dashboard/
  ├── layout.tsx      ← NOT wrapped by error.tsx (stays visible during errors)
  ├── loading.tsx     ← auto-Suspense for page.tsx
  ├── error.tsx       ← auto-ErrorBoundary for page.tsx
  └── page.tsx        ← wrapped by BOTH automatically
```

```tsx
// app/dashboard/error.tsx
'use client'  // error.tsx MUST be a Client Component

export default function Error({
  error,
  reset,    // function to retry rendering the page
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  )
}
```

What Next.js generates internally:

```tsx
<DashboardLayout>                           {/* NOT wrapped — stays visible */}
  <ErrorBoundary fallback={<Error />}>      {/* ← from error.tsx */}
    <Suspense fallback={<Loading />}>       {/* ← from loading.tsx */}
      <DashboardPage />                     {/* ← your page.tsx */}
    </Suspense>
  </ErrorBoundary>
</DashboardLayout>
```

Notice the order: `ErrorBoundary` outside, `Suspense` inside — exactly the correct pattern.

#### When Do You Need Manual ErrorBoundary vs `error.tsx`?

| Scenario | Use `error.tsx` | Use manual `<ErrorBoundary>` |
|---|---|---|
| Page-level errors | Yes — catches any error in the page | Overkill |
| Per-component error UI | No — too coarse (wraps entire page) | Yes — wrap individual `<Suspense>` boundaries |
| Retry with TanStack Query reset | No — `error.tsx` doesn't know about TanStack Query | Yes — use `useQueryErrorResetBoundary` |
| Layout errors | No — `error.tsx` doesn't wrap the layout | Need `global-error.tsx` at app root |

**Rule of thumb:**
- Always have `error.tsx` as a page-level safety net
- Add manual `<ErrorBoundary>` around individual `<Suspense>` boundaries when you want per-component error UI or TanStack Query retry integration

### Next.js Counterpart
Next.js `loading.tsx` files and `<Suspense>` boundaries with async Server Components achieve the same pattern natively:

```tsx
// app/user/[id]/loading.tsx — auto-Suspense boundary
export default function Loading() {
  return <UserProfileSkeleton />
}

// app/user/[id]/page.tsx — async Server Component
export default async function UserPage({ params }) {
  const { id } = await params
  const user = await fetchUser(id) // Suspense handles the loading state
  return <UserProfile user={user} />
}
```

### useSuspenseQuery with Server-Side Prefetching (The Full Pattern)

This is the **production pattern** — server prefetches the data, `HydrationBoundary` transfers it to the client, and `useSuspenseQuery` picks it up instantly with no loading state.

```tsx
// 1. Shared query options — used by BOTH server and client
// hooks/queries/use-user.ts
import { queryOptions } from '@tanstack/react-query'

export function userQueryOptions(userId: string) {
  return queryOptions({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000,
  })
}
```

```tsx
// 2. Server Component — prefetches + dehydrates + wraps with boundaries
// app/user/[id]/page.tsx
import { dehydrate, HydrationBoundary } from '@tanstack/react-query'
import { getQueryClient } from '@/lib/query-client'
import { userQueryOptions } from '@/hooks/queries/use-user'
import { UserProfile } from './user-profile'
import { Suspense } from 'react'
import { ErrorBoundary } from 'react-error-boundary'

export default async function UserPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const queryClient = getQueryClient()

  // Prefetch on the server — data goes into queryClient cache
  await queryClient.prefetchQuery(userQueryOptions(id))

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ErrorBoundary fallback={<div>Failed to load profile</div>}>
        <Suspense fallback={<UserProfileSkeleton />}>
          <UserProfile userId={id} />
        </Suspense>
      </ErrorBoundary>
    </HydrationBoundary>
  )
}
```

```tsx
// 3. Client Component — useSuspenseQuery picks up hydrated data
// app/user/[id]/user-profile.tsx
'use client'

import { useSuspenseQuery } from '@tanstack/react-query'
import { userQueryOptions } from '@/hooks/queries/use-user'

export function UserProfile({ userId }: { userId: string }) {
  const { data: user } = useSuspenseQuery(userQueryOptions(userId))

  // data is GUARANTEED defined
  // On initial load: instant (hydrated from server prefetch)
  // On client navigation: Suspense fallback shows until fetch completes
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  )
}
```

**What happens on initial page load:**
```
Server:
  1. prefetchQuery runs → data stored in queryClient
  2. dehydrate() → serializes cache to JSON
  3. HydrationBoundary → sends JSON to client
  4. HTML with rendered content sent to browser

Client:
  1. HydrationBoundary hydrates data into browser's QueryClient
  2. useSuspenseQuery checks cache → finds ['user', id] → data exists!
  3. Component renders instantly — Suspense fallback NEVER shows
  4. ErrorBoundary sits idle — no error
```

**What happens on client-side navigation (no server prefetch):**
```
Client:
  1. useSuspenseQuery checks cache → no data for this userId
  2. Throws Promise → Suspense catches it → shows <UserProfileSkeleton />
  3. Fetch runs client-side
  4. Data arrives → component renders
  (ErrorBoundary catches if fetch fails)
```

**Key insight:** The `<Suspense>` fallback is **invisible on server-prefetched pages** (data is already there) but **essential for client-side navigations** where no prefetch happened. Always include it.

### useQuery() vs useSuspenseQuery() — Full Behavioral Comparison

#### Do I always need `if(!data)` with useQuery?

**Yes.** `useQuery` returns `data: T | undefined`. Before the first successful fetch, `data` is `undefined`. You must guard every access:

```tsx
// useQuery — data can be undefined at any point before first fetch
const { data: user } = useQuery({ queryKey: ['user'], queryFn: fetchUser })

user.name    // ❌ Runtime crash if data hasn't loaded yet
user?.name   // ⚠️ Works but returns undefined — your UI shows nothing
if (!user) return <Skeleton />  // ✅ You MUST do this every time

// useSuspenseQuery — data is NEVER undefined
const { data: user } = useSuspenseQuery({ queryKey: ['user'], queryFn: fetchUser })

user.name    // ✅ Always safe — component only renders when data exists
```

This isn't just convenience — it's a **TypeScript type difference**:

```tsx
// useQuery
const { data } = useQuery<User>(...)
//     ^? data: User | undefined

// useSuspenseQuery
const { data } = useSuspenseQuery<User>(...)
//     ^? data: User              ← no undefined in the union
```

#### The Full Comparison

| Behavior | `useQuery` | `useSuspenseQuery` | Why the difference |
|---|---|---|---|
| **`data` type** | `T \| undefined` | `T` (guaranteed) | `useSuspenseQuery` doesn't render the component until data is ready — so `undefined` is impossible |
| **`if(!data)` guard needed?** | Yes, always | No, never | Same reason — component only exists after data resolves |
| **Loading state** | Manual: `if (isPending) return <Skeleton />` inside the component | Declarative: `<Suspense fallback={<Skeleton />}>` in the parent | `useSuspenseQuery` throws a Promise that React Suspense catches — the component never renders in a loading state |
| **Error state** | Manual: `if (isError) return <Error />` inside the component | Declarative: `<ErrorBoundary>` in the parent | `useSuspenseQuery` throws the error — React Error Boundary catches it |
| **`enabled` option** | Available — can disable/enable fetching conditionally | NOT available | Suspense contract: a component that "might not fetch" breaks the Suspense model. If the component renders, it MUST fetch. React can't show a fallback for "maybe loading, maybe not" |
| **`placeholderData`** | Available — show placeholder while real data loads | NOT available | No need — Suspense fallback IS the placeholder |
| **`isLoading` / `isPending`** | Available and useful | Exists but always `false` when component renders | The component never renders in a pending state — Suspense handles that phase |
| **`isFetching`** | Useful for background refetch indicators | Still useful — `true` during background refetches | Background refetches don't suspend (only initial fetch does) |
| **Parallel queries in same component** | Run in parallel (both start immediately) | WATERFALL — second waits for first | Each `useSuspenseQuery` throws independently. First throws → component suspends → React retries → first has data → second throws → suspends again. Fix: use `useSuspenseQueries()` |
| **Where skeleton lives** | Inside the component (`if (isPending) return ...`) | In the parent `<Suspense fallback={...}>` | Fundamentally different mental model — Suspense pushes loading UI UP the tree, out of the data component |
| **Re-renders on refetch** | Component stays mounted, `isFetching` toggles | Component stays mounted (background refetches don't re-suspend) | Only the INITIAL fetch suspends. Once data exists, refetches are background operations |
| **Server-side with HydrationBoundary** | Shows data instantly if hydrated; shows loading if not | Shows data instantly if hydrated; suspends if not | Same hydration mechanism — the difference is what happens when cache is empty |

#### When to Use Each

```
Does this component make sense WITHOUT the data?
├── NO (e.g., user profile, post detail, data table)
│   └── useSuspenseQuery ← component can't render anything useful without data
│
└── YES (e.g., form with optional prefill, dashboard with progressive loading)
    └── useQuery ← component has meaningful partial UI

Do you need conditional fetching?
├── YES (e.g., "fetch details only when user clicks an item")
│   └── useQuery with enabled: !!selectedId
│
└── NO (always fetch when component renders)
    └── useSuspenseQuery

Are you fetching multiple independent queries in one component?
├── YES
│   ├── useQuery → they run in parallel automatically ✅
│   └── useSuspenseQuery → they WATERFALL ❌ (use useSuspenseQueries instead)
│
└── NO (single query)
    └── Either works — useSuspenseQuery is cleaner
```

**Rule of thumb:** Default to `useSuspenseQuery`. Fall back to `useQuery` only when you need `enabled`, `placeholderData`, or the component has meaningful partial UI without data.

---

## 6. usePrefetchQuery()
### Why
Starts fetching data **before** a component that needs it mounts. The prefetched data lands in the cache so when `useQuery()` runs, it finds data immediately — no loading spinner.

### What
A hook that triggers a prefetch during render. Does NOT return any data. It just warms the cache.

### How

```tsx
'use client'

import { usePrefetchQuery, useQuery } from '@tanstack/react-query'
import { useState } from 'react'

function TodoList() {
  const [selectedId, setSelectedId] = useState<string | null>(null)

  // Prefetch details for the first item while list renders
  usePrefetchQuery({
    queryKey: ['todo', '1'],
    queryFn: () => fetchTodo('1'),
  })

  const { data: todos } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  })

  return (
    <div>
      {todos?.map(todo => (
        <div
          key={todo.id}
          onMouseEnter={() => {
            // Prefetch on hover — by the time they click, data is ready
            queryClient.prefetchQuery({
              queryKey: ['todo', todo.id],
              queryFn: () => fetchTodo(todo.id),
            })
          }}
          onClick={() => setSelectedId(todo.id)}
        >
          {todo.title}
        </div>
      ))}
      {selectedId && <TodoDetail id={selectedId} />}
    </div>
  )
}
```

### When
- Prefetch the "next likely" page/data on hover
- Prefetch data for a tab the user hasn't clicked yet
- Warm the cache during render for a child component

### Gotcha
`usePrefetchQuery` will only prefetch if there's no existing data in the cache for that key. If data exists and is still fresh (`staleTime`), it's a no-op.

---

## 7. queryClient.prefetchQuery()
### Why
This is the **server-side** prefetching method. Used in Next.js Server Components to pre-populate the cache before sending HTML to the client.

### What
An imperative method on `QueryClient` that fetches data and stores it in the cache. Returns a `Promise<void>` (not the data itself). The data is then available via `dehydrate()` → `HydrationBoundary`.

### How (The Full Server → Client Flow)

```tsx
// app/todos/page.tsx (Server Component)
import { dehydrate, HydrationBoundary } from '@tanstack/react-query'
import { getQueryClient } from '@/lib/get-query-client'
import { TodoList } from './todo-list'

export default async function TodosPage() {
  const queryClient = getQueryClient()

  // Prefetch on the server — this runs the queryFn and stores result in cache
  await queryClient.prefetchQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  })

  // dehydrate() serializes the cache state
  // HydrationBoundary sends it to the client
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <TodoList />
    </HydrationBoundary>
  )
}
```

```tsx
// app/todos/todo-list.tsx (Client Component)
'use client'

import { useQuery } from '@tanstack/react-query'

export function TodoList() {
  // This finds the prefetched data INSTANTLY — no loading state
  const { data: todos } = useQuery({
    queryKey: ['todos'],  // same key as prefetch
    queryFn: fetchTodos,  // same fn (used for refetching later)
  })

  return (
    <ul>
      {todos?.map(todo => <li key={todo.id}>{todo.title}</li>)}
    </ul>
  )
}
```

### Prefetching Multiple Queries in Parallel

```tsx
export default async function DashboardPage() {
  const queryClient = getQueryClient()

  // Fire all prefetches concurrently
  await Promise.all([
    queryClient.prefetchQuery({
      queryKey: ['user'],
      queryFn: fetchUser,
    }),
    queryClient.prefetchQuery({
      queryKey: ['notifications'],
      queryFn: fetchNotifications,
    }),
    queryClient.prefetchQuery({
      queryKey: ['stats'],
      queryFn: fetchStats,
    }),
  ])

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <Dashboard />
    </HydrationBoundary>
  )
}
```

### When
- **Every page that uses TanStack Query** should prefetch in its Server Component
- This is the primary integration point between Next.js and TanStack Query

### Gotcha — prefetchQuery swallows errors

> **TanStack Docs quote:** *"prefetchQuery never throws errors — if the query fails, the error is silently caught and the query is put into an error state. The error will be thrown when a component tries to use the data."*

This means your prefetch might fail silently, and the client component will see `isPending: true` instead of instant data, then re-fetch client-side. This is actually a reasonable fallback behavior.

---

## 8. useMutation / mutation.mutate / mutation.mutateAsync
### Why
Mutations are for **creating, updating, or deleting** data. Unlike queries (reads), mutations are imperative — they run when YOU tell them to, not automatically.

### What
`useMutation` returns a mutation object with `.mutate()` (fire-and-forget) and `.mutateAsync()` (returns a Promise). Provides lifecycle callbacks (`onMutate`, `onSuccess`, `onError`, `onSettled`).

### How

```tsx
'use client'

import { useMutation, useQueryClient } from '@tanstack/react-query'

async function createTodo(newTodo: { title: string }): Promise<Todo> {
  const res = await fetch('/api/todos', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(newTodo),
  })
  if (!res.ok) throw new Error('Failed to create todo')
  return res.json()
}

export function CreateTodoForm() {
  const queryClient = useQueryClient()

  const mutation = useMutation({
    mutationFn: createTodo,

    // Called BEFORE mutationFn — perfect for optimistic updates
    onMutate: async (newTodo) => {
      // Cancel any outgoing refetches so they don't overwrite our optimistic update
      await queryClient.cancelQueries({ queryKey: ['todos'] })

      // Snapshot the previous value (for rollback)
      const previousTodos = queryClient.getQueryData(['todos'])

      // Optimistically update the cache
      queryClient.setQueryData(['todos'], (old: Todo[]) => [
        ...old,
        { id: 'temp-id', ...newTodo, completed: false },
      ])

      // Return context with the snapshot
      return { previousTodos }
    },

    onSuccess: (data, variables, context) => {
      // Invalidate and refetch the real data
      queryClient.invalidateQueries({ queryKey: ['todos'] })
    },

    onError: (err, newTodo, context) => {
      // Rollback the optimistic update
      queryClient.setQueryData(['todos'], context?.previousTodos)
    },

    onSettled: () => {
      // Always runs (like finally) — good for cleanup
      queryClient.invalidateQueries({ queryKey: ['todos'] })
    },
  })

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault()
        const formData = new FormData(e.currentTarget)
        mutation.mutate({ title: formData.get('title') as string })
      }}
    >
      <input name="title" required />
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Creating...' : 'Create Todo'}
      </button>

      {mutation.isError && <p>Error: {mutation.error.message}</p>}
      {mutation.isSuccess && <p>Todo created!</p>}
    </form>
  )
}
```

### mutation.mutate vs mutation.mutateAsync

```tsx
// .mutate() — fire and forget, handles errors via onError callback
mutation.mutate(data)
// You CANNOT await this. Error handling is through the callbacks.

// .mutateAsync() — returns a Promise, you can await it
try {
  const result = await mutation.mutateAsync(data)
  console.log('Created:', result)
} catch (error) {
  console.error('Failed:', error)
}
// ⚠️ If you use .mutateAsync, you MUST handle errors yourself
// onError callback STILL fires, but the error also propagates to your catch
```

> **TanStack Docs quote:** *"By default, onError and onSettled callbacks on useMutation are called even if mutateAsync is used. If you want to prevent this, you can use throwOnError: true."*

### When
- Form submissions
- Creating / updating / deleting resources
- Any user-initiated action that changes server state

### Next.js Counterpart — Server Actions

```tsx
// Server Action
'use server'
import { revalidateTag } from 'next/cache'

export async function createTodo(formData: FormData) {
  const title = formData.get('title') as string
  await db.todos.create({ data: { title } })
  revalidateTag('todos') // Invalidates cache automatically
}

// Usage in component
<form action={createTodo}>
  <input name="title" />
  <button type="submit">Create</button>
</form>
```

**Key difference:** Server Actions don't give you `isPending`, optimistic updates, or `onError` callbacks natively. You'd need `useActionState()` (React 19) or `useTransition()` for pending states. TanStack Query mutations are more powerful for complex client interactions.

---

## 9. queryClient.invalidateQueries()
### Why
After a mutation changes data on the server, the cached data is stale. Invalidation marks queries as stale and optionally triggers a refetch, ensuring the UI shows fresh data.

### What
Marks matching queries as stale. If the query is currently being rendered by a component, it also triggers a background refetch.

### How

```tsx
const queryClient = useQueryClient()

// Invalidate a specific query
queryClient.invalidateQueries({ queryKey: ['todos'] })

// Invalidate all queries starting with 'todos'
// This matches ['todos'], ['todos', 1], ['todos', { filter: 'active' }], etc.
queryClient.invalidateQueries({ queryKey: ['todos'] })

// Invalidate ONLY exact match
queryClient.invalidateQueries({ queryKey: ['todos'], exact: true })
// Only matches ['todos'], NOT ['todos', 1]

// Invalidate everything
queryClient.invalidateQueries()

// Invalidate with a predicate function
queryClient.invalidateQueries({
  predicate: (query) =>
    query.queryKey[0] === 'todos' && (query.queryKey[1] as any)?.version >= 10,
})

// Refetch type control
queryClient.invalidateQueries({
  queryKey: ['todos'],
  refetchType: 'active',   // default: only refetch active (mounted) queries
  // refetchType: 'inactive' — only refetch unmounted queries
  // refetchType: 'all' — refetch everything
  // refetchType: 'none' — don't refetch, just mark stale
})
```

### The Invalidation → Refetch Flow

```
invalidateQueries(['todos'])
  ↓
All queries with key starting with ['todos'] marked as stale
  ↓
Any component currently rendering those queries triggers a background refetch
  ↓
Component shows old data while fetching (isFetching: true, isPending: false)
  ↓
New data arrives → component re-renders with fresh data
```

### Common Pattern — Invalidate After Mutation

```tsx
const mutation = useMutation({
  mutationFn: updateTodo,
  onSuccess: () => {
    // Invalidate and refetch
    queryClient.invalidateQueries({ queryKey: ['todos'] })
    // Also invalidate related queries
    queryClient.invalidateQueries({ queryKey: ['todoCount'] })
  },
})
```

### When
- After any mutation that changes data
- When you know external data has changed
- When a WebSocket event tells you something is stale

### Next.js Counterpart

```tsx
// Server Action
'use server'
import { revalidateTag, revalidatePath } from 'next/cache'

export async function updateTodo(id: string, data: any) {
  await db.todos.update({ where: { id }, data })

  revalidateTag('todos')      // Invalidate by cache tag
  revalidatePath('/todos')    // Invalidate by route path
}
```

**Key difference:** Next.js invalidation triggers a full page re-render from the server. TanStack Query invalidation triggers a background refetch and updates only the affected components — much more granular.

---

## 10. HydrationBoundary
### Why
This is the **bridge** between Server Components and Client Components for TanStack Query. It transfers the server-side prefetched cache to the client so `useQuery()` finds data instantly.

### What
A component that takes dehydrated (serialized) query cache state and hydrates it into the client's `QueryClient`. On the server, `dehydrate(queryClient)` serializes the cache. On the client, `<HydrationBoundary>` deserializes it.

### How — The Complete Pattern

```tsx
// app/posts/page.tsx (Server Component)
import { dehydrate, HydrationBoundary } from '@tanstack/react-query'
import { getQueryClient } from '@/lib/get-query-client'
import { PostList } from './post-list'

export default async function PostsPage() {
  const queryClient = getQueryClient()

  await queryClient.prefetchQuery({
    queryKey: ['posts'],
    queryFn: async () => {
      // This runs on the server — can access DB directly!
      const res = await fetch('https://api.example.com/posts', {
        headers: { Authorization: `Bearer ${process.env.API_SECRET}` },
      })
      return res.json()
    },
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostList />
    </HydrationBoundary>
  )
}
```

```tsx
// app/posts/post-list.tsx (Client Component)
'use client'

import { useQuery } from '@tanstack/react-query'

export function PostList() {
  const { data: posts } = useQuery({
    queryKey: ['posts'],  // MUST match the prefetch key exactly
    queryFn: fetchPosts,  // Used for subsequent refetches
  })

  // On first render: posts is IMMEDIATELY available (no loading state)
  // On subsequent renders: TanStack Query manages refetching
  return (
    <ul>
      {posts?.map(post => <li key={post.id}>{post.title}</li>)}
    </ul>
  )
}
```

### Nesting HydrationBoundaries

You can nest them. Each one merges its state into the client cache:

```tsx
// app/dashboard/page.tsx
export default async function DashboardPage() {
  const queryClient = getQueryClient()

  await queryClient.prefetchQuery({
    queryKey: ['user'],
    queryFn: fetchUser,
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <UserHeader />
      {/* Nested Server Component with its own prefetch */}
      <StatsSection />
    </HydrationBoundary>
  )
}

// app/dashboard/stats-section.tsx (Server Component)
export default async function StatsSection() {
  const queryClient = getQueryClient()

  await queryClient.prefetchQuery({
    queryKey: ['stats'],
    queryFn: fetchStats,
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <StatsChart />
    </HydrationBoundary>
  )
}
```

> **TanStack Docs quote:** *"HydrationBoundary can be used multiple times in the same component tree. Each one will merge its dehydrated state with the existing cache."*

### When
- **Every page** that needs TanStack Query data should use this pattern
- It's the standard way to avoid the "flash of loading state" with SSR

### Gotcha — staleTime MUST be > 0

If `staleTime` is `0` (the default), the hydrated data is **immediately considered stale**, and `useQuery` will trigger a refetch on mount — defeating the purpose of prefetching.

> **TanStack Docs quote:** *"If you set staleTime to 0 (which is the default), the query will be immediately refetched on the client after hydration. This might not be what you want if the data was just fetched on the server."*

**Always set `staleTime` in your default options or per-query when using SSR hydration.**

### Can/Should You Add `<Suspense>` Inside `<HydrationBoundary>`?

**Can you?** Yes — they solve completely different problems and compose naturally.

- `HydrationBoundary` = data transfer (injects server cache into client `QueryClient`)
- `Suspense` = rendering pause (shows fallback while a component suspends)

They don't interfere with each other. `HydrationBoundary` is not a loading boundary — it's a cache hydration point.

#### The Three Patterns

**Pattern 1: `Suspense` outside `HydrationBoundary`**

```tsx
// app/posts/page.tsx (Server Component)
export default async function PostsPage() {
  const queryClient = getQueryClient()
  await queryClient.prefetchQuery(postsQueryOptions())

  return (
    <Suspense fallback={<PostsSkeleton />}>
      <HydrationBoundary state={dehydrate(queryClient)}>
        <PostList />  {/* useSuspenseQuery inside */}
      </HydrationBoundary>
    </Suspense>
  )
}
```

Works, but the `<Suspense>` wraps the entire hydration boundary. If you have multiple client components inside, they all share one fallback.

**Pattern 2: `Suspense` inside `HydrationBoundary` (recommended)**

```tsx
// app/dashboard/page.tsx (Server Component)
export default async function DashboardPage() {
  const queryClient = getQueryClient()
  await Promise.all([
    queryClient.prefetchQuery(userQueryOptions()),
    queryClient.prefetchQuery(statsQueryOptions()),
    queryClient.prefetchQuery(notificationsQueryOptions()),
  ])

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      {/* Each component gets its own Suspense boundary */}
      <Suspense fallback={<UserHeaderSkeleton />}>
        <UserHeader />
      </Suspense>

      <Suspense fallback={<StatsSkeleton />}>
        <StatsPanel />
      </Suspense>

      <Suspense fallback={<NotificationsSkeleton />}>
        <NotificationList />
      </Suspense>
    </HydrationBoundary>
  )
}
```

This is the **recommended** pattern. One `HydrationBoundary` hydrates all the data, and individual `<Suspense>` boundaries give each component its own fallback.

**Pattern 3: `loading.tsx` — Next.js auto-wraps your page in `<Suspense>` (most common)**

You don't write `<Suspense>` yourself. Just create `loading.tsx` and Next.js does it for you:

```
app/dashboard/
  ├── layout.tsx
  ├── loading.tsx    ← you create this
  └── page.tsx       ← Next.js auto-wraps this in <Suspense fallback={<Loading />}>
```

```tsx
// loading.tsx — this becomes the Suspense fallback
export default function Loading() {
  return <DashboardSkeleton />
}

// page.tsx — has HydrationBoundary inside
export default async function DashboardPage() {
  const queryClient = getQueryClient()
  await queryClient.prefetchQuery(postsQueryOptions())

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostList />
    </HydrationBoundary>
  )
}
```

What Next.js actually renders (you never write this — it's automatic):

```tsx
<DashboardLayout>
  <Suspense fallback={<Loading />}>       {/* ← Next.js adds this from loading.tsx */}
    <DashboardPage />                      {/* ← your page.tsx (with HydrationBoundary inside) */}
  </Suspense>                              {/* ← Next.js adds this */}
</DashboardLayout>
```

**`loading.tsx` = page-level Suspense** — one skeleton for the entire page. If you need granular control (different skeletons for different sections), use manual `<Suspense>` boundaries inside your page (Pattern 2) instead of or in addition to `loading.tsx`.

```tsx
// loading.tsx → one skeleton wraps the whole page
// User sees: DashboardSkeleton → everything appears at once

// Manual <Suspense> inside page.tsx → per-section skeletons
// User sees: h1 immediately + CardsSkeleton + ChartSkeleton → Cards appear → Chart appears
```

#### But When Does the Suspense Fallback Actually Show?

Here's the subtle part. If you `await` the prefetch:

```tsx
await queryClient.prefetchQuery(postsQueryOptions())
// ↑ Page waits for data → data is in cache → dehydrated → hydrated on client
```

Then `useSuspenseQuery` on the client **finds data in cache instantly and never suspends**. The `<Suspense>` fallback **never shows** on initial page load.

So why add `<Suspense>` at all? It's a **safety net** for these scenarios:

| Scenario | What happens without `<Suspense>` | What happens with `<Suspense>` |
|---|---|---|
| **Prefetch succeeded** (happy path) | Data loads instantly, no issue | Data loads instantly, fallback never shows |
| **Prefetch failed silently** (`prefetchQuery` swallows errors) | `useSuspenseQuery` throws a Promise → bubbles to nearest ancestor Suspense (wrong skeleton or blank page) | Your fallback shows, `useSuspenseQuery` refetches client-side |
| **Key mismatch** (server prefetched `['posts']`, client queries `['post']`) | Same as above — no cache hit, uncontrolled suspend | Your fallback shows while client fetches |
| **Client-side navigation** (no server prefetch on this route) | Promise bubbles up — uncontrolled loading UI | Your fallback shows properly |
| **`void` prefetch** (non-blocking, data not ready at dehydrate time) | Promise bubbles up | Your fallback shows while client fetches |

**Rule of thumb:** Always add `<Suspense>` even when you `await` the prefetch. The `await` makes the fallback invisible on the happy path (which is great — no loading spinner!), but the `<Suspense>` catches edge cases gracefully.

---

## 11. useInfiniteQuery()
### Why
For "load more" or infinite scroll patterns where data comes in pages/cursors and you accumulate results.

### What
Like `useQuery()` but manages an array of pages. Provides `fetchNextPage`, `fetchPreviousPage`, `hasNextPage`, `hasPreviousPage`, and `isFetchingNextPage`.

### How

```tsx
'use client'

import { useInfiniteQuery } from '@tanstack/react-query'
import { useInView } from 'react-intersection-observer' // for auto-loading
import { useEffect } from 'react'

interface PostsResponse {
  posts: Post[]
  nextCursor: string | null
}

async function fetchPosts({ pageParam }: { pageParam: string | null }): Promise<PostsResponse> {
  const url = pageParam
    ? `/api/posts?cursor=${pageParam}`
    : '/api/posts'
  const res = await fetch(url)
  return res.json()
}

export function InfinitePostList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isPending,
    isError,
    error,
  } = useInfiniteQuery({
    queryKey: ['posts', 'infinite'],
    queryFn: fetchPosts,
    initialPageParam: null as string | null,
    getNextPageParam: (lastPage) => lastPage.nextCursor, // return undefined to signal no more pages

    // Optional: bi-directional scrolling
    // getPreviousPageParam: (firstPage) => firstPage.prevCursor,

    // Optional: limit cached pages to save memory
    maxPages: 10, // only keep last 10 pages in cache
  })

  // Auto-load on scroll with Intersection Observer
  const { ref, inView } = useInView()
  useEffect(() => {
    if (inView && hasNextPage && !isFetchingNextPage) {
      fetchNextPage()
    }
  }, [inView, hasNextPage, isFetchingNextPage, fetchNextPage])

  if (isPending) return <div>Loading...</div>
  if (isError) return <div>Error: {error.message}</div>

  return (
    <div>
      {/* data.pages is an array of pages, each containing your response */}
      {data.pages.map((page, pageIndex) => (
        <div key={pageIndex}>
          {page.posts.map(post => (
            <PostCard key={post.id} post={post} />
          ))}
        </div>
      ))}

      {/* Sentinel element for intersection observer */}
      <div ref={ref}>
        {isFetchingNextPage ? 'Loading more...' : hasNextPage ? 'Scroll for more' : 'No more posts'}
      </div>
    </div>
  )
}
```

### The `data.pages` Structure

```tsx
// After loading 3 pages, data looks like:
{
  pages: [
    { posts: [post1, post2, ...], nextCursor: 'abc' },  // page 0
    { posts: [post11, post12, ...], nextCursor: 'def' }, // page 1
    { posts: [post21, post22, ...], nextCursor: null },  // page 2 (last)
  ],
  pageParams: [null, 'abc', 'def'], // the params used to fetch each page
}
```

### Flattening Pages

```tsx
// Flatten all pages into a single array
const allPosts = data?.pages.flatMap(page => page.posts) ?? []
```

### When
- Social feeds, comment threads
- Product listings with "Load More"
- Chat history (bi-directional with `getPreviousPageParam`)
- Any cursor-based or offset-based paginated API

### Gotcha — Refetching Infinite Queries

> **TanStack Docs quote:** *"When an infinite query becomes stale and needs to be refetched, each group is fetched sequentially, starting from the first one. This ensures that even if the underlying data is mutated, we don't use stale cursors."*

This means refetching an infinite query with 10 loaded pages makes **10 sequential requests**. Use `maxPages` to limit this.

### Gotcha — Cache Key Must Be Different From useQuery

If you use `['posts']` for both `useQuery` and `useInfiniteQuery`, they will **conflict** because they store data in different shapes. Always differentiate:

```tsx
useQuery({ queryKey: ['posts', 'list'] })           // single-page
useInfiniteQuery({ queryKey: ['posts', 'infinite'] }) // multi-page
```

---

## 12. Pagination with TanStack Query
### Traditional Pagination (Page Numbers)

```tsx
'use client'

import { useQuery, keepPreviousData } from '@tanstack/react-query'
import { useState } from 'react'

export function PaginatedPosts() {
  const [page, setPage] = useState(1)

  const { data, isPending, isPlaceholderData, isFetching } = useQuery({
    queryKey: ['posts', { page }],
    queryFn: () => fetchPosts(page),
    placeholderData: keepPreviousData, // ← KEY: shows old page while new page loads
  })

  return (
    <div>
      {/* Show old data with opacity while fetching new page */}
      <div style={{ opacity: isPlaceholderData ? 0.5 : 1 }}>
        {data?.posts.map(post => <PostCard key={post.id} post={post} />)}
      </div>

      <div>
        <button
          onClick={() => setPage(old => Math.max(old - 1, 1))}
          disabled={page === 1}
        >
          Previous
        </button>

        <span>Page {page} of {data?.totalPages}</span>

        <button
          onClick={() => setPage(old => old + 1)}
          // Disable Next if we're on last page or showing placeholder data
          disabled={isPlaceholderData || !data?.hasMore}
        >
          Next
        </button>
      </div>

      {isFetching && <span>Updating...</span>}
    </div>
  )
}
```

### keepPreviousData — The Key to Smooth Pagination

> **TanStack Docs quote:** *"keepPreviousData was previously a boolean option in v4. In v5, it has been replaced with the identity function keepPreviousData that you pass to placeholderData."*

**What it does:** When the query key changes (e.g., `page` 1 → 2), instead of showing a loading state, it shows the **previous page's data** while the new page loads. `isPlaceholderData` becomes `true` so you can show a visual indicator.

### Prefetching the Next Page

```tsx
import { useQueryClient } from '@tanstack/react-query'

function PaginatedPosts() {
  const queryClient = useQueryClient()
  const [page, setPage] = useState(1)

  const { data } = useQuery({
    queryKey: ['posts', { page }],
    queryFn: () => fetchPosts(page),
    placeholderData: keepPreviousData,
  })

  // Prefetch next page eagerly
  useEffect(() => {
    if (data?.hasMore) {
      queryClient.prefetchQuery({
        queryKey: ['posts', { page: page + 1 }],
        queryFn: () => fetchPosts(page + 1),
      })
    }
  }, [data, page, queryClient])

  // ...
}
```

### When to Use Pagination vs. Infinite Query

| | Pagination | Infinite Query |
|---|---|---|
| UI | Page numbers, prev/next buttons | "Load more" button or infinite scroll |
| Data | Replace current page | Accumulate pages |
| Memory | Constant (one page) | Grows (all loaded pages) |
| URL | Easy to bookmark (`?page=3`) | Hard to bookmark |
| Refetch | One request | N requests (one per loaded page) |

---

## 13. Next.js Counterparts Comparison
| TanStack Query | Next.js 16+ Counterpart | When to Choose TanStack Query |
|---|---|---|
| `useQuery()` | `async` Server Component with `await fetch()` | Client-side interactivity, real-time data, filters |
| `useQueries()` | `Promise.all()` in Server Component | Dynamic number of client-side queries |
| `useSuspenseQuery()` | `loading.tsx` + async Server Component | Client-side Suspense with cache control |
| `usePrefetchQuery()` | React's `<link rel="prefetch">`, Route prefetching | Client-side navigation prefetching |
| `queryClient.prefetchQuery()` | `use cache` / `fetch()` with cache options | When you need both server prefetch AND client-side cache |
| `useMutation` / `.mutate()` | Server Actions (`'use server'`) | Optimistic updates, complex mutation flows |
| `queryClient.invalidateQueries()` | `revalidateTag()` / `revalidatePath()` | Granular client-side cache control |
| `HydrationBoundary` | Streaming + `<Suspense>` | When you need TanStack Query client-side after SSR |
| `useInfiniteQuery()` | No direct counterpart (build custom) | Infinite scroll, cursor pagination |
| Pagination | `searchParams` + `use cache` | Client-side pagination without page reload |

### The Decision Framework

```
Does this data change based on user interaction on the client?
├── YES → TanStack Query (useQuery, useMutation)
│         Examples: search results, filters, shopping cart, chat
└── NO → Server Component
    ├── Does it need to be real-time?
    │   ├── YES → TanStack Query with refetchInterval
    │   └── NO → Server Component with `use cache`
    └── Does it need optimistic updates?
        ├── YES → TanStack Query mutation
        └── NO → Server Action
```

---

## 14. Scalable Architecture — Putting It All Together
### File Structure

```
src/
├── app/
│   ├── layout.tsx              # Root layout (wraps with QueryProvider)
│   ├── (dashboard)/
│   │   ├── page.tsx            # Server Component — prefetches with queryClient
│   │   └── components/
│   │       ├── stats-panel.tsx  # Client Component — useQuery
│   │       └── activity-feed.tsx # Client Component — useInfiniteQuery
│   └── posts/
│       ├── page.tsx            # Server Component — prefetches posts
│       ├── [id]/
│       │   └── page.tsx        # Server Component — prefetches single post
│       └── components/
│           ├── post-list.tsx   # Client Component — useQuery
│           └── create-post.tsx # Client Component — useMutation
├── lib/
│   ├── get-query-client.ts     # Server/client QueryClient factory
│   └── api.ts                  # Shared API functions
├── hooks/
│   └── queries/
│       ├── use-posts.ts        # Custom hook wrapping useQuery for posts
│       ├── use-create-post.ts  # Custom hook wrapping useMutation
│       └── use-user.ts         # Custom hook wrapping useQuery for user
└── providers/
    └── query-provider.tsx      # QueryClientProvider wrapper
```

### Custom Query Hooks (The Scalable Pattern)

**Do NOT scatter `useQuery` calls with inline `queryKey` and `queryFn` across components.** Extract them into custom hooks:

```tsx
// hooks/queries/use-posts.ts
'use client'

import { useQuery, useMutation, useQueryClient, queryOptions } from '@tanstack/react-query'
import { fetchPosts, fetchPost, createPost, updatePost, deletePost } from '@/lib/api'

// queryOptions() — creates a reusable options object
// Can be used by both useQuery (client) and prefetchQuery (server)
export function postsQueryOptions(filters?: PostFilters) {
  return queryOptions({
    queryKey: ['posts', filters ?? 'all'],
    queryFn: () => fetchPosts(filters),
    staleTime: 5 * 60 * 1000,
  })
}

export function postQueryOptions(id: string) {
  return queryOptions({
    queryKey: ['posts', id],
    queryFn: () => fetchPost(id),
    staleTime: 5 * 60 * 1000,
  })
}

// Custom hooks for components
export function usePosts(filters?: PostFilters) {
  return useQuery(postsQueryOptions(filters))
}

export function usePost(id: string) {
  return useQuery(postQueryOptions(id))
}

export function useCreatePost() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: createPost,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['posts'] })
    },
  })
}

export function useUpdatePost() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Partial<Post> }) =>
      updatePost(id, data),
    onSuccess: (_, { id }) => {
      queryClient.invalidateQueries({ queryKey: ['posts'] })
      queryClient.invalidateQueries({ queryKey: ['posts', id] })
    },
  })
}

export function useDeletePost() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: deletePost,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['posts'] })
    },
  })
}
```

### Using queryOptions() for Server ↔ Client Consistency

The `queryOptions()` helper ensures the **same key and function** are used on both server and client:

```tsx
// app/posts/page.tsx (Server Component)
import { dehydrate, HydrationBoundary } from '@tanstack/react-query'
import { getQueryClient } from '@/lib/get-query-client'
import { postsQueryOptions } from '@/hooks/queries/use-posts'
import { PostList } from './components/post-list'

export default async function PostsPage() {
  const queryClient = getQueryClient()

  // Uses the SAME options as the client hook
  await queryClient.prefetchQuery(postsQueryOptions())

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostList />
    </HydrationBoundary>
  )
}
```

```tsx
// app/posts/components/post-list.tsx (Client Component)
'use client'

import { usePosts, useDeletePost } from '@/hooks/queries/use-posts'

export function PostList() {
  const { data: posts, isPending } = usePosts()
  const deletePost = useDeletePost()

  if (isPending) return <div>Loading...</div>

  return (
    <ul>
      {posts?.map(post => (
        <li key={post.id}>
          {post.title}
          <button onClick={() => deletePost.mutate(post.id)}>Delete</button>
        </li>
      ))}
    </ul>
  )
}
```

### Query Key Factory Pattern

For large apps, use a key factory to prevent key collisions:

```tsx
// lib/query-keys.ts
export const queryKeys = {
  posts: {
    all: ['posts'] as const,
    lists: () => [...queryKeys.posts.all, 'list'] as const,
    list: (filters: PostFilters) => [...queryKeys.posts.lists(), filters] as const,
    details: () => [...queryKeys.posts.all, 'detail'] as const,
    detail: (id: string) => [...queryKeys.posts.details(), id] as const,
  },
  users: {
    all: ['users'] as const,
    detail: (id: string) => [...queryKeys.users.all, id] as const,
    posts: (userId: string) => [...queryKeys.users.all, userId, 'posts'] as const,
  },
} as const

// Usage:
useQuery({ queryKey: queryKeys.posts.detail('123'), queryFn: ... })
queryClient.invalidateQueries({ queryKey: queryKeys.posts.all })
// ↑ invalidates ALL post queries (lists, details, everything)
```

---

## 15. Gotchas & Caveats
### 1. `cacheTime` was renamed to `gcTime` in v5
```tsx
// ❌ v4
useQuery({ queryKey: ['posts'], cacheTime: 5 * 60 * 1000 })
// ✅ v5
useQuery({ queryKey: ['posts'], gcTime: 5 * 60 * 1000 })
```

### 2. `isLoading` vs `isPending` vs `isFetching`
- `isPending`: No data in cache yet (first load OR after cache eviction)
- `isLoading`: `isPending` AND `isFetching` — the very first fetch
- `isFetching`: ANY request is in-flight (including background refetches)

```
First load:     isPending=true,  isLoading=true,  isFetching=true
Background:     isPending=false, isLoading=false, isFetching=true
Cache hit:      isPending=false, isLoading=false, isFetching=false
```

### 3. Query keys are serialized, not compared by reference
```tsx
// These are THE SAME query (object keys are sorted):
useQuery({ queryKey: ['posts', { page: 1, sort: 'date' }] })
useQuery({ queryKey: ['posts', { sort: 'date', page: 1 }] })

// But arrays are order-dependent:
useQuery({ queryKey: ['posts', 1] })   // different from
useQuery({ queryKey: [1, 'posts'] })   // this
```

### 4. `enabled: false` prevents ALL fetching
```tsx
// This will NEVER fetch, even on mount
useQuery({
  queryKey: ['user'],
  queryFn: fetchUser,
  enabled: false, // must call refetch() manually
})
```

### 5. Mutations don't have cache keys (by default)
Mutations are not cached. Each `useMutation` is independent. If you call `mutation.mutate()` twice, it runs twice.

### 6. `select` doesn't affect what's cached
```tsx
useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  select: (data) => data.filter(p => p.published), // only affects THIS component
  // The FULL unfiltered data is stored in cache
})
```

### 7. `onSuccess`/`onError`/`onSettled` were REMOVED from useQuery in v5
```tsx
// ❌ v4 pattern — REMOVED in v5
useQuery({
  queryKey: ['posts'],
  onSuccess: (data) => { ... }, // DOES NOT EXIST in v5
})

// ✅ v5 — use useEffect or handle in the component
const { data } = useQuery({ queryKey: ['posts'], queryFn: fetchPosts })
useEffect(() => {
  if (data) { /* do something */ }
}, [data])
```

> **TanStack Docs:** These callbacks were removed because they fire on every render that has data, not just when fresh data arrives — leading to bugs with stale closures and infinite loops.

### 8. Server Components CANNOT use hooks
```tsx
// ❌ This will crash — Server Components can't use hooks
export default async function Page() {
  const { data } = useQuery({ ... }) // ERROR
}

// ✅ Use queryClient.prefetchQuery() in Server Components
export default async function Page() {
  const queryClient = getQueryClient()
  await queryClient.prefetchQuery({ ... })
  return <HydrationBoundary state={dehydrate(queryClient)}><ClientComponent /></HydrationBoundary>
}
```

### 9. `staleTime` vs `gcTime` confusion

```
staleTime: How long until data is considered "stale" (triggers refetch on mount/focus)
gcTime:    How long until UNUSED cache entries are garbage collected

Timeline:
  [fetch] ──── staleTime (fresh) ────→ [stale] ──── gcTime ────→ [removed from cache]
                                         ↑
                                    refetch triggers here
                                    (on mount, focus, etc.)
```

### 10. HydrationBoundary with mismatched keys = silent failure
If the server prefetches `['posts']` but the client queries `['post']` (typo), there's no error — the client just doesn't find cached data and fetches again. Use `queryOptions()` to prevent this.

### 11. Next.js `use cache` and TanStack Query serve different layers
- `use cache` caches on the **server** (between requests)
- TanStack Query caches on the **client** (between navigations)
- They can (and should) be used together: server caches API responses, TanStack Query caches on the client for interactivity

### 12. `refetchOnWindowFocus` can surprise you in development
In dev mode, switching between your IDE and browser triggers a refetch every time. This is working as intended but can be confusing. You can disable it in development:

```tsx
new QueryClient({
  defaultOptions: {
    queries: {
      refetchOnWindowFocus: process.env.NODE_ENV === 'production',
    },
  },
})
```

### 13. `fetchQuery` respects `staleTime` — it won't refetch fresh data

`fetchQuery` is designed for **getting data**, not forcing a refresh. If data for the key already exists in the cache and is still within `staleTime`, it returns the cached data without hitting the network.

```tsx
// ❌ "Why isn't this fetching new data?!"
function handleProjectChange(projectId: string) {
  queryClient.fetchQuery({
    queryKey: ['get-all-tickets'],
    queryFn: getAllTickets,
  })
  // If ['get-all-tickets'] is already cached and fresh → NOTHING HAPPENS
}

// ✅ Use refetchQueries to force a network request
function handleProjectChange(projectId: string) {
  queryClient.refetchQueries({
    queryKey: ['get-all-tickets'],
  })
  // Always hits the network, regardless of staleTime
}
```

```
fetchQuery:        "Do I have fresh data? Yes → return cached. No → fetch."
refetchQueries:    "Fetch NOW. I don't care what's in the cache."
invalidateQueries: "Mark as stale. Refetch if any component is currently using it."
```

### 14. Manual `refetchQueries` is usually a code smell — use queryKey variables instead

If you're calling `refetchQueries` manually when some state changes, your queryKey is probably missing a variable.

```tsx
// ❌ Missing variable in key → forced to manually refetch
const { data } = useSuspenseQuery({
  queryKey: ['get-all-tickets'],         // same key for ALL projects
  queryFn: getAllTickets,
})
// When project changes: queryClient.refetchQueries({ queryKey: ['get-all-tickets'] })
// Problem: old project's tickets flash briefly before new data arrives

// ✅ Include the variable in the key → automatic refetch on change
const { data } = useSuspenseQuery({
  queryKey: ['get-all-tickets', projectId],   // unique key per project
  queryFn: () => getAllTickets(projectId),
})
// When projectId changes → new key → no cached data → auto-fetches
// Bonus: switching BACK to old project shows cached data instantly
```

**Why this matters for UX:** With a shared key, when you refetch, the old data is **replaced** by new data in the same cache entry. With a key that includes the variable, each project gets its own cache entry — switching between projects feels instant because both are cached separately.

```
// ❌ Shared key: one cache slot, data gets replaced
cache: { ['get-all-tickets'] → Project A's tickets }
switch to B → refetch → cache: { ['get-all-tickets'] → Project B's tickets }
switch to A → refetch → cache: { ['get-all-tickets'] → Project A's tickets }  // fetched AGAIN

// ✅ Variable in key: separate cache slots
cache: {
  ['get-all-tickets', 'project-a'] → Project A's tickets,
  ['get-all-tickets', 'project-b'] → Project B's tickets,  // both cached!
}
switch between A and B → instant (no network request)
```

**Real-world example from this codebase:**

```tsx
// WorkspaceSelectorClient — just update state, no manual refetch
function handleValueChange(projectName: string) {
  setSelectedProject(projectName)  // or update URL param
  // NO refetchQueries needed — the key change in TicketBoard handles it
}

// TicketBoard — key includes projectId, TanStack Query handles the rest
const { data: tickets, isFetching } = useSuspenseQuery({
  queryKey: ['get-all-tickets', projectId],
  queryFn: () => getAllTickets(projectId),
})
```

> **Rule of thumb:** If your `queryFn` depends on a variable, that variable MUST be in the `queryKey`. If it's in the key, you almost never need manual refetch.

---

## 16. React Suspense — Server & Client Side in Next.js 16+

### What Is Suspense, Really?

Suspense is React's mechanism for **pausing rendering** of a component tree until some async work (data fetching, code loading) completes, showing a fallback UI in the meantime.

> **React Docs:** *"`<Suspense>` lets you display a fallback until its children have finished loading."*
> — https://react.dev/reference/react/Suspense

```tsx
<Suspense fallback={<Loading />}>
  <SomeComponent />
</Suspense>
```

#### How It Works Under the Hood — The Promise-Throwing Mechanism

When a component "suspends," it literally **throws a Promise**. React catches this thrown Promise, shows the `fallback` instead of the component tree, and waits. When the Promise resolves, React **re-renders** the suspended tree from scratch.

```
Component renders
  ↓
Encounters async data (not ready yet)
  ↓
Throws a Promise ← React catches this
  ↓
React shows <fallback> instead of the component tree
  ↓
Promise resolves (data is ready)
  ↓
React re-renders the component (data is now available)
  ↓
<fallback> is replaced with the actual component
```

**Critical:** Only **Suspense-enabled data sources** trigger this:
- `React.lazy()` for code splitting
- React's `use()` hook for reading cached Promises
- Suspense-enabled frameworks (Next.js, Relay)
- Libraries that implement the Suspense protocol (`useSuspenseQuery` in TanStack Query)

> **React Docs:** *"Suspense does not detect when data is fetched inside an Effect or event handler."*

This means plain `useEffect` + `fetch` does **NOT** work with Suspense:

```tsx
// ❌ This will NEVER trigger Suspense — useEffect doesn't throw promises
function Profile() {
  const [user, setUser] = useState(null)
  useEffect(() => {
    fetch('/api/user').then(r => r.json()).then(setUser)
  }, [])
  if (!user) return <Spinner /> // manual loading state — not Suspense
  return <div>{user.name}</div>
}

// ✅ This DOES trigger Suspense — use() reads a cached promise
import { use } from 'react'

function Profile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise) // suspends until promise resolves
  return <div>{user.name}</div>
}

// ✅ This DOES trigger Suspense — useSuspenseQuery throws a promise
import { useSuspenseQuery } from '@tanstack/react-query'

function Profile() {
  const { data: user } = useSuspenseQuery({
    queryKey: ['user'],
    queryFn: fetchUser,
  })
  return <div>{user.name}</div>
}
```

---

### Server-Side Suspense in Next.js 16+

On the server, Suspense enables **streaming** — sending HTML in chunks as components resolve, instead of waiting for the entire page.

#### 1. `loading.tsx` — Automatic Page-Level Suspense

Place a `loading.tsx` file in the same directory as `page.tsx`. Next.js automatically wraps the page in a `<Suspense>` boundary with this file as the fallback.

```tsx
// app/dashboard/loading.tsx
import { DashboardSkeleton } from '@/components/skeletons'

export default function Loading() {
  return <DashboardSkeleton />
}
```

> **Next.js Docs:** *"In the same folder, `loading.js` will be nested inside `layout.js`. It will automatically wrap the `page.js` file and any children below in a `<Suspense>` boundary."*
> — https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming

What Next.js generates internally:

```tsx
// What you write:
// app/dashboard/layout.tsx + loading.tsx + page.tsx

// What Next.js creates (conceptually):
<DashboardLayout>
  <Suspense fallback={<Loading />}>
    <DashboardPage />
  </Suspense>
</DashboardLayout>
```

**Key behaviors:**
- The layout renders **immediately** (it's NOT wrapped in Suspense)
- The `loading.tsx` skeleton is shown instantly while the page streams
- Navigation is **interruptible** — user can navigate away before page finishes loading
- The fallback UI is **prefetched**, making navigation feel instant

> **Next.js Learn (Vercel):** *"Since `<SideNav>` is static, it's shown immediately. The user can interact with `<SideNav>` while the dynamic content is loading."*
> — https://nextjs.org/learn/dashboard-app/streaming

**Gotcha — `loading.tsx` scope:**
- It wraps `page.tsx` and all children below
- It does **NOT** wrap `layout.tsx`, `template.tsx`, or `error.tsx` in the same segment
- Use route groups `(folder)` to scope `loading.tsx` to specific pages:

```
app/dashboard/
  ├── (overview)/       ← route group — "(overview)" doesn't appear in URL
  │   ├── loading.tsx   ← only applies to the overview page
  │   └── page.tsx      ← URL: /dashboard
  ├── invoices/
  │   └── page.tsx      ← URL: /dashboard/invoices — no loading.tsx here
  └── layout.tsx
```

> **Next.js Learn (Vercel):** *"Route groups allow you to organize files into logical groups without affecting the URL path structure. When you create a new folder using parentheses `()`, the name won't be included in the URL path."*

#### 2. Component-Level `<Suspense>` — Granular Streaming

For finer control, wrap individual async Server Components in `<Suspense>`:

```tsx
// app/dashboard/page.tsx (Server Component)
import { Suspense } from 'react'
import {
  RevenueChartSkeleton,
  LatestInvoicesSkeleton,
  CardsSkeleton,
} from '@/components/skeletons'

export default async function DashboardPage() {
  return (
    <main>
      <h1>Dashboard</h1>

      {/* Cards load together as a group */}
      <Suspense fallback={<CardsSkeleton />}>
        <CardWrapper />
      </Suspense>

      <div className="grid grid-cols-2 gap-6 mt-6">
        {/* These two stream independently — whichever resolves first appears first */}
        <Suspense fallback={<RevenueChartSkeleton />}>
          <RevenueChart />
        </Suspense>

        <Suspense fallback={<LatestInvoicesSkeleton />}>
          <LatestInvoices />
        </Suspense>
      </div>
    </main>
  )
}
```

```tsx
// components/revenue-chart.tsx (async Server Component)
// The data fetching lives INSIDE the component, not in the page
export default async function RevenueChart() {
  const revenue = await fetchRevenue() // takes 3 seconds

  return (
    <div>
      {/* render chart */}
    </div>
  )
}
```

> **Next.js Learn (Vercel):** *"By moving data fetching down to the components that need it, you can create more granular Suspense Boundaries. This allows you to stream specific components and prevent the UI from blocking."*

**The streaming timeline:**

```
T0:   Browser requests /dashboard
T0:   Server sends static shell:
        <h1>Dashboard</h1>
        <CardsSkeleton />
        <RevenueChartSkeleton />
        <LatestInvoicesSkeleton />
      ↑ User sees this IMMEDIATELY

T200ms: CardWrapper resolves → streams <CardWrapper /> to replace <CardsSkeleton />
T500ms: LatestInvoices resolves → streams to replace <LatestInvoicesSkeleton />
T3s:    RevenueChart resolves → streams to replace <RevenueChartSkeleton />
      ↑ Each piece appears independently as it becomes ready
```

#### 3. Grouping Components — Controlling Load Order

Wrap multiple related components in a single `<Suspense>` boundary so they appear together:

```tsx
// All 4 cards appear at once (not one-by-one)
<Suspense fallback={<CardsSkeleton />}>
  <CardWrapper />
</Suspense>

// CardWrapper fetches data and renders all cards
async function CardWrapper() {
  const { paid, pending, invoices, customers } = await fetchCardData()

  return (
    <>
      <Card title="Collected" value={paid} />
      <Card title="Pending" value={pending} />
      <Card title="Total Invoices" value={invoices} />
      <Card title="Total Customers" value={customers} />
    </>
  )
}
```

> **React Docs:** *"By default, the whole tree inside Suspense is treated as a single unit. Even if only one of these components suspends waiting for some data, all of them together will be replaced by the loading indicator."*

#### 4. Nesting Suspense — Staggered Loading Sequence

Each Suspense boundary is independent. Inner boundaries don't block outer ones:

```tsx
<Suspense fallback={<BigSpinner />}>         {/* Outermost boundary */}
  <Header />                                   {/* Must load for anything to show */}
  <Suspense fallback={<SidebarSkeleton />}>   {/* Independent */}
    <Sidebar />
  </Suspense>
  <Suspense fallback={<ContentSkeleton />}>   {/* Independent */}
    <MainContent />
  </Suspense>
</Suspense>
```

> **React Docs:** *"Suspense boundaries let you coordinate which parts of your UI should always 'pop in' together, and which parts should progressively reveal more content in a sequence of loading states."*

**The resolution sequence:**
1. If `Header` hasn't loaded → `<BigSpinner />` replaces the entire tree
2. Once `Header` loads → Header shows, `<SidebarSkeleton />` and `<ContentSkeleton />` show
3. `Sidebar` resolves → replaces `<SidebarSkeleton />`
4. `MainContent` resolves → replaces `<ContentSkeleton />`

Steps 3 and 4 happen independently — whichever resolves first appears first.

---

### Client-Side Suspense in Next.js 16+

#### 1. With TanStack Query (`useSuspenseQuery`)

The most common client-side Suspense pattern. `useSuspenseQuery` throws a promise while loading → Suspense catches it → shows fallback → data resolves → component renders.

```tsx
// Parent — provides the Suspense boundary
import { Suspense } from 'react'
import { ErrorBoundary } from 'react-error-boundary'

export function UserSection({ userId }: { userId: string }) {
  return (
    <ErrorBoundary fallback={<div>Failed to load user</div>}>
      <Suspense fallback={<UserSkeleton />}>
        <UserProfile userId={userId} />
      </Suspense>
    </ErrorBoundary>
  )
}

// Child — uses useSuspenseQuery (data is guaranteed defined)
'use client'
import { useSuspenseQuery } from '@tanstack/react-query'

function UserProfile({ userId }: { userId: string }) {
  const { data: user } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  })

  // No loading check needed — Suspense handles it
  // No undefined check — data is guaranteed
  return <div>{user.name}</div>
}
```

#### 2. With React `use()` Hook

React 19+ provides the `use()` hook to read Promises inside components. When the Promise hasn't resolved, the component suspends.

```tsx
// Server Component — starts the fetch, passes promise down
import { Suspense } from 'react'

export default function Page() {
  // Start fetch but DON'T await — pass the promise
  const userPromise = fetchUser()

  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  )
}
```

```tsx
// Client Component — resolves the promise with use()
'use client'
import { use } from 'react'

function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise) // suspends until resolved
  return <div>{user.name}</div>
}
```

> **React Docs:** *"Reading the value of a cached Promise with `use` is Suspense-enabled."*

#### 3. With `React.lazy()` — Code Splitting

Lazy-load components so their JavaScript bundles load on demand:

```tsx
'use client'
import { lazy, Suspense } from 'react'

// Component code is NOT included in the initial bundle
// It loads only when <HeavyChart /> first renders
const HeavyChart = lazy(() => import('./HeavyChart'))

export function Dashboard() {
  const [showChart, setShowChart] = useState(false)

  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Chart</button>

      {showChart && (
        <Suspense fallback={<div>Loading chart module...</div>}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  )
}
```

#### 4. Preventing Visible Content From Hiding — `startTransition`

When a **state change** causes a component to suspend, Suspense would normally replace the visible content with the fallback. This is jarring. Wrap the state update in `startTransition` to keep showing the old content:

```tsx
'use client'
import { useState, useTransition, Suspense } from 'react'

function Tabs() {
  const [tab, setTab] = useState('home')
  const [isPending, startTransition] = useTransition()

  function selectTab(nextTab: string) {
    // Without startTransition: current tab content vanishes, spinner shows
    // WITH startTransition: current tab stays visible (slightly dimmed) until new tab is ready
    startTransition(() => {
      setTab(nextTab)
    })
  }

  return (
    <>
      <nav style={{ opacity: isPending ? 0.7 : 1 }}>
        <button onClick={() => selectTab('home')}>Home</button>
        <button onClick={() => selectTab('posts')}>Posts</button>
        <button onClick={() => selectTab('contact')}>Contact</button>
      </nav>

      <Suspense fallback={<TabSkeleton />}>
        <TabContent tab={tab} />
      </Suspense>
    </>
  )
}
```

> **React Docs:** *"This tells React that the state transition is not urgent, and it's better to keep showing the previous page instead of hiding any already revealed content."*

This is exactly what Next.js `<Link>` does internally — route transitions are wrapped in `startTransition`, which is why navigating between pages doesn't flash a blank screen.

#### 5. Showing Stale Content While Fresh Loads — `useDeferredValue`

Similar to TanStack Query's `keepPreviousData`, but built into React:

```tsx
'use client'
import { useState, useDeferredValue, Suspense } from 'react'

function SearchPage() {
  const [query, setQuery] = useState('')
  const deferredQuery = useDeferredValue(query)
  const isStale = query !== deferredQuery

  return (
    <>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />

      <Suspense fallback={<SearchSkeleton />}>
        <div style={{ opacity: isStale ? 0.5 : 1 }}>
          <SearchResults query={deferredQuery} />
        </div>
      </Suspense>
    </>
  )
}
```

The input updates immediately (`query`), but `SearchResults` shows old results (dimmed) until the new results are ready.

#### 6. Resetting Suspense Boundaries with `key`

When navigating between similar pages (e.g., user profiles), React might reuse the component tree. Add a `key` to force a fresh Suspense cycle:

```tsx
// Without key: navigating user/1 → user/2 might show user 1's data briefly
<ProfilePage />

// With key: React treats each user as a completely different tree
<ProfilePage key={userId} />
```

> **React Docs:** *"Specify a key to tell React that different content is being rendered."*

---

### Suspense Gotchas & Caveats

#### 1. State Is Lost on First Suspend

> **React Docs:** *"React does not preserve any state for renders that got suspended before they were able to mount for the first time. When the component has loaded, React will retry rendering the suspended tree from scratch."*

If a component suspends on its **first** render, any `useState` values set during that render are lost. React re-runs the component from scratch.

#### 2. Layout Effects Are Cleaned Up When Content Hides

If already-visible content suspends again (without `startTransition`), React cleans up `useLayoutEffect`. When content reappears, layout effects re-fire. This can cause flickering with DOM measurements.

#### 3. `loading.tsx` Doesn't Wrap the Layout

`loading.tsx` wraps `page.tsx` and children — NOT the `layout.tsx` in the same segment. If your layout has slow data fetching, the loading skeleton won't cover it.

**Gotcha with runtime data in layouts:**

```tsx
// ❌ layout.tsx fetches cookies — blocks the entire navigation
export default async function Layout({ children }) {
  const theme = (await cookies()).get('theme')  // blocks instant loading
  return <div data-theme={theme}>{children}</div>
}

// ✅ Move runtime data into its own Suspense boundary
export default function Layout({ children }) {
  return (
    <div>
      <Suspense fallback={<div>Loading theme...</div>}>
        <ThemeProvider />
      </Suspense>
      {children}
    </div>
  )
}
```

> **Next.js Docs:** *"If the layout accesses uncached or runtime data (e.g. `cookies()`, `headers()`), `loading.js` will not show a fallback for it."*

#### 4. Streaming Status Code Is Always 200

Once streaming starts (first Suspense fallback renders), the HTTP response headers are already sent with status `200`. If a streamed component later throws an error, the status code can't change.

> **Next.js Docs:** *"The server can still communicate errors or issues to the client within the streamed content itself. Because the response headers have already been sent to the client, the status code of the response cannot be updated."*

#### 5. Browser Buffering

> **Next.js Docs:** *"Some browsers buffer a streaming response. You may not see the streamed response until the response exceeds 1024 bytes."*

In development with tiny pages, you might not see streaming behavior. Real apps exceed this threshold naturally.

#### 6. If `fallback` Itself Suspends

> **React Docs:** *"If fallback suspends while rendering, it will activate the closest parent Suspense boundary."*

If your skeleton component itself triggers Suspense (e.g., it lazy-loads), the PARENT Suspense boundary takes over. Keep fallbacks synchronous and lightweight.

---

### When to Use Each Suspense Pattern

| Pattern | Where | When |
|---|---|---|
| `loading.tsx` | Server | Page-level loading state, instant navigation |
| `<Suspense>` around async Server Component | Server | Granular streaming of independent sections |
| `<Suspense>` + `useSuspenseQuery` | Client | TanStack Query data with declarative loading |
| `<Suspense>` + `use(promise)` | Client | Reading server-initiated promises |
| `<Suspense>` + `React.lazy()` | Client | Code-splitting heavy components |
| `startTransition` + Suspense | Client | Preventing visible content from hiding during updates |
| `useDeferredValue` + Suspense | Client | Showing stale content while fresh loads (search, filters) |
| `key` prop on Suspense subtree | Both | Force fresh loading state on navigation between similar pages |

---

### Docs Referenced
- **React Suspense Reference:** https://react.dev/reference/react/Suspense
- **Next.js Loading UI & Streaming:** https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming
- **Next.js `loading.js` API:** https://nextjs.org/docs/app/api-reference/file-conventions/loading
- **Next.js Learn — Streaming:** https://nextjs.org/learn/dashboard-app/streaming

---

## 17. Deep Dive Q&A
### Q: Why does a shared QueryClient leak data between requests on the server?

#### The Root Cause: Node.js Is a Long-Lived Process

Unlike PHP where each request spawns a fresh process that dies after responding, Node.js keeps a **single process** alive that serves **all** users:

```
PHP model:
  Request 1 → [spawn process] → [handle] → [process dies] ← memory wiped
  Request 2 → [spawn process] → [handle] → [process dies] ← memory wiped

Node.js model:
  [server starts] ──────────────────────────────────────────→ [runs forever]
       ↑ Request 1          ↑ Request 2          ↑ Request 3
       All share the SAME process memory
```

A `QueryClient` is just a JavaScript object with an internal `Map`. It has **zero awareness** of HTTP requests, cookies, sessions, or which user is asking. It's application-level code, not framework middleware.

```tsx
// Simplified — this is roughly what QueryClient is internally:
class QueryClient {
  cache = new Map()  // just a plain Map in memory

  prefetchQuery({ queryKey, queryFn }) {
    const key = serialize(queryKey)
    if (this.cache.has(key) && !this.isStale(key)) {
      return // "already have it, skip" ← THE PROBLEM
    }
    const data = await queryFn()
    this.cache.set(key, data)
  }
}
```

#### The Leak Scenario — Step by Step

If you create a `QueryClient` at module scope:

```tsx
// ❌ THIS IS A SINGLETON — lives as long as the server runs
const queryClient = new QueryClient()

export default async function Page() {
  await queryClient.prefetchQuery({
    queryKey: ['user'],
    queryFn: () => fetchUser(), // fetches based on cookie/session
  })
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <UserDashboard />
    </HydrationBoundary>
  )
}
```

Timeline:

```
Time    Event                           queryClient cache state
─────   ─────                           ───────────────────────
T1      User A (Alice) hits /dashboard  cache: {}
T2      prefetchQuery runs for Alice    cache: { ['user'] → Alice's profile, SSN, email }

T3      User B (Bob) hits /dashboard    cache: { ['user'] → Alice's profile, SSN, email }
T4      prefetchQuery checks cache...
        staleTime hasn't passed yet...
        TanStack says: "I already have
        ['user'] cached and it's fresh,
        skip the fetch"                 cache: { ['user'] → Alice's profile } ← NOT BOB'S

T5      dehydrate(queryClient) sends
        Alice's data to Bob's browser   💀 DATA LEAK — Bob sees Alice's private data
```

#### Even With userId in the Key, You're Still Unsafe

```tsx
await queryClient.prefetchQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
})
```

Now Alice and Bob get separate cache entries, but the cache **accumulates forever**:

```
After 10,000 requests over a week:
cache: {
  ['user', 'alice']   → Alice's full profile with PII
  ['user', 'bob']     → Bob's private data
  ['user', 'charlie'] → Charlie's sensitive info
  ... 9,997 more entries sitting in server memory ...
}
```

This is both a **memory leak** (unbounded growth, eventual OOM) and a **security risk** (sensitive data persisted indefinitely in server memory).

#### The Fix

```tsx
export function getQueryClient() {
  if (isServer) {
    return new QueryClient() // FRESH for every call — isolated per request
  }
  if (!browserQueryClient) browserQueryClient = new QueryClient()
  return browserQueryClient
}
```

```
Request 1 (Alice):  new QueryClient() → prefetch Alice → dehydrate → send HTML → GC'd ✓
Request 2 (Bob):    new QueryClient() → prefetch Bob   → dehydrate → send HTML → GC'd ✓
```

Each request's QueryClient is garbage collected when the request handler finishes. No data persists between requests.

#### Why Reusing on the Client Is Safe

On the client (browser), a singleton QueryClient is correct because:
1. **One user per browser tab** — there's nobody else to leak to
2. **You WANT persistence** — navigating `/posts` → `/settings` → `/posts` should use cached posts
3. **The cache IS the feature** — that's why you're using TanStack Query client-side

---

### Q: What is HydrationBoundary, how does it work, and why should I use it per page.tsx?

#### What It Is

`HydrationBoundary` is a React component from TanStack Query that acts as a **one-way data bridge** from Server Components to Client Components. It takes serialized (dehydrated) query cache and injects it into the client's `QueryClient` during hydration.

#### How It Works — The Full Lifecycle

**Step 1: Server-Side — Prefetch and Dehydrate**

```tsx
// app/posts/page.tsx (Server Component)
export default async function PostsPage() {
  // 1. Create a fresh QueryClient for this request
  const queryClient = getQueryClient()

  // 2. Prefetch — runs queryFn, stores result in queryClient's internal cache
  await queryClient.prefetchQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  })

  // 3. At this point, queryClient's cache looks like:
  // Map { '["posts"]' → { data: [...posts], status: 'success', dataUpdatedAt: 1711100000 } }

  // 4. dehydrate() converts that Map into a plain JSON-serializable object
  const dehydratedState = dehydrate(queryClient)
  // dehydratedState = {
  //   mutations: [],
  //   queries: [{
  //     queryKey: ['posts'],
  //     queryHash: '["posts"]',
  //     state: {
  //       data: [{ id: 1, title: 'Hello' }, ...],  ← your actual data
  //       status: 'success',
  //       dataUpdatedAt: 1711100000,                ← timestamp for staleTime calculation
  //       error: null,
  //       fetchStatus: 'idle',
  //     }
  //   }]
  // }

  // 5. HydrationBoundary serializes this into the HTML as a <script> tag / RSC payload
  return (
    <HydrationBoundary state={dehydratedState}>
      <PostList />
    </HydrationBoundary>
  )
}
```

**Step 2: The Wire — What Gets Sent**

The dehydrated state travels to the client as part of the RSC (React Server Components) payload. React serializes it alongside the component tree. It's essentially JSON embedded in the response.

**Step 3: Client-Side — Hydration**

```tsx
// When <HydrationBoundary> mounts on the client, it:
// 1. Reads the `state` prop (the dehydrated cache)
// 2. Calls queryClient.hydrate(state) internally
// 3. This MERGES the dehydrated entries into the browser's QueryClient cache
//
// Now the browser's QueryClient has:
// Map { '["posts"]' → { data: [...posts], status: 'success', dataUpdatedAt: 1711100000 } }
```

```tsx
// app/posts/post-list.tsx (Client Component)
'use client'

export function PostList() {
  const { data: posts } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  })

  // useQuery checks the cache → finds ['posts'] → data is there!
  // isPending = false, data = [...posts]
  // No loading spinner, no flash of empty content
  //
  // If staleTime hasn't elapsed since dataUpdatedAt:
  //   → data is "fresh", no refetch triggered
  // If staleTime HAS elapsed:
  //   → data is shown immediately BUT a background refetch fires
  //   → isFetching = true, data = stale data (shown to user)
  //   → when refetch completes, data updates seamlessly
}
```

#### The Dehydrate/Hydrate Mental Model

Think of it like freeze-drying food:

```
SERVER                              WIRE                              CLIENT
┌─────────────────┐                                        ┌─────────────────┐
│ QueryClient     │    dehydrate()     JSON blob            │ QueryClient     │
│ (live cache     │ ──────────────→ { queries: [...] } ──→  │ (empty cache    │
│  with data)     │    serialize       (travels in          │  initially)     │
│                 │                     RSC payload)        │                 │
└─────────────────┘                                        └─────────────────┘
                                                                    │
                                                            hydrate(state)
                                                                    │
                                                                    ▼
                                                           ┌─────────────────┐
                                                           │ QueryClient     │
                                                           │ (cache now has  │
                                                           │  the same data) │
                                                           └─────────────────┘
```

#### Why Per page.tsx?

Each `page.tsx` knows what data its children need. By prefetching at the page level:

1. **No loading spinners on initial page load** — data is already in cache when Client Components mount
2. **Server can access secrets** — your `queryFn` on the server can use `process.env.API_SECRET`, database connections, internal APIs — things the browser can't reach
3. **Parallel prefetching** — you can `Promise.all` multiple prefetches at the page level
4. **Colocated data requirements** — each page declares its own data dependencies, easy to reason about

```tsx
// app/dashboard/page.tsx — colocated data requirements
export default async function DashboardPage() {
  const queryClient = getQueryClient()

  // This page needs these 3 pieces of data — declare them here
  await Promise.all([
    queryClient.prefetchQuery(userQueryOptions()),
    queryClient.prefetchQuery(statsQueryOptions()),
    queryClient.prefetchQuery(notificationsQueryOptions()),
  ])

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <DashboardHeader />    {/* uses useUser() — finds cached data */}
      <StatsPanel />          {/* uses useStats() — finds cached data */}
      <NotificationList />    {/* uses useNotifications() — finds cached data */}
    </HydrationBoundary>
  )
}
```

If you DON'T prefetch at the page level, your Client Components will:
1. Mount with `isPending: true`
2. Show loading spinners
3. Fire client-side fetch requests
4. Wait for responses
5. THEN show data

That's the "SPA waterfall" you're trying to avoid.

#### Can I Nest HydrationBoundaries?

Yes. Each `HydrationBoundary` **merges** its state into the existing cache. They don't overwrite each other:

```tsx
// Outer boundary adds ['user'] to cache
<HydrationBoundary state={dehydrate(outerQueryClient)}>
  <UserHeader />
  {/* Inner boundary adds ['stats'] to cache — ['user'] still there */}
  <HydrationBoundary state={dehydrate(innerQueryClient)}>
    <StatsPanel />
  </HydrationBoundary>
</HydrationBoundary>
```

#### Gotchas

1. **Mismatched keys = silent failure.** If server prefetches `['posts']` but client queries `['post']` (typo), no error — client just doesn't find data and fetches client-side. Use `queryOptions()` to share keys.

2. **staleTime: 0 defeats the purpose.** With default `staleTime: 0`, hydrated data is immediately stale and `useQuery` triggers a refetch on mount. Set `staleTime >= 60000` in defaultOptions.

3. **Non-serializable data will crash.** `dehydrate()` uses JSON serialization. If your `queryFn` returns a `Date` object, `Map`, `Set`, class instance, or anything non-JSON-serializable, it will be lost or throw.

```tsx
// ❌ Date objects become strings after dehydration
queryFn: async () => {
  const posts = await fetchPosts()
  return posts.map(p => ({ ...p, createdAt: new Date(p.createdAt) }))
  // After dehydrate → hydrate: createdAt is a STRING, not a Date
}

// ✅ Keep as strings, parse on the client if needed
queryFn: async () => fetchPosts() // return raw API response
```

4. **dehydrate only includes successful and pending queries by default.** Failed queries are NOT dehydrated unless you configure `shouldDehydrateQuery`. This means a failed prefetch silently disappears — the client sees `isPending: true` and refetches.

---

### Q: What happens with `void queryClient.prefetchQuery()` vs `await queryClient.prefetchQuery()`?

This is a critical difference that changes the entire rendering behavior.

#### `await` — Blocking Prefetch (Most Common)

```tsx
export default async function PostsPage() {
  const queryClient = getQueryClient()

  // ⏳ Page rendering PAUSES here until data arrives
  await queryClient.prefetchQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts, // takes 500ms
  })

  // ✅ By this point, cache has the data
  // dehydrate() captures it
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostList /> {/* Client sees data INSTANTLY — no loading state */}
    </HydrationBoundary>
  )
}
```

**Timeline:**
```
T0:   Browser requests /posts
T0:   Server starts rendering PostsPage
T0:   prefetchQuery fires, server waits...
T500: fetchPosts resolves, data stored in cache
T500: dehydrate() captures the data ← data IS in the dehydrated state
T500: HTML sent to browser with embedded data
T500: Client mounts PostList, useQuery finds cached data
      → isPending: false, data: [...posts] ← NO loading spinner
```

**Use when:** You want guaranteed data on first paint. Good for SEO-critical content, above-the-fold data.

#### `void` — Non-Blocking (Fire and Forget) Prefetch

```tsx
export default async function PostsPage() {
  const queryClient = getQueryClient()

  // 🔥 Fire the fetch but DON'T wait for it
  void queryClient.prefetchQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts, // takes 500ms
  })

  // ⚠️ Rendering continues IMMEDIATELY — data probably NOT in cache yet
  // dehydrate() runs NOW, before the fetch completes
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostList /> {/* Client will see isPending: true initially */}
    </HydrationBoundary>
  )
}
```

**Timeline:**
```
T0:   Browser requests /posts
T0:   Server starts rendering PostsPage
T0:   prefetchQuery fires (in background)
T0:   dehydrate() runs IMMEDIATELY ← data is NOT in cache yet!
T0:   HTML sent to browser WITHOUT the posts data
T0:   Client mounts PostList, useQuery finds NO cached data
      → isPending: true ← LOADING SPINNER shown
T0:   useQuery fires its own client-side fetch (since no cached data)
T500: Client-side fetch completes, data shown
```

**But wait — there's a nuance with `shouldDehydrateQuery` and pending queries.**

If you configured your QueryClient with:

```tsx
dehydrate: {
  shouldDehydrateQuery: (query) =>
    defaultShouldDehydrateQuery(query) || query.state.status === 'pending',
}
```

Then `void` behaves differently! The **pending** query IS dehydrated:

```
T0:   prefetchQuery fires (status: 'pending')
T0:   dehydrate() captures the PENDING query (no data, but the key and status)
T0:   HTML sent with the pending query state
T0:   Client hydrates → useQuery finds a pending query
      → It knows a fetch was started, so it starts its own fetch
      → isPending: true, but the queryKey is already "known"
```

This is still worse than `await` (user sees a loading state), but slightly better than nothing (the client knows what query to fire).

#### When Would You Actually Use `void`?

**For non-critical, below-the-fold data that shouldn't block the page:**

```tsx
export default async function DashboardPage() {
  const queryClient = getQueryClient()

  // CRITICAL — above the fold, must be in initial HTML
  await queryClient.prefetchQuery({
    queryKey: ['user'],
    queryFn: fetchUser,
  })

  // NON-CRITICAL — below the fold, OK to load on client
  void queryClient.prefetchQuery({
    queryKey: ['activityFeed'],
    queryFn: fetchActivityFeed, // slow — 2 seconds
  })

  // Page renders fast (only waited for user data)
  // Activity feed loads client-side with a spinner
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <UserHeader />              {/* instant — data from await */}
      <Suspense fallback={<Spinner />}>
        <ActivityFeed />          {/* loading state — data wasn't awaited */}
      </Suspense>
    </HydrationBoundary>
  )
}
```

#### The Comparison Table

| | `await prefetchQuery()` | `void prefetchQuery()` |
|---|---|---|
| **Server waits for data?** | Yes | No |
| **Data in dehydrated state?** | Yes (complete) | No (or pending status only) |
| **Client sees loading state?** | No — instant data | Yes — spinner then data |
| **Time to First Byte (TTFB)** | Slower (waits for fetch) | Faster (sends HTML immediately) |
| **Time to Interactive (TTI)** | Same | Same |
| **SEO** | Good (data in HTML) | Bad (no data in HTML) |
| **Use case** | Critical/above-fold data | Nice-to-have/below-fold data |

#### Gotcha — `void` + `prefetchQuery` error handling

`prefetchQuery` already swallows errors (returns `Promise<void>`, catches internally). Using `void` on top means you have zero visibility into failures. The query silently fails on the server, and the client re-attempts client-side. This is usually fine, but be aware.

#### Gotcha — Race condition with very fast fetches

If your `void` prefetch resolves before `dehydrate()` runs (e.g., data is cached or fetch takes <1ms), the data WILL be in the dehydrated state. This makes behavior inconsistent between fast and slow APIs — sometimes data is there, sometimes not. Stick with `await` for predictable behavior.

---

### Q: When I refetch with `useSuspenseQuery`, do I see the skeleton again or the stale data?

**Stale data stays visible. The skeleton does NOT reappear.**

Suspense only triggers when there's **zero data** in the cache (`status: 'pending'`). A refetch happens while data already exists (`status: 'success'`, `fetchStatus: 'fetching'`), so `useSuspenseQuery` never throws a Promise — Suspense has nothing to catch.

```
FIRST LOAD (no cached data):
  status: 'pending'   →  useSuspenseQuery throws Promise  →  <Suspense> catches  →  skeleton shows
  ...fetch completes...
  status: 'success'   →  data renders

REFETCH (data already in cache):
  status: 'success', fetchStatus: 'fetching'  →  does NOT throw  →  stale data stays visible
  ...refetch completes...
  status: 'success', fetchStatus: 'idle'      →  new data replaces old (seamless swap)
```

#### How to Show Loading State During Refetch

**Option 1: Dim the stale data (recommended — best UX)**

```tsx
function PostsList() {
  const { data, isFetching } = useSuspenseQuery(postsQueryOptions())

  return (
    // Stale data stays visible but dimmed while refetching
    <div className={isFetching ? 'opacity-50 pointer-events-none' : ''}>
      {isFetching && <p className="text-sm text-muted">Refreshing...</p>}
      {data.map(post => <PostCard key={post.id} post={post} />)}
    </div>
  )
}
```

**Option 2: Inline spinner alongside data**

```tsx
function PostsList() {
  const { data, isRefetching } = useSuspenseQuery(postsQueryOptions())

  return (
    <div>
      <h2>Posts {isRefetching && <Spinner className="inline size-4" />}</h2>
      {data.map(post => <PostCard key={post.id} post={post} />)}
    </div>
  )
}
```

**Option 3: Force the skeleton to reappear (nuclear option)**

```tsx
// resetQueries CLEARS the cache → status goes back to 'pending'
// → useSuspenseQuery throws again → Suspense catches → skeleton shows
queryClient.resetQueries({ queryKey: ['posts'] })
```

> **⚠️ Gotcha:** `resetQueries` is different from `invalidateQueries`:
> - `invalidateQueries` → marks data as stale, triggers background refetch, **keeps data in cache** → no skeleton
> - `resetQueries` → **removes data from cache entirely** → skeleton shows again
>
> Use `resetQueries` sparingly — the jarring skeleton flash is usually worse UX than showing stale data with a spinner.

#### The Mental Model

Think of it as two separate concerns:

| Concern | Who handles it | When it fires |
|---|---|---|
| "No data yet" (initial load) | `<Suspense>` + skeleton | Only when cache is empty for this key |
| "Updating data" (refetch) | `isFetching` / `isRefetching` | Every background refetch |

This is **intentional design** — TanStack Query's philosophy is that showing stale data with a background refresh indicator is better UX than flashing a skeleton every time data refreshes. Users see content immediately and a subtle indicator that it's being updated.

---

### Q: If I'm using `<Suspense>` explicitly inside my page, do I still need `loading.tsx`?

**No.** They're two ways to achieve the same thing — `loading.tsx` is just Next.js's shortcut for wrapping your `page.tsx` in `<Suspense>`. If you're already using manual `<Suspense>` boundaries inside your page, `loading.tsx` is redundant for those sections.

#### What Each One Covers

```
loading.tsx:
  Wraps the ENTIRE page.tsx in one <Suspense>
  Shows ONE skeleton until the page component finishes rendering
  You don't control placement — Next.js decides

Manual <Suspense> inside page.tsx:
  You wrap SPECIFIC components
  Different skeletons for different sections
  Full control over what streams independently
```

#### Three Approaches

**1. Only `loading.tsx` (simple pages)**

```
app/posts/
  ├── loading.tsx    ← one skeleton for the whole page
  └── page.tsx
```

User sees: `PostsSkeleton` → entire page appears at once.

**2. Only manual `<Suspense>` (granular control — most common with TanStack Query)**

```
app/dashboard/
  └── page.tsx       ← manual <Suspense> boundaries inside
```

```tsx
export default async function DashboardPage() {
  const queryClient = getQueryClient()
  await queryClient.prefetchQuery(userQueryOptions())

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <h1>Dashboard</h1>  {/* shows immediately */}

      <Suspense fallback={<StatsSkeleton />}>
        <StatsPanel />     {/* streams when ready */}
      </Suspense>

      <Suspense fallback={<FeedSkeleton />}>
        <ActivityFeed />   {/* streams independently */}
      </Suspense>
    </HydrationBoundary>
  )
}
```

User sees: `h1` immediately + `StatsSkeleton` + `FeedSkeleton` → Stats appear → Feed appears (independently).

No `loading.tsx` needed — the manual boundaries handle everything.

**3. Both together (layered — for pages with top-level awaits)**

```
app/dashboard/
  ├── loading.tsx    ← covers the gap while page.tsx's top-level await runs
  └── page.tsx       ← manual <Suspense> for sections inside
```

```tsx
// loading.tsx — shown while page.tsx is still awaiting at the top level
export default function Loading() {
  return <FullPageSkeleton />
}

// page.tsx
export default async function DashboardPage() {
  // This top-level await BLOCKS the page from rendering at all
  // loading.tsx covers this gap
  const config = await fetchAppConfig()

  return (
    <div>
      <h1>{config.title}</h1>  {/* only renders after await completes */}

      {/* These stream independently AFTER the page starts rendering */}
      <Suspense fallback={<StatsSkeleton />}>
        <StatsPanel />
      </Suspense>

      <Suspense fallback={<FeedSkeleton />}>
        <ActivityFeed />
      </Suspense>
    </div>
  )
}
```

Timeline:
```
T0:      loading.tsx shows <FullPageSkeleton />         (page.tsx is awaiting fetchAppConfig)
T200ms:  fetchAppConfig resolves → page starts rendering
         loading.tsx replaced by: h1 + <StatsSkeleton /> + <FeedSkeleton />
T500ms:  StatsPanel resolves → replaces <StatsSkeleton />
T2s:     ActivityFeed resolves → replaces <FeedSkeleton />
```

#### When to Use Which

| Situation | Use `loading.tsx`? | Use manual `<Suspense>`? |
|---|---|---|
| Simple page, one data source | Yes (easy) | Not needed |
| Multiple independent sections that should stream | No (too coarse) | Yes |
| TanStack Query with prefetch + `HydrationBoundary` | Usually no | Yes (per-component boundaries) |
| Page has a top-level `await` before any JSX renders | Yes (covers that gap) | Also yes (for sections inside) |
| You want different skeletons for different components | No (one skeleton for entire page) | Yes |

**Rule of thumb:** If you're using TanStack Query with `HydrationBoundary` and manual `<Suspense>` boundaries inside your page, you typically don't need `loading.tsx`. The manual boundaries give you more control.

---

### Q: Which hooks should I focus on to get started?

For your use cases (data fetching, pagination, infinite scroll, optimistic updates with error rollback), here are the **6 things** you need:

#### Your Starter Kit

| Use Case | What to Use | Section |
|---|---|---|
| **Data fetching** | `useQuery()` + `queryClient.prefetchQuery()` + `HydrationBoundary` | Sections 3, 7, 10 |
| **Pagination** | `useQuery()` + `keepPreviousData` + `queryClient.prefetchQuery()` (for next page) | Section 12 |
| **Infinite scroll** | `useInfiniteQuery()` | Section 11 |
| **Optimistic updates + error rollback** | `useMutation()` with `onMutate`/`onError` + `queryClient.invalidateQueries()` | Sections 8, 9 |
| **SSR hydration** (every page) | `getQueryClient()` + `dehydrate()` + `HydrationBoundary` | Sections 2, 10 |

#### The Minimal Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│ page.tsx (Server Component)                                      │
│                                                                  │
│   const queryClient = getQueryClient()                           │
│   await queryClient.prefetchQuery(postsQueryOptions())           │
│                                                                  │
│   return (                                                       │
│     <HydrationBoundary state={dehydrate(queryClient)}>          │
│       <PostList />          ← useQuery (reads, pagination)       │
│       <InfiniteFeed />      ← useInfiniteQuery (infinite scroll) │
│       <CreatePostForm />    ← useMutation (optimistic updates)   │
│     </HydrationBoundary>                                         │
│   )                                                              │
└─────────────────────────────────────────────────────────────────┘
```

#### Complete Optimistic Update with Error Rollback + Toast

This is the pattern for "optimistic UI that shows an error if it fails":

```tsx
'use client'

import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { toast } from 'sonner' // or your toast library

export function TodoList() {
  const queryClient = useQueryClient()
  const { data: todos } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  })

  const updateTodo = useMutation({
    mutationFn: async ({ id, completed }: { id: string; completed: boolean }) => {
      const res = await fetch(`/api/todos/${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ completed }),
      })
      if (!res.ok) throw new Error('Failed to update todo')
      return res.json()
    },

    // 1. BEFORE the API call — optimistically update the UI
    onMutate: async ({ id, completed }) => {
      // Cancel outgoing refetches so they don't overwrite our optimistic update
      await queryClient.cancelQueries({ queryKey: ['todos'] })

      // Snapshot current state for rollback
      const previousTodos = queryClient.getQueryData<Todo[]>(['todos'])

      // Optimistically update cache — UI updates INSTANTLY
      queryClient.setQueryData<Todo[]>(['todos'], (old) =>
        old?.map((todo) =>
          todo.id === id ? { ...todo, completed } : todo
        )
      )

      // Return snapshot for rollback in onError
      return { previousTodos }
    },

    // 2. ON ERROR — rollback and show error toast
    onError: (error, variables, context) => {
      // Restore the previous state
      if (context?.previousTodos) {
        queryClient.setQueryData(['todos'], context.previousTodos)
      }

      // Show error to user
      toast.error(`Failed to update: ${error.message}`, {
        description: 'Your change was reverted. Please try again.',
        action: {
          label: 'Retry',
          onClick: () => updateTodo.mutate(variables), // retry with same args
        },
      })
    },

    // 3. ALWAYS — refetch to make sure cache matches server
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] })
    },
  })

  return (
    <ul>
      {todos?.map((todo) => (
        <li
          key={todo.id}
          onClick={() =>
            updateTodo.mutate({ id: todo.id, completed: !todo.completed })
          }
          style={{
            textDecoration: todo.completed ? 'line-through' : 'none',
            cursor: 'pointer',
            // Visual feedback while mutation is in flight
            opacity: updateTodo.isPending ? 0.7 : 1,
          }}
        >
          {todo.title}
        </li>
      ))}
    </ul>
  )
}
```

**The flow:**
```
User clicks todo → onMutate fires:
  ├── Cache updated instantly (UI shows ✓ immediately)
  ├── API call starts in background
  │
  ├── API succeeds? → onSettled → invalidateQueries → background refetch
  │                    (user sees no change — optimistic was correct)
  │
  └── API fails? → onError fires:
       ├── Cache rolled back to snapshot (UI reverts the ✓)
       ├── Toast appears: "Failed to update. Your change was reverted."
       └── onSettled → invalidateQueries → ensures consistency
```

---

### Q: I prefetch in a Server Component but still see loading spinners — what am I doing wrong?

This is the most common mistake when integrating TanStack Query with Next.js Server Components. Here's a real-world example:

#### The Broken Code

```tsx
// workspace-selector.tsx (Server Component)
"use server";
import { Suspense } from "react";
import { WorkspaceSelectorClient } from "./workspace-selector-client";
import { getQueryClient } from "@/lib/query-client";
import { getAllProjects } from "@/app/actions/getAllProjects";

export async function WorkspaceSelector() {
  const queryClient = getQueryClient();
  await queryClient.prefetchQuery({
    queryKey: ["all-projects"],
    queryFn: getAllProjects,
  });
  return (
    <Suspense>
      <WorkspaceSelectorClient />
    </Suspense>
  );
}
```

```tsx
// workspace-selector-client.tsx (Client Component)
"use client";
import { useQuery } from "@tanstack/react-query";

export function WorkspaceSelectorClient() {
  const { data: projects, isLoading } = useQuery<Project[]>({
    queryKey: ["all-projects"],
    queryFn: getAllProjects,
  });

  if (isLoading || !projects) {
    return <WorkspaceSelectorSkeleton />;
  }

  return <Select>...</Select>;
}
```

```tsx
// layout.tsx — wraps with Suspense
<Suspense fallback={<WorkspaceSelectorSkeleton />}>
  <WorkspaceSelector />
</Suspense>
```

#### What's wrong? THREE things.

**Problem 1: No `dehydrate()` + `HydrationBoundary` — prefetched data is thrown away**

The server prefetches data into `queryClient`, but never serializes it or sends it to the client. The data exists only in the server's memory and is garbage collected after the response. The client's `QueryClient` is empty — `useQuery` finds nothing and fires a brand new fetch.

```
Server queryClient:  { ['all-projects'] → [project1, project2, ...] }  ← DATA IS HERE
                                    ↓
                          No dehydrate() call
                          No HydrationBoundary
                                    ↓
                              DATA LOST ← never sent to client
                                    ↓
Client queryClient:  { }  ← EMPTY, useQuery fetches from scratch
```

**Problem 2: `useQuery` doesn't integrate with Suspense**

The `<Suspense>` boundary in the server component is pointless here. `useQuery` handles loading states via `isLoading`/`isPending` return values — it does NOT throw promises, so React Suspense never catches anything. The `<Suspense>` boundary just sits there doing nothing.

To integrate with Suspense, you need `useSuspenseQuery`, which throws a promise while loading (React Suspense catches it and shows the fallback).

**Problem 3: Manual `isLoading` checks defeat the purpose**

With `useQuery`, you're forced to write `if (isLoading) return <Skeleton />` in every component. This is the "old way." With `useSuspenseQuery` + `<Suspense>` boundaries, the loading state is handled declaratively by the boundary — the component only renders when data is ready.

#### The Fixed Code

```tsx
// workspace-selector.tsx (Server Component)
import { dehydrate, HydrationBoundary } from "@tanstack/react-query";
import { getQueryClient } from "@/lib/query-client";
import { getAllProjects } from "@/app/actions/getAllProjects";
import { WorkspaceSelectorClient } from "./workspace-selector-client";

export async function WorkspaceSelector() {
  const queryClient = getQueryClient();
  await queryClient.prefetchQuery({
    queryKey: ["all-projects"],
    queryFn: getAllProjects,
  });

  // dehydrate() serializes the cache → HydrationBoundary sends it to the client
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <WorkspaceSelectorClient />
    </HydrationBoundary>
  );
}
```

```tsx
// workspace-selector-client.tsx (Client Component)
"use client";
import { useSuspenseQuery } from "@tanstack/react-query";

export function WorkspaceSelectorClient() {
  // useSuspenseQuery — data is GUARANTEED defined when this renders
  // No isLoading check. No undefined check. No skeleton here.
  const { data: projects } = useSuspenseQuery<Project[]>({
    queryKey: ["all-projects"],
    queryFn: getAllProjects,
  });

  return (
    <Select onValueChange={(v) => console.log({ v })}>
      <SelectTrigger className="w-full max-w-48">
        <SelectValue placeholder={projects[0]?.name} />
      </SelectTrigger>
      <SelectContent>
        <SelectGroup>
          <SelectLabel>Projects</SelectLabel>
          {projects.map((project) => (
            <SelectItem value={project.name} key={project.id}>
              {project.name}
            </SelectItem>
          ))}
        </SelectGroup>
      </SelectContent>
    </Select>
  );
}
```

```tsx
// layout.tsx — Suspense with fallback (THIS is where the skeleton goes)
<Suspense fallback={<WorkspaceSelectorSkeleton />}>
  <WorkspaceSelector />
</Suspense>
```

#### The Fixed Flow

```
layout.tsx
  └── <Suspense fallback={<WorkspaceSelectorSkeleton />}>
        │
        └── WorkspaceSelector (Server Component)
              │
              ├── await prefetchQuery(['all-projects'])
              │     └── Data fetched on server, stored in queryClient cache
              │
              ├── dehydrate(queryClient)
              │     └── Cache serialized to JSON: { queries: [{ queryKey: ['all-projects'], state: { data: [...] } }] }
              │
              └── <HydrationBoundary state={dehydratedState}>
                    │  └── On client mount: hydrates data into browser's QueryClient
                    │
                    └── WorkspaceSelectorClient
                          │
                          └── useSuspenseQuery(['all-projects'])
                                │
                                ├── Checks client QueryClient cache
                                ├── Finds hydrated data ← PUT THERE BY HydrationBoundary
                                ├── data = [project1, project2, ...] ← INSTANTLY available
                                └── Component renders immediately, no loading state
```

#### But wait — why does it STILL work without HydrationBoundary (just slower)?

Because `useSuspenseQuery` (and `useQuery`) always has a `queryFn` as fallback. If there's no hydrated data in the cache:

1. `useSuspenseQuery` throws a promise (Suspense shows the skeleton)
2. It fires `queryFn` client-side
3. When the fetch completes, the promise resolves
4. React re-renders the component with data

So the code works either way — but **without** HydrationBoundary, you get:
- An unnecessary loading spinner on first page load
- A wasted client-side fetch (server already had this data!)
- Slower Time to Interactive

**With** HydrationBoundary, the data is already in the cache when the component mounts, so `useSuspenseQuery` returns immediately — no spinner, no extra fetch.

#### When should I use `useQuery` vs `useSuspenseQuery`?

| | `useQuery` | `useSuspenseQuery` |
|---|---|---|
| Loading state | Manual: `if (isLoading) return <Skeleton />` | Declarative: `<Suspense fallback={<Skeleton />}>` |
| `data` type | `T \| undefined` | `T` (guaranteed defined) |
| Error handling | Manual: `if (isError) return <Error />` | Declarative: `<ErrorBoundary>` |
| `enabled` option | Available (conditional fetching) | NOT available (breaks Suspense contract) |
| Where skeleton lives | Inside the component | In parent `<Suspense>` boundary |
| Best for | Conditional fetching, dependent queries | Everything else |

**Rule of thumb:** Default to `useSuspenseQuery`. Only fall back to `useQuery` when you need `enabled: false` for conditional fetching (e.g., "only fetch when user selects an item").

---

## 18. Official Doc Links
### TanStack Query v5
- **Overview:** https://tanstack.com/query/latest/docs/framework/react/overview
- **SSR Guide:** https://tanstack.com/query/latest/docs/framework/react/guides/ssr
- **Advanced SSR (Next.js App Router):** https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr
- **useQuery Reference:** https://tanstack.com/query/latest/docs/framework/react/reference/useQuery
- **useSuspenseQuery:** https://tanstack.com/query/latest/docs/framework/react/reference/useSuspenseQuery
- **useQueries:** https://tanstack.com/query/latest/docs/framework/react/reference/useQueries
- **useMutation:** https://tanstack.com/query/latest/docs/framework/react/reference/useMutation
- **useInfiniteQuery:** https://tanstack.com/query/latest/docs/framework/react/reference/useInfiniteQuery
- **HydrationBoundary:** https://tanstack.com/query/latest/docs/framework/react/reference/HydrationBoundary
- **Paginated Queries:** https://tanstack.com/query/latest/docs/framework/react/guides/paginated-queries
- **Infinite Queries:** https://tanstack.com/query/latest/docs/framework/react/guides/infinite-queries
- **Prefetching:** https://tanstack.com/query/latest/docs/framework/react/guides/prefetching
- **Mutations:** https://tanstack.com/query/latest/docs/framework/react/guides/mutations
- **Query Invalidation:** https://tanstack.com/query/latest/docs/framework/react/guides/query-invalidation
- **Suspense:** https://tanstack.com/query/latest/docs/framework/react/guides/suspense
- **Parallel Queries:** https://tanstack.com/query/latest/docs/framework/react/guides/parallel-queries

### Next.js
- **Data Fetching:** https://nextjs.org/docs/app/building-your-application/data-fetching
- **Server Actions:** https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations
- **Caching:** https://nextjs.org/docs/app/building-your-application/caching
- **`use cache` Directive:** https://nextjs.org/docs/app/api-reference/directives/use-cache
- **revalidateTag:** https://nextjs.org/docs/app/api-reference/functions/revalidateTag
- **revalidatePath:** https://nextjs.org/docs/app/api-reference/functions/revalidatePath
- **Streaming:** https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming
- **Server Components:** https://nextjs.org/docs/app/building-your-application/rendering/server-components

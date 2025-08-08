# T3 Stack, tRPC, and TanStack Query: Deep Dive & Best Practices

This document addresses common questions and best practices when integrating tRPC and TanStack Query within a T3 Stack Next.js application, with a strong focus on the server-side tRPC configuration and its implications for client-side usage and rendering strategies.

## The Core tRPC Server Configuration (`src/server/api/trpc.ts` & `src/server/api/root.ts`)

Let's break down the essential components of the provided tRPC server configuration, which is foundational for understanding how queries and mutations are defined and consumed.

```typescript
// src/server/api/trpc.ts (Central tRPC configuration)
/**
 * YOU PROBABLY DON'T NEED TO EDIT THIS FILE, UNLESS:
 * 1. You want to modify request context (see Part 1).
 * 2. You want to create a new middleware or type of procedure (see Part 3).
 *
 * TL;DR - This is where all the tRPC server stuff is created and plugged in. The pieces you will
 * need to use are documented accordingly near the end.
 */

import { initTRPC, TRPCError } from "@trpc/server";
import superjson from "superjson";
import { ZodError } from "zod";

import { auth } from "@/server/auth"; // Assumed NextAuth.js setup
import { db } from "@/server/db"; // Assumed Prisma client instance

/**
 * 1. CONTEXT
 *
 * This section defines the "contexts" that are available in the backend API.
 *
 * These allow you to access things when processing a request, like the database, the session, etc.
 *
 * This helper generates the "internals" for a tRPC context. The API handler and RSC clients each
 * wrap this and provides the required context.
 *
 * @see https://trpc.io/docs/server/context
 */
export const createTRPCContext = async (opts: { headers: Headers }) => {
  const session = await auth(); // Get the user session from NextAuth.js

  return {
    db, // Your Prisma DB client
    session, // User session data
    ...opts, // Headers, etc.
  };
};

/**
 * 2. INITIALIZATION
 *
 * This is where the tRPC API is initialized, connecting the context and transformer. We also parse
 * ZodErrors so that you get typesafety on the frontend if your procedure fails due to validation
 * errors on the backend.
 */
const t = initTRPC.context<typeof createTRPCContext>().create({
  transformer: superjson, // Handles complex data types serialization/deserialization
  errorFormatter({ shape, error }) {
    return {
      ...shape,
      data: {
        ...shape.data,
        zodError:
          error.cause instanceof ZodError ? error.cause.flatten() : null, // Flatten Zod errors for client
      },
    };
  },
});

/**
 * Create a server-side caller.
 *
 * @see https://trpc.io/docs/server/server-side-calls
 */
export const createCallerFactory = t.createCallerFactory;

/**
 * 3. ROUTER & PROCEDURE (THE IMPORTANT BIT)
 *
 * These are the pieces you use to build your tRPC API. You should import these a lot in the
 * "/src/server/api/routers" directory.
 */

/**
 * This is how you create new routers and sub-routers in your tRPC API.
 *
 * @see https://trpc.io/docs/router
 */
export const createTRPCRouter = t.router;

/**
 * Middleware for timing procedure execution and adding an artificial delay in development.
 *
 * You can remove this if you don't like it, but it can help catch unwanted waterfalls by simulating
 * network latency that would occur in production but not in local development.
 */
const timingMiddleware = t.middleware(async ({ next, path }) => {
  const start = Date.now();

  if (t._config.isDev) {
    // artificial delay in dev to simulate network latency
    const waitMs = Math.floor(Math.random() * 400) + 100;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }

  const result = await next();

  const end = Date.now();
  console.log(`[TRPC] ${path} took ${end - start}ms to execute`);

  return result;
});

/**
 * Public (unauthenticated) procedure
 *
 * This is the base piece you use to build new queries and mutations on your tRPC API. It does not
 * guarantee that a user querying is authorized, but you can still access user session data if they
 * are logged in.
 */
export const publicProcedure = t.procedure.use(timingMiddleware);

/**
 * Protected (authenticated) procedure
 *
 * If you want a query or mutation to ONLY be accessible to logged in users, use this. It verifies
 * the session is valid and guarantees `ctx.session.user` is not null.
 *
 * @see https://trpc.io/docs/procedures
 */
export const protectedProcedure = t.procedure
  .use(timingMiddleware)
  .use(({ ctx, next }) => {
    if (!ctx.session?.user) {
      throw new TRPCError({ code: "UNAUTHORIZED" }); // Throws tRPC error for client to catch
    }
    // If authenticated, infer session.user as non-nullable
    return next({
      ctx: {
        session: { ...ctx.session, user: ctx.session.user },
      },
    });
  });
```

```typescript
// src/server/api/root.ts (Root API Router - often named _app.ts)
import { createCallerFactory, createTRPCRouter } from "@/server/api/trpc";
import { todoRouter } from "./routers/todo"; // Your specific router

/**
 * This is the primary router for your server.
 * All routers added in /api/routers should be manually added here.
 */
export const appRouter = createTRPCRouter({
  todo: todoRouter, // Plug in your todo router here
});

// Export type definition of API for client-side type inference
export type AppRouter = typeof appRouter;

/**
 * Create a server-side caller for the tRPC API.
 * This is used to make direct server-to-server calls or to prefetch data in Server Components.
 * @example
 * const trpc = createCaller(createContext);
 * const res = await trpc.post.all();
 */
export const createCaller = createCallerFactory(appRouter);
```

### Explanation of Key Parts:

1.  **`createTRPCContext` (Context Definition):**

    - **Purpose:** This function defines the "context" that will be available to _every_ tRPC procedure executed on the server. Think of it as data that's relevant to the current request.
    - **Contents:** In this T3 Stack setup, it fetches the user `session` (presumably using NextAuth.js's `auth()` function) and provides access to the `db` (your Prisma client instance). It also forwards `headers` from the incoming request.
    - **Importance:** This is how your tRPC procedures get access to essential resources like the database for data operations, or user authentication status to determine access rights.

2.  **`initTRPC` (Initialization):**

    - **`t = initTRPC.context<typeof createTRPCContext>().create(...)`**: This is the core `tRPC` instance.
    - **`context<typeof createTRPCContext>()`**: Links the `tRPC` instance to your defined `createTRPCContext` function, ensuring all procedures receive the correct context types.
    - **`transformer: superjson`**: `superjson` is crucial for serializing and deserializing complex JavaScript types (like `Date`, `Map`, `Set`) that JSON doesn't natively support, allowing them to be safely passed between the server and client.
    - **`errorFormatter`**: This custom formatter intercepts errors. Specifically, it catches `ZodError` (thrown during input validation) and flattens it. This provides highly detailed, type-safe validation error messages to the client, which is incredibly helpful for displaying form errors.

3.  **`createCallerFactory`:**

    - **Purpose:** This function generates a "caller" that allows you to make direct, type-safe tRPC calls _from within your server-side code_ (e.g., in `getServerSideProps`, Server Components, or API routes).
    - **Usage:** You typically combine it with `createTRPCContext` to create an instance for server-side fetching: `const trpc = createCaller(createTRPCContext);`

4.  **`createTRPCRouter` (Router Builder):**

    - **Purpose:** This is the function you use to define your API routers. Each router acts as a namespace for a group of related procedures (queries/mutations).
    - **Usage:** As seen in `src/server/api/root.ts`, you use it to create `appRouter` and sub-routers like `todoRouter`.

5.  **`timingMiddleware` (Middleware Example):**

    - **Purpose:** This is a custom tRPC middleware that runs before and after your procedures.
    - **Functionality:**
      - **Logging:** It logs the execution time of each procedure.
      - **Artificial Delay (Dev):** Crucially, in development (`t._config.isDev`), it introduces a random artificial delay.
    - **Benefit:** This delay helps simulate real-world network latency that wouldn't be present in a local development environment. It's excellent for identifying and preventing "waterfalls" (where a subsequent request depends on the completion of a previous one, leading to slow load times) early in development. You might remove this in production or if it interferes with specific testing.

6.  **`publicProcedure` (Public Procedure Builder):**

    - **Purpose:** This is the base procedure builder for endpoints that do _not_ require authentication.
    - **Middleware:** It uses the `timingMiddleware`.
    - **Authentication:** While it doesn't enforce authentication, the `ctx.session` is still available if a user is logged in, allowing for personalized experiences even for public data.

7.  **`protectedProcedure` (Protected Procedure Builder):**
    - **Purpose:** This is the procedure builder for endpoints that _require_ an authenticated user.
    - **Middleware Chain:** It uses `timingMiddleware` and then adds another middleware:
      - **Authentication Check:** It checks `ctx.session?.user`. If the user is not logged in (`ctx.session?.user` is null), it throws an `TRPCError` with code `"UNAUTHORIZED"`. This error is propagated to the client.
      - **Type Inference:** If authentication passes, it tells TypeScript that `ctx.session.user` is now guaranteed to be non-nullable, improving type safety within the protected procedure's logic.
    - **Security:** This is your go-to for sensitive operations, ensuring only authenticated users can access them.

---

## 1. Why `HydrateClient` in `page.tsx` and Not `layout.tsx`? Should it be in every `page.tsx`?

**Q:** Given its purpose to hydrate client components with server-fetched data, why isn't the `QueryClientProvider` and associated hydration logic placed in the root `layout.tsx`? Should I include it in every `page.tsx` if I want to hydrate client components without passing data as props?

**A:** This is a crucial distinction related to Next.js App Router's architecture and TanStack Query's caching mechanisms, now further clarified by the `TRPCReactProvider` in the provided tRPC client setup.

**Understanding the T3 Stack App Router Setup:**

In a typical T3 Stack Next.js App Router project:

- The **`QueryClientProvider`** (and thus the `TRPCReactProvider` from `src/trpc/react.tsx`) is usually placed in your **root `layout.tsx`**. This provides a single, consistent `QueryClient` instance for your entire client-side application.
- The **`Hydrate` component** (from `@tanstack/react-query/hydration`) is what then receives the specific server-dehydrated state **per-page or per-route segment**.

**Why this pattern?**

1.  **Global Cache Management (`QueryClientProvider` in `layout.tsx`):**

    - Placing `TRPCReactProvider` (which internally includes `QueryClientProvider`) in `layout.tsx` ensures that your client-side application maintains a _single, consistent `QueryClient` cache_. This is generally desirable for TanStack Query, as it allows queries to share data, prevents unnecessary re-fetches, and enables global invalidation strategies.
    - `layout.tsx` generally does **not re-render on soft navigations**. This is perfect for the `QueryClientProvider`, as you want its internal state (the cache) to persist across pages.

2.  **Page-Specific Data Hydration (`Hydrate` in `page.tsx` or wrapped components):**
    - **Purpose:** While the `QueryClientProvider` is global, the `Hydrate` component's role is to inject server-fetched data _into that global cache_ specifically for the current page load.
    - **Behavior:** When you fetch data in a Server Component (`page.tsx`) using `api.createContext().queryClient.prefetchQuery()` and then `dehydrate` it, the resulting `dehydratedState` is passed to a `<Hydrate state={dehydratedState}>` component within that `page.tsx`. This ensures that:
      - The HTML for the page includes the pre-fetched data from the server.
      - When the client-side JavaScript runs, the `QueryClient`'s cache is populated with _this specific page's data_ before your `useQuery` hooks even execute.
      - This avoids an immediate client-side re-fetch and prevents a loading spinner, providing a seamless SSR experience.

**Should you put `<Hydrate>` in every `page.tsx`?**

- **Yes, for pages that perform server-side data fetching for client components:** If a `page.tsx` (Server Component) fetches data using the server-side tRPC caller and you want client components within that page to automatically pick up this pre-fetched data via `api.useQuery()`, then you should `dehydrate` the `QueryClient` state in that `page.tsx` and pass it to a `<Hydrate>` component wrapping your client components.
- **No, for pages that _only_ render server-side:** If a page is purely a Server Component and doesn't pass data to `api.useQuery()` in any child Client Components (i.e., it only uses `api.server.someRouter.someQuery.query()` directly), then `Hydrate` is not needed.

**Code Example (Revisited with tRPC Config in Mind):**

```tsx
// src/app/layout.tsx (Root Layout - Global QueryClientProvider)
import "~/styles/globals.css";
import { GeistSans } from "geist/font/sans";
import { TRPCReactProvider } from "~/trpc/react"; // This imports and wraps children with QueryClientProvider

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en" className={`${GeistSans.variable}`}>
      <body>
        {/* TRPCReactProvider initializes the client-side tRPC instance and QueryClientProvider */}
        <TRPCReactProvider>{children}</TRPCReactProvider>
      </body>
    </html>
  );
}
```

```tsx
// src/app/page.tsx (Server Component - Prefetches and Hydrates Page-Specific Data)
import { api } from "~/trpc/server"; // Server-side tRPC caller (uses createCaller from root.ts)
import { Hydrate, dehydrate } from "@tanstack/react-query";
import ClientTodoListComponent from "./_components/ClientTodoListComponent"; // A client component

export default async function HomePage() {
  // 1. Create a fresh QueryClient instance for THIS server request
  const queryClient = api.createContext().queryClient;

  // 2. Prefetch data using the server-side tRPC caller
  await queryClient.prefetchQuery(api.todo.getAll.getQueryKey(), () =>
    api.todo.getAll.query()
  );

  // 3. Dehydrate the state of this specific QueryClient instance
  const dehydratedState = dehydrate(queryClient);

  return (
    <main>
      <h1 className="text-3xl font-bold text-white mb-4">My Todos</h1>
      {/* 4. Hydrate the client with the server-fetched data */}
      <Hydrate state={dehydratedState}>
        <ClientTodoListComponent />{" "}
        {/* Client component that uses `api.todo.getAll.useQuery()` */}
      </Hydrate>
    </main>
  );
}
```

```tsx
// src/app/_components/ClientTodoListComponent.tsx (Client Component - Consumes Hydrated Data)
"use client";

import { api } from "~/trpc/react"; // Client-side tRPC hooks (from createTRPCReact)

export default function ClientTodoListComponent() {
  // TanStack Query automatically finds pre-fetched data in the cache from Hydrate
  const { data: todos, isLoading, isError, error } = api.todo.getAll.useQuery();

  // --- Example Mutation ---
  const addTodoMutation = api.todo.create.useMutation({
    onSuccess: () => {
      // Invalidate the 'getAll' query to refetch and update UI after successful creation
      api.todo.getAll.invalidate();
    },
    onError: (err) => {
      alert(`Error creating todo: ${err.message}`);
      // Zod errors are flattened from server-side config
      if (err.data?.zodError) {
        console.error("Validation errors:", err.data.zodError);
      }
    },
  });

  const handleDelete = api.todo.delete.useMutation({
    onSuccess: () => {
      api.todo.getAll.invalidate();
    },
  });

  const handleToggleDone = api.todo.toggleDone.useMutation({
    onSuccess: () => {
      api.todo.getAll.invalidate();
    },
  });

  const [newTodoText, setNewTodoText] = useState("");

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (newTodoText.trim()) {
      addTodoMutation.mutate({ text: newTodoText });
      setNewTodoText("");
    }
  };

  if (isLoading) return <p className="text-white">Loading todos...</p>;
  if (isError) return <p className="text-red-500">Error: {error.message}</p>;

  return (
    <div className="bg-gray-800 p-6 rounded-lg shadow-md">
      <form onSubmit={handleSubmit} className="flex gap-2 mb-4">
        <input
          type="text"
          value={newTodoText}
          onChange={(e) => setNewTodoText(e.target.value)}
          placeholder="Add a new todo"
          className="flex-grow rounded-md p-2 text-black"
          disabled={addTodoMutation.isLoading}
        />
        <button
          type="submit"
          className="bg-blue-500 hover:bg-blue-600 text-white p-2 rounded-md"
          disabled={addTodoMutation.isLoading}
        >
          {addTodoMutation.isLoading ? "Adding..." : "Add Todo"}
        </button>
      </form>

      {todos?.length === 0 ? (
        <p className="text-gray-400">No todos yet. Add one!</p>
      ) : (
        <ul className="space-y-2">
          {todos?.map((todo) => (
            <li
              key={todo.id}
              className="flex items-center justify-between bg-gray-700 p-3 rounded-md"
            >
              <span
                className={`text-white ${
                  todo.done ? "line-through text-gray-400" : ""
                }`}
              >
                {todo.text}
              </span>
              <div className="flex gap-2">
                <button
                  onClick={() =>
                    handleToggleDone.mutate({ id: todo.id, done: !todo.done })
                  }
                  className={`px-3 py-1 rounded-md text-sm ${
                    todo.done
                      ? "bg-yellow-500 hover:bg-yellow-600"
                      : "bg-green-500 hover:bg-green-600"
                  } text-white`}
                >
                  {todo.done ? "Undo" : "Done"}
                </button>
                <button
                  onClick={() => handleDelete.mutate({ id: todo.id })}
                  className="bg-red-500 hover:bg-red-600 text-white px-3 py-1 rounded-md text-sm"
                >
                  Delete
                </button>
              </div>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

## 2. Using TanStack Query and tRPC Together: Server Components vs. Client Components

**Q:** How do I integrate TanStack Query and tRPC with Next.js Server Components? Do I need to make every component a "client component" (which feels counter-intuitive given Server Components' utility) and then use the `initialData` prop in `useQuery` to populate the initial data?

**A:** This is where the integration becomes really powerful, leveraging both Server Components and Client Components effectively. You absolutely **do not** need to make every component a client component.

The strategy depends on whether data needs to be _interactive/updateable_ on the client-side after the initial render.

### Scenario A: Data is Static or Only Needs Initial Render (No Client-Side Refetching/Updates)

**Use Pure Server Components:**
If you fetch data solely to render UI on the server, and that data doesn't need to change or be re-fetched on the client, you can use the **server-side tRPC caller** directly within Server Components. TanStack Query's client-side hooks aren't strictly necessary here, as its primary benefits (caching, re-fetching, mutations) are client-side.

```tsx
// src/app/server-only-page/page.tsx
import { api } from "~/trpc/server"; // Import the server-side tRPC caller (`createCaller`)

export default async function ServerOnlyPage() {
  const posts = await api.post.getAll.query(); // Fetch directly on the server via the server caller

  return (
    <div className="p-8 text-white">
      <h1 className="text-4xl font-bold mb-6">Posts (Server Rendered Only)</h1>
      {posts.length === 0 ? (
        <p>No posts yet.</p>
      ) : (
        <ul className="list-disc pl-5">
          {posts.map((post) => (
            <li key={post.id} className="mb-2">
              <h2 className="text-2xl font-semibold">{post.title}</h2>
              <p className="text-gray-300">{post.content}</p>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

In this case, `api` imported from `~/trpc/server` is the `createCaller` instance, specifically configured for server-side calls, bypassing client-side QueryClient setup.

### Scenario B: Data Needs to be Interactive, Refetchable, or Mutated on the Client

**Prefetch on Server, Hydrate on Client, Use `useQuery`:**
This is the most common and powerful pattern for truly interactive applications in the T3 Stack.

1.  **Fetch data on the Server Component:** Use the server-side tRPC caller to `prefetchQuery` into a `QueryClient` instance obtained via `api.createContext().queryClient`.
2.  **Dehydrate state:** Transform the `QueryClient`'s state into a serializable format using `dehydrate()`.
3.  **Pass dehydrated state:** Provide the `dehydratedState` to the client-side `<Hydrate>` component.
4.  **Use `useQuery` in a Client Component:** Within a client component, call `api.yourRouter.yourQuery.useQuery()`. TanStack Query will automatically pick up the hydrated data from the global `QueryClient` cache.

**Refer to the `src/app/page.tsx` and `src/app/_components/ClientTodoListComponent.tsx` examples in Question 1's answer for this pattern.**

### Regarding `initialData` Prop

While technically possible, the `dehydrate`/`Hydrate` pattern is generally preferred with the App Router and tRPC in the T3 Stack for managing cached data across components and routes.

**When `initialData` might be considered (less common in T3 App Router for primary data fetches):**

If you have a server component fetching data, and a _specific, isolated_ client component within it needs _only that data_, and you don't necessarily want that data to be managed by the global TanStack Query cache for potential re-use or invalidation by other parts of the app, you could pass it directly:

```tsx
// src/app/some-page.tsx (Server Component)
import IsolatedClientDisplay from "./_components/IsolatedClientDisplay";
import { api } from "~/trpc/server";

export default async function SomePage() {
  const serverFetchedIsolatedData = await api.isolatedRouter.getData.query();

  return (
    <div className="text-white p-8">
      <h1 className="text-3xl font-bold mb-4">Server Rendered Content</h1>
      <IsolatedClientDisplay initialData={serverFetchedIsolatedData} />
    </div>
  );
}
```

```tsx
// src/app/_components/IsolatedClientDisplay.tsx (Client Component)
"use client";
import { api } from "~/trpc/react"; // Client-side tRPC hooks

interface Props {
  initialData: { id: string; value: string }[];
}

export default function IsolatedClientDisplay({ initialData }: Props) {
  // Use initialData. TanStack Query will manage subsequent fetches, but
  // this data is isolated from global cache population by Hydrate.
  const { data, isLoading } = api.isolatedRouter.getData.useQuery(undefined, {
    initialData,
    staleTime: 1000 * 60 * 5, // Data considered fresh for 5 minutes
    // You might also add select to transform data if needed
  });

  if (isLoading) return <p>Loading isolated data...</p>;

  return (
    <div className="bg-gray-700 p-4 rounded-md">
      <h2 className="text-xl font-semibold mb-2">Isolated Data</h2>
      <ul className="list-disc pl-5">
        {data?.map((item) => (
          <li key={item.id}>Value: {item.value}</li>
        ))}
      </ul>
    </div>
  );
}
```

**Caveats with `initialData`:**

- **`staleTime` is Crucial:** If you provide `initialData` without a sufficient `staleTime`, TanStack Query will immediately mark the data as stale and potentially re-fetch it on mount, negating the benefit of server rendering. Set `staleTime` to a value that makes sense for how fresh the data needs to be.
- **Limited Cache Sharing:** Data provided via `initialData` to one `useQuery` hook isn't automatically added to the global `QueryClient` cache in a way that other `useQuery` calls (or `api.useContext().invalidate()`) easily benefit from. If you have multiple components fetching the same data, or if you need to invalidate it later from a different part of the app, the `dehydrate`/`Hydrate` approach is generally more robust for comprehensive cache management.

## 3. `initialData` for SSR and SEO Benefit

**Q:** If I use the `initialData` prop (or the `dehydrate`/`Hydrate` pattern), will it provide the "holy grail of SSR" or will I be stuck with "hellish CSR"? Can spiders/web crawlers read and understand it for SEO benefits?

**A:** Yes, both the `dehydrate`/`Hydrate` pattern and effectively used `initialData` will provide the benefits of **Server-Side Rendering (SSR)**.

### How it Works (SSR "Holy Grail"):

1.  **Server Request:** A user (or web crawler) makes a request to your Next.js application.
2.  **Server Component Execution:** Next.js executes the relevant Server Component (`page.tsx`).
3.  **tRPC on Server:**
    - Using the **server-side tRPC caller** (`api.createContext().queryClient.prefetchQuery()` or `api.router.query()`), your application fetches the necessary data from your backend tRPC API.
    - Crucially, this fetch happens _before_ the HTML is generated.
4.  **HTML Generation:** Next.js then renders the React tree into a complete HTML string. This HTML includes all the data retrieved in the previous step, statically embedded within the markup.
5.  **Dehydration/Hydration (for Interactive Data):**
    - If using the `prefetchQuery` and `dehydrate` pattern: The state of the `QueryClient` (which now holds the server-fetched data) is serialized (using `superjson` thanks to your `trpc.ts` configuration) into a JSON object. This JSON is then embedded within a `<script>` tag in the generated HTML.
    - This complete HTML response (containing your content and the dehydrated state) is sent to the client's browser.
6.  **Client Hydration:**
    - The browser receives the full HTML and can render it immediately. The user sees content on the screen without waiting for JavaScript to load or execute. **This is the core of SSR.**
    - Once the JavaScript bundles load, React "hydrates" the application: it takes over the server-rendered HTML, attaches event listeners, and makes the application interactive.
    - TanStack Query's `QueryClient` on the client-side detects the `dehydratedState` (if provided by `<Hydrate>`), loads this data into its cache, and then your `useQuery` hooks find the data already present. This means your components don't show a loading state for initial data, resulting in a smooth user experience.

### SEO Benefits:

**Absolutely, yes!** Spiders and web crawlers will read and understand the content.

- **Page Source Validation:** When you view the "Page Source" (or "View Source" in your browser) of an SSR'd page, you will see the full, rendered HTML **including all the data fetched from tRPC on the server**. This is exactly what search engine crawlers see.
- **Crawlability:** Since the content is present directly in the initial HTML response, crawlers don't need to execute JavaScript to discover your content. While modern crawlers like Googlebot are capable of rendering JavaScript, providing fully pre-rendered HTML from the start is always the most reliable and efficient method for good SEO, as it ensures all content is immediately discoverable and indexed.

### Beware of Time-Dependent Data / Hydration Errors:

As you correctly noted, this is a common pitfall.
"Though beware of time-dependent data (if you have things such as x seconds ago, current time, etc.) as that will cause hydration errors."

**Why this happens:**

- **Server Render Time:** When the HTML is generated on the server, any time-sensitive data (e.g., `new Date().toLocaleString()`, "5 minutes ago" calculations) will reflect the server's time at the moment of rendering.
- **Client Hydration Time:** When the client's browser hydrates the page, it re-renders the React components. If these components calculate time-sensitive data based on the _client's_ current time, and this time differs from the server's render time (even by milliseconds), React will detect a mismatch between the server-generated HTML and what it's trying to render on the client. This results in a "hydration error" (or warning in development), which can lead to unexpected UI behavior or even breaking parts of your application.

**Solutions for Time-Dependent Data:**

- **Render Only on Client:** If the data is purely for client-side display and a slight delay until hydration is acceptable, render it only on the client within a `useEffect` hook.

  ```tsx
  // Client Component
  "use client";
  import { useEffect, useState } from "react";

  export default function TimeDisplay() {
    const [currentTime, setCurrentTime] = useState("");

    useEffect(() => {
      // This code only runs on the client after hydration
      setCurrentTime(new Date().toLocaleString());
      const interval = setInterval(() => {
        setCurrentTime(new Date().toLocaleString());
      }, 1000);
      return () => clearInterval(interval);
    }, []);

    return <p className="text-white">Current Client Time: {currentTime}</p>;
  }
  ```

- **Pass Server Timestamp to Client for Client-Side Formatting:** Render a static timestamp string from the server (e.g., ISO string), and then have a client component parse and format it dynamically.

  ```tsx
  // Server Component (e.g., within a PostCard component in page.tsx)
  function PostCard({ post }: { post: { title: string; createdAt: Date } }) {
    return (
      <div className="text-white">
        <h2>{post.title}</h2>
        {/* Pass the server-generated Date object as an ISO string */}
        <ClientTimeAgo timestamp={post.createdAt.toISOString()} />
      </div>
    );
  }

  // Client Component (createdAt prop is a string)
  ("use client");
  import { useState, useEffect } from "react";
  import dayjs from "dayjs";
  import relativeTime from "dayjs/plugin/relativeTime";
  dayjs.extend(relativeTime); // Extend Day.js with relativeTime plugin

  interface ClientTimeAgoProps {
    timestamp: string; // ISO string from server
  }

  export default function ClientTimeAgo({ timestamp }: ClientTimeAgoProps) {
    // Initialize with the immediate relative time, then update
    const [timeAgo, setTimeAgo] = useState(() => dayjs(timestamp).fromNow());

    useEffect(() => {
      // This effect runs only on the client
      const interval = setInterval(() => {
        setTimeAgo(dayjs(timestamp).fromNow());
      }, 60000); // Update every minute for relative time
      return () => clearInterval(interval);
    }, [timestamp]);

    return <p className="text-gray-400">Posted: {timeAgo}</p>;
  }
  ```

  The server provides a consistent, immutable timestamp. The client-side component then takes over for any dynamic, time-relative calculations, avoiding hydration mismatches.

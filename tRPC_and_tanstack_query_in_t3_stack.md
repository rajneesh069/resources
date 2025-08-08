# T3 Stack, tRPC, and TanStack Query: Deep Dive & Best Practices

This document provides a comprehensive overview of using tRPC and TanStack Query within a T3 Stack Next.js application, emphasizing the client-side and server-side configurations for optimal performance and developer experience.

## The Core tRPC Server Configuration (`src/server/api/trpc.ts` & `src/server/api/root.ts`)

Let's break down the essential components of the tRPC server configuration, which is foundational for understanding how queries and mutations are defined and consumed.

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

## The tRPC Client-Side Integration (`src/trpc/react.tsx` & `src/trpc/query-client.ts`)

These files define how your frontend application interacts with your tRPC API and manages client-side data caching with TanStack Query.

**`src/trpc/query-client.ts`**

```typescript
// src/trpc/query-client.ts
import {
  defaultShouldDehydrateQuery,
  QueryClient,
} from "@tanstack/react-query";
import SuperJSON from "superjson"; // Note: renamed from SuperJSON to SuperJSON for consistency

export const createQueryClient = () =>
  new QueryClient({
    defaultOptions: {
      queries: {
        // With SSR, we usually want to set some default staleTime
        // above 0 to avoid refetching immediately on the client
        staleTime: 30 * 1000, // Data is considered fresh for 30 seconds
      },
      dehydrate: {
        // Use SuperJSON for serialization during dehydration
        serializeData: SuperJSON.serialize,
        // Custom logic to decide which queries to dehydrate
        shouldDehydrateQuery: (query) =>
          defaultShouldDehydrateQuery(query) || // Dehydrate by default if no errors/observers
          query.state.status === "pending", // Also dehydrate pending queries (e.g., if still loading on server)
      },
      hydrate: {
        // Use SuperJSON for deserialization during hydration
        deserializeData: SuperJSON.deserialize,
      },
    },
  });
```

**Explanation of `src/trpc/query-client.ts`:**

- **`createQueryClient()`**: This function provides a standardized way to create a new `QueryClient` instance.
- **`defaultOptions.queries.staleTime`**: Setting a `staleTime` (e.g., `30 * 1000` ms) is crucial for SSR. It means that once the data is hydrated on the client, TanStack Query will consider it "fresh" for this duration. This prevents an immediate re-fetch on component mount, saving network requests and improving perceived performance.
- **`dehydrate.serializeData: SuperJSON.serialize`**: Configures the dehydration process to use `SuperJSON` for serializing data. This ensures that complex types like `Date` objects, `Maps`, or `Sets` are correctly transmitted from the server to the client.
- **`dehydrate.shouldDehydrateQuery`**: A powerful option to control which queries get included in the dehydrated state. `defaultShouldDehydrateQuery(query)` includes queries that are successfully fetched and not errored. The addition of `query.state.status === "pending"` means that if a query is still in a "loading" state when the server renders, its pending status will also be dehydrated, potentially allowing for a "streaming" or pre-loading effect on the client.
- **`hydrate.deserializeData: SuperJSON.deserialize`**: Configures the hydration process to use `SuperJSON` for deserializing the data received from the server, correctly reconstructing complex types.

---

**`src/trpc/react.tsx`**

```typescript
// src/trpc/react.tsx (Client-side tRPC and React Query integration)
"use client"; // Marks this file as a Client Component

import { QueryClientProvider, type QueryClient } from "@tanstack/react-query";
import { httpBatchStreamLink, loggerLink } from "@trpc/client"; // Note: using httpBatchStreamLink
import { createTRPCReact } from "@trpc/react-query";
import { type inferRouterInputs, type inferRouterOutputs } from "@trpc/server"; // For type inference
import { useState } from "react";
import SuperJSON from "superjson";

import { type AppRouter } from "@/server/api/root"; // Your main tRPC router type
import { createQueryClient } from "./query-client"; // Your shared query client creator

let clientQueryClientSingleton: QueryClient | undefined = undefined;
const getQueryClient = () => {
  if (typeof window === "undefined") {
    // Server: When this function is called during SSR, always create a new QueryClient.
    // This is vital to prevent state leakage between different requests on the server.
    return createQueryClient();
  }
  // Browser: use singleton pattern to keep the same query client
  // All client components share the same QueryClient instance, allowing for a unified cache.
  clientQueryClientSingleton ??= createQueryClient();

  return clientQueryClientSingleton;
};

export const api = createTRPCReact<AppRouter>(); // Creates client-side tRPC hooks

/**
 * Inference helper for inputs.
 * Enables type-safe inference of input types for your tRPC procedures.
 * @example type CreateTodoInput = RouterInputs['todo']['create']
 */
export type RouterInputs = inferRouterInputs<AppRouter>;

/**
 * Inference helper for outputs.
 * Enables type-safe inference of output types for your tRPC procedures.
 * @example type AllTodosOutput = RouterOutputs['todo']['getAll']
 */
export type RouterOutputs = inferRouterOutputs<AppRouter>;

export function TRPCReactProvider(props: { children: React.ReactNode }) {
  const queryClient = getQueryClient(); // Get the appropriate QueryClient (new for server, singleton for client)

  const [trpcClient] = useState(() =>
    api.createClient({
      links: [
        // Logger for development, useful for debugging network requests
        loggerLink({
          enabled: (op) =>
            process.env.NODE_ENV === "development" ||
            (op.direction === "down" && op.result instanceof Error),
        }),
        // Main tRPC link: batches requests and supports streaming
        httpBatchStreamLink({
          transformer: SuperJSON, // Ensure complex types are handled
          url: getBaseUrl() + "/api/trpc",
          headers: () => {
            const headers = new Headers();
            // Custom header to identify the request source (helpful for server-side logging/middleware)
            headers.set("x-trpc-source", "nextjs-react");
            return headers;
          },
        }),
      ],
    })
  );

  return (
    // QueryClientProvider makes the QueryClient available to all components below it
    <QueryClientProvider client={queryClient}>
      {/* api.Provider makes the tRPC client hooks available */}
      <api.Provider client={trpcClient} queryClient={queryClient}>
        {props.children}
      </api.Provider>
    </QueryClientProvider>
  );
}

function getBaseUrl() {
  if (typeof window !== "undefined") return window.location.origin; // Browser
  if (process.env.VERCEL_URL) return `https://${process.env.VERCEL_URL}`; // Vercel deployment
  return `http://localhost:${process.env.PORT ?? 3000}`; // Local development
}
```

**Explanation of `src/trpc/react.tsx`:**

- **`"use client"`**: This directive explicitly marks this file and its exports as Client Components, meaning they will be rendered on the client side.
- **`getQueryClient()`**: This function implements a singleton pattern for the `QueryClient` _in the browser_. On the server, it always returns a _new_ `QueryClient` to prevent state leakage between concurrent requests. On the client, it reuses the same `QueryClient` instance, maintaining a consistent cache throughout the user's session.
- **`export const api = createTRPCReact<AppRouter>()`**: This is the magic! It generates the actual React hooks (`useQuery`, `useMutation`, `useContext`) based on the type of your `AppRouter`. This provides end-to-end type safety from your backend API definition to your React components.
- **`RouterInputs` / `RouterOutputs`**: These types are generated using `inferRouterInputs` and `inferRouterOutputs`. They are incredibly useful for type-checking when creating forms, handling API responses, or mocking data, ensuring your frontend code aligns perfectly with your backend API.
- **`TRPCReactProvider`**: This component wraps your entire application (typically in `layout.tsx`). It sets up:
  - **`QueryClientProvider`**: Makes the `QueryClient` instance (from `getQueryClient()`) available to all child components, enabling `useQuery`, `useMutation`, etc.
  - **`api.Provider`**: Makes the tRPC client instance available to all child components, enabling `api.yourRouter.yourQuery.useQuery()` calls.
  - **`httpBatchStreamLink`**: This is an advanced `tRPC` link that not only **batches multiple requests** into a single HTTP call (improving network efficiency) but also supports **streaming** responses, which can further optimize data delivery.
- **`getBaseUrl()`**: A utility function to correctly determine the API endpoint URL for both development, Vercel deployments, and client-side access.

---

## The tRPC Server-Side Components Integration (`src/trpc/server.ts`)

This file is a critical piece for making server-side data fetching with tRPC and TanStack Query seamless in Next.js Server Components.

```typescript
// src/trpc/server.ts (Server-side tRPC helpers for React Server Components)
import "server-only"; // Ensures this file is only ever imported on the server

import { createHydrationHelpers } from "@trpc/react-query/rsc"; // Helper for RSC hydration
import { headers } from "next/headers"; // Next.js specific for getting request headers
import { cache } from "react"; // React's built-in cache for memoization

import { createCaller, type AppRouter } from "@/server/api/root"; // Your server-side tRPC caller and router type
import { createTRPCContext } from "@/server/api/trpc"; // Your tRPC context creator
import { createQueryClient } from "./query-client"; // Your shared query client creator

/**
 * This wraps the `createTRPCContext` helper and provides the required context for the tRPC API when
 * handling a tRPC call from a React Server Component.
 */
const createContext = cache(async () => {
  const heads = new Headers(await headers());
  heads.set("x-trpc-source", "rsc"); // Set custom header to identify RSC source

  // Create and return the tRPC context for this specific server request
  return createTRPCContext({
    headers: heads,
  });
});

const getQueryClient = cache(createQueryClient); // Memoize the query client creation for RSC

const caller = createCaller(createContext); // Create a tRPC caller that uses the RSC-specific context

// This is the exported object that provides the server-side `api` for prefetching
// and the `HydrateClient` component for dehydrating state.
export const { trpc: api, HydrateClient } = createHydrationHelpers<AppRouter>(
  caller,
  getQueryClient
);
```

**Explanation of `src/trpc/server.ts`:**

- **`"server-only"`**: This special directive is a build-time check provided by Next.js. It ensures that this file (and any code imported from it) **can only ever be used on the server**. If you accidentally try to import it into a client component, the build will fail, preventing server-side code from leaking to the browser.
- **`createContext = cache(async () => { ... })`**:
  - This function creates the tRPC context for **React Server Components (RSC)**.
  - It uses `next/headers` to get request headers, setting an `x-trpc-source: "rsc"` header for debugging or server-side logging.
  - The `cache` utility from React memoizes this function's result for a given request. This means `createContext` will only be executed once per server request, even if called multiple times within different Server Components during the same render pass. This is vital for performance and consistency.
- **`getQueryClient = cache(createQueryClient)`**: Memoizes the `createQueryClient` function call specifically for Server Components. Each server request will get its own _new_ `QueryClient` instance, ensuring no state is shared between requests.
- **`caller = createCaller(createContext)`**: This creates the **server-side tRPC API caller**. This is the `api` object you'll use in your Server Components (e.g., `await api.todo.getAll.query()`) to fetch data directly from your tRPC backend. It's configured to use the `createContext` logic defined specifically for RSC.
- **`export const { trpc: api, HydrateClient } = createHydrationHelpers<AppRouter>(caller, getQueryClient)`**:
  - This is the powerful `tRPC` utility from `@trpc/react-query/rsc` that bundles everything together for Server Components.
  - It exports `trpc` (aliased as `api`) which is your server-side tRPC caller (the one capable of `prefetchQuery` into a `QueryClient`).
  - It also exports `HydrateClient`, which is a specialized React component. You use `HydrateClient` in your Server Components to wrap Client Components, passing the `dehydratedState` (implicitly managed by `createHydrationHelpers` when you call `api.someRouter.someQuery.prefetch()`). This component handles the actual injection of the server-fetched data into the client's `QueryClient` cache.

---

## 1. Why `HydrateClient` in `page.tsx` and Not `layout.tsx`? Should it be in every `page.tsx`?

**Q:** Given its purpose to hydrate client components with server-fetched data, why isn't the `QueryClientProvider` and associated hydration logic placed in the root `layout.tsx`? Should I include it in every `page.tsx` if I want to hydrate client components without passing data as props?

**A:** This is a crucial distinction related to Next.js App Router's architecture and TanStack Query's caching mechanisms, now fully clarified by the provided `TRPCReactProvider` and `createHydrationHelpers` setup.

**Understanding the T3 Stack App Router Setup:**

- The **`QueryClientProvider`** (wrapped by `TRPCReactProvider` from `src/trpc/react.tsx`) is placed in your **root `layout.tsx`**. This establishes a _single, consistent `QueryClient` instance_ for your entire client-side application, allowing for a shared cache and global invalidation strategies. Since `layout.tsx` does not re-render on soft navigations, this global cache persists across routes.

- The **`HydrateClient` component** (exported from `src/trpc/server.ts` via `createHydrationHelpers`) is what then receives the specific server-dehydrated state **per-page or per-route segment**.

**Why this pattern?**

1.  **Global Cache Management (`QueryClientProvider` in `layout.tsx`):**

    - Having `TRPCReactProvider` in `layout.tsx` makes the client-side `QueryClient` available globally. This allows `api.useQuery`, `api.useMutation`, etc., to function across your application with a shared cache. This is ideal for efficiency.

2.  **Page-Specific Data Hydration (`HydrateClient` in `page.tsx`):**
    - **Purpose:** The `HydrateClient` component's role is to inject server-fetched data _into that global client-side `QueryClient` cache_ specifically for the current page load.
    - **Behavior:** When you fetch data in a Server Component (`page.tsx`) using `api.yourRouter.yourQuery.prefetch()` (from `src/trpc/server.ts`), the `createHydrationHelpers` behind the scenes collects this pre-fetched data. The `HydrateClient` component then takes this collected dehydrated state and passes it to the underlying `@tanstack/react-query`'s `Hydrate` component.
    - This ensures that:
      - The initial HTML includes the pre-fetched data.
      - When client-side JavaScript takes over, the global `QueryClient`'s cache is populated with _this specific page's data_ _before_ client components render, providing immediate content.
      - The `staleTime` (e.g., 30 seconds from `src/trpc/query-client.ts`) means the client-side `useQuery` calls won't immediately re-fetch the data, relying on the server-provided content until it becomes stale.

**Should you put `HydrateClient` in every `page.tsx`?**

- **Yes, for pages that perform server-side data fetching intended for client components:** If a `page.tsx` (Server Component) uses the **server-side `api` (`src/trpc/server.ts`) to `prefetchQuery` data**, and you want client components within that page to automatically pick up this pre-fetched data via their **client-side `api.useQuery()` calls (`src/trpc/react.tsx`)**, then you **must** wrap your client components with `HydrateClient`. This is how the server-fetched data gets transferred to the client's TanStack Query cache.
- **No, for pages that _only_ render server-side:** If a page is purely a Server Component and uses the server-side `api` to fetch data and then renders it _without_ passing it to any child Client Components that use `api.useQuery()`, then `HydrateClient` is not needed. The data is consumed entirely on the server.

**Code Example (Refined for Modern T3 Stack):**

```tsx
// src/app/layout.tsx (Root Layout - Global QueryClientProvider via TRPCReactProvider)
import "~/styles/globals.css";
import { GeistSans } from "geist/font/sans";
import { TRPCReactProvider } from "~/trpc/react"; // Client-side setup for QueryClientProvider

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
// src/app/page.tsx (Server Component - Prefetches and Uses HydrateClient)
import { api, HydrateClient } from "~/trpc/server"; // Server-side tRPC for RSC & HydrateClient
import ClientTodoListComponent from "./_components/ClientTodoListComponent"; // A client component

export default async function HomePage() {
  // Prefetch data on the server using the server-side tRPC `api` (from createHydrationHelpers)
  // This call automatically hydrates the QueryClient passed to createHydrationHelpers
  await api.todo.getAll.prefetch(); // `prefetch()` is a shorthand from createHydrationHelpers

  return (
    <main className="flex min-h-screen flex-col items-center justify-center bg-gradient-to-b from-[#2e026d] to-[#15162c] text-white">
      <h1 className="text-4xl font-extrabold tracking-tight text-white sm:text-[5rem] mb-8">
        T3 <span className="text-[hsl(280,100%,70%)]">Todos</span>
      </h1>
      {/* HydrateClient passes the server's dehydrated state to the client's QueryClient */}
      <HydrateClient>
        <ClientTodoListComponent />{" "}
        {/* Client component that uses `api.todo.getAll.useQuery()` */}
      </HydrateClient>
    </main>
  );
}
```

```tsx
// src/app/_components/ClientTodoListComponent.tsx (Client Component - Consumes Hydrated Data)
"use client";

import { useState } from "react";
import { api } from "~/trpc/react"; // Client-side tRPC hooks

export default function ClientTodoListComponent() {
  // `useQuery` automatically picks up pre-fetched data from the QueryClient's cache.
  // Due to staleTime in query-client.ts, it won't refetch immediately.
  const { data: todos, isLoading, isError, error } = api.todo.getAll.useQuery();

  const addTodoMutation = api.todo.create.useMutation({
    onSuccess: () => {
      api.todo.getAll.invalidate(); // Refetch the list to show new todo
      setNewTodoText("");
    },
    onError: (err) => {
      alert(`Error creating todo: ${err.message}`);
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
    <div className="bg-gray-800 p-6 rounded-lg shadow-md w-full max-w-lg">
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
          className="bg-blue-500 hover:bg-blue-600 text-white p-2 rounded-md disabled:opacity-50"
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
If you fetch data solely to render UI on the server, and that data doesn't need to change or be re-fetched on the client, you can use the **server-side tRPC caller** (`api` from `src/trpc/server.ts`) directly within Server Components. TanStack Query's client-side hooks (`useQuery`, etc.) aren't strictly necessary here, as their primary benefits (caching, re-fetching, mutations) are client-side concerns.

```tsx
// src/app/server-only-page/page.tsx
import { api } from "~/trpc/server"; // Import the server-side tRPC caller (from createHydrationHelpers)

export default async function ServerOnlyPage() {
  // Fetch directly on the server via the server `api`
  const posts = await api.post.getAll.query();

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

In this scenario, `api` imported from `~/trpc/server` is the special server-side instance that directly calls your tRPC procedures without involving the client-side TanStack Query cache.

### Scenario B: Data Needs to be Interactive, Refetchable, or Mutated on the Client

**Prefetch on Server, Hydrate on Client, Use `useQuery`:**
This is the most common and powerful pattern for truly interactive applications in the T3 Stack, leveraging `createHydrationHelpers`.

1.  **Fetch data on the Server Component:** Use the server-side `api.yourRouter.yourQuery.prefetch()` from `~/trpc/server.ts`. This call automatically records the data to be dehydrated.
2.  **Render `HydrateClient`:** Place `HydrateClient` from `~/trpc/server.ts` to wrap your Client Components.
3.  **Use `useQuery` in a Client Component:** Within a client component, call `api.yourRouter.yourQuery.useQuery()` (from `~/trpc/react.ts`). TanStack Query will automatically pick up the hydrated data from the global `QueryClient` cache.

**Refer to the `src/app/page.tsx` and `src/app/_components/ClientTodoListComponent.tsx` examples in Question 1's answer for this exact pattern.**

### Regarding `initialData` Prop

While technically possible, the `dehydrate`/`HydrateClient` pattern from `createHydrationHelpers` is the standard and preferred approach in the modern T3 Stack with Next.js App Router for managing cached data across components and routes. It offers a more holistic and integrated caching solution.

Using `initialData` directly is generally reserved for more isolated data passing where you explicitly don't want the data to enter the shared TanStack Query cache, or when not using the `createHydrationHelpers` setup for simpler cases. The main drawbacks (manual `staleTime`, lack of shared cache) still apply.

## 3. `initialData` for SSR and SEO Benefit

**Q:** If I use the `initialData` prop (or the `dehydrate`/`HydrateClient` pattern), will it provide the "holy grail of SSR" or will I be stuck with "hellish CSR"? Can spiders/web crawlers read and understand it for SEO benefits?

**A:** Yes, both the `dehydrate`/`HydrateClient` pattern and effectively used `initialData` will provide the benefits of **Server-Side Rendering (SSR)**.

### How it Works (SSR "Holy Grail"):

1.  **Server Request:** A user (or web crawler) makes a request to your Next.js application.
2.  **Server Component Execution:** Next.js executes the relevant Server Component (`page.tsx`).
3.  **tRPC on Server (`api.prefetch()`):**
    - Using the **server-side `api.yourRouter.yourQuery.prefetch()`** (from `~/trpc/server.ts`), your application fetches the necessary data from your backend tRPC API.
    - Crucially, this fetch happens _before_ the HTML is generated.
4.  **HTML Generation:** Next.js then renders the React tree into a complete HTML string. This HTML includes all the data retrieved in the previous step, statically embedded within the markup.
5.  **Dehydration/Hydration (`HydrateClient`):**
    - The `createHydrationHelpers` (from `~/trpc/server.ts`) behind the `api.prefetch()` call automatically collects the state of the `QueryClient` after the prefetch.
    - The `HydrateClient` component then takes this collected dehydrated state, serializes it (using `SuperJSON` thanks to `query-client.ts`), and embeds it within a `<script>` tag in the generated HTML.
    - This complete HTML response (containing your content and the dehydrated state) is sent to the client's browser.
6.  **Client Hydration:**
    - The browser receives the full HTML and can render it immediately. The user sees content on the screen without waiting for JavaScript to load or execute. **This is the core of SSR.**
    - Once the JavaScript bundles load, React "hydrates" the application: it takes over the server-rendered HTML, attaches event listeners, and makes the application interactive.
    - TanStack Query's `QueryClient` on the client-side picks up the `dehydratedState` from the script tag, loads this data into its cache, and then your `useQuery` hooks find the data already present. Due to the `staleTime` (e.g., 30s) configured in `query-client.ts`, these initial queries won't immediately re-fetch the data, leading to a smooth user experience without loading flickers.

### SEO Benefits:

**Absolutely, yes!** Spiders and web crawlers will read and understand the content.

- **Page Source Validation:** When you view the "Page Source" (or "View Source" in your browser) of an SSR'd page, you will see the full, rendered HTML **including all the data fetched from tRPC on the server**. This is exactly what search engine crawlers see.
- **Crawlability:** Since the content is present directly in the initial HTML response, crawlers don't need to execute JavaScript to discover your content. While modern crawlers like Googlebot are capable of rendering JavaScript, providing fully pre-rendered HTML from the start is always the most reliable and efficient method for good SEO, as it ensures all content is immediately discoverable and indexed.

### Beware of Time-Dependent Data / Hydration Errors:

As you correctly noted, this is a common pitfall.
"Though beware of time-dependent data (if you have things such as x seconds ago, current time, etc.) as that will cause hydration errors."

Why this happens:

- Server Render Time: When the HTML is generated on the server, any time-sensitive data (e.g., new Date().toLocaleString(), "5 minutes ago" calculations) will reflect the server's time at the moment of rendering.
- Client Hydration Time: When the client's browser hydrates the page, it re-renders the React components. If these components calculate time-sensitive data based on the client's current time, and this time differs from the server's render time (even by milliseconds), React will detect a mismatch between the server-generated HTML and what it's trying to render on the client. This results in a "hydration error" (or warning in development), which can lead to unexpected UI behavior or even breaking parts of your application.
  Solutions for Time-Dependent Data:

- Render Only on Client: If the data is purely for client-side display and a slight delay until hydration is acceptable, render it only on the client within a useEffect hook.

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

- Pass Server Timestamp to Client for Client-Side Formatting: Render a static timestamp string from the server (e.g., ISO string), and then have a client component parse and format it dynamically.

```tsx
// Server Component (e.g., within a PostCard component in page.tsx)
function PostCard({ post }: { post: { title: string; createdAt: Date } }) {
  return (
    <div className="text-white">
      <h2>{post.title}</h2>
      {/_ Pass the server-generated Date object as an ISO string _/}
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

You've hit on some of the most critical patterns for data fetching in Next.js with the App Router, tRPC, and TanStack Query. Let's break down `initialData` vs. `HydrateClient` and then deep dive into the server/client component fetching strategies, providing clear examples and explanations.

---

# T3 Stack: Data Fetching Patterns in Next.js App Router

This section clarifies the different approaches to data fetching in the T3 Stack, focusing on the interplay between Server Components, Client Components, tRPC, and TanStack Query. We'll specifically compare `initialData` with `HydrateClient` and explore various data flow patterns.

## Demystifying `initialData` vs. `HydrateClient` (The Recommended Pattern)

Both `initialData` and `HydrateClient` achieve Server-Side Rendering (SSR) by providing initial data to a `useQuery` hook on the client. However, they differ fundamentally in how they manage that data within TanStack Query's cache.

### The `initialData` Pattern

**How it works:**

1.  A Server Component fetches data using the server-side tRPC caller (e.g., `api.someRoute.someQuery.query()`).
2.  This fetched data is then passed directly as a prop to a Client Component.
3.  Inside the Client Component, the `useQuery` hook is called, and the received prop is used as its `initialData`.

**Code Demo (`initialData`):**

```typescript
// src/server/api/routers/product.ts (New server-side tRPC router)
import { z } from "zod";
import { createTRPCRouter, publicProcedure } from "../trpc";

// Mock Product data
const MOCK_PRODUCTS = [
  {
    id: "p1",
    name: "Laptop",
    price: 1200.0,
    description: "Powerful portable computer.",
  },
  {
    id: "p2",
    name: "Mouse",
    price: 25.0,
    description: "Ergonomic wireless mouse.",
  },
  {
    id: "p3",
    name: "Keyboard",
    price: 75.0,
    description: "Mechanical RGB keyboard.",
  },
];

export const productRouter = createTRPCRouter({
  getProducts: publicProcedure.query(() => {
    console.log("Server fetching products for initialData demo...");
    return MOCK_PRODUCTS;
  }),
  getProductById: publicProcedure
    .input(z.object({ id: z.string() }))
    .query(({ input }) => {
      console.log(
        `Server fetching product by ID: ${input.id} for initialData demo...`
      );
      return MOCK_PRODUCTS.find((p) => p.id === input.id) || null;
    }),
});
```

```typescript
// src/server/api/root.ts (Ensure productRouter is added)
import { createCallerFactory, createTRPCRouter } from "@/server/api/trpc";
import { todoRouter } from "./routers/todo";
import { productRouter } from "./routers/product"; // <-- Add this

export const appRouter = createTRPCRouter({
  todo: todoRouter,
  product: productRouter, // <-- Add this
});

export type AppRouter = typeof appRouter;
export const createCaller = createCallerFactory(appRouter);
```

```tsx
// src/app/initial-data-demo/page.tsx (Server Component)
import { api } from "~/trpc/server"; // Server-side tRPC caller
import ProductListInitialData from "./_components/ProductListInitialData"; // Client Component

export default async function InitialDataDemoPage() {
  // 1. Fetch data on the server
  const products = await api.product.getProducts.query();

  return (
    <main className="flex min-h-screen flex-col items-center justify-center p-8 bg-gray-900 text-white">
      <h1 className="text-4xl font-bold mb-8">
        Initial Data Demo (Manual Prop Passing)
      </h1>
      {/* 2. Pass data as a prop to a Client Component */}
      <ProductListInitialData initialProducts={products} />
    </main>
  );
}
```

```tsx
// src/app/initial-data-demo/_components/ProductListInitialData.tsx (Client Component)
"use client";

import { api } from "~/trpc/react"; // Client-side tRPC hooks
import { type RouterOutputs } from "~/trpc/react"; // For type inference

// Infer the type of products from your tRPC router output
type Product = RouterOutputs["product"]["getProducts"][number];

interface ProductListInitialDataProps {
  initialProducts: Product[];
}

export default function ProductListInitialData({
  initialProducts,
}: ProductListInitialDataProps) {
  // 3. Use the initialData prop in useQuery
  const {
    data: products,
    isLoading,
    isError,
    error,
  } = api.product.getProducts.useQuery(undefined, {
    initialData: initialProducts, // Data from server-side prop
    staleTime: 5 * 60 * 1000, // Data considered fresh for 5 minutes
    // You MUST set a staleTime > 0, otherwise it will refetch immediately on mount!
    // queryFn: () => api.product.getProducts.query() // Not strictly needed if `initialData` provides it and `staleTime` is set
  });

  if (isLoading) return <p>Loading products...</p>; // This should ideally not show if initialData is provided
  if (isError) return <p className="text-red-500">Error: {error.message}</p>;

  return (
    <div className="bg-gray-800 p-6 rounded-lg shadow-md w-full max-w-2xl">
      <h2 className="text-2xl font-semibold mb-4">
        Products (Client Interactive)
      </h2>
      <ul className="space-y-3">
        {products.map((product) => (
          <li
            key={product.id}
            className="bg-gray-700 p-4 rounded-md flex justify-between items-center"
          >
            <div>
              <p className="font-bold text-lg">{product.name}</p>
              <p className="text-sm text-gray-400">{product.description}</p>
            </div>
            <p className="text-xl font-semibold">${product.price.toFixed(2)}</p>
          </li>
        ))}
      </ul>
      <p className="mt-4 text-sm text-gray-400">
        Data provided via `initialData` prop. Will refetch after 5 minutes if
        component remains mounted.
      </p>
    </div>
  );
}
```

### The `HydrateClient` Pattern (Recommended by T3 Stack)

**How it works:**

1.  A Server Component fetches data using `api.someRoute.someQuery.prefetch()` from `~/trpc/server.ts`. This call internally tells TanStack Query's server-side QueryClient to fetch and store this data.
2.  The `HydrateClient` component (also from `~/trpc/server.ts`) is rendered, which implicitly takes the dehydrated state from the server-side QueryClient and embeds it in the HTML.
3.  On the client, the global `QueryClient` (managed by `TRPCReactProvider` in `layout.tsx`) automatically picks up this dehydrated state and populates its cache.
4.  When `useQuery` is called in a Client Component, it finds the data already in the cache.

**Code Demo (`HydrateClient`):**

```tsx
// src/app/hydrate-client-demo/page.tsx (Server Component)
import { api, HydrateClient } from "~/trpc/server"; // Server-side tRPC for RSC & HydrateClient
import ProductListHydrate from "./_components/ProductListHydrate"; // Client Component

export default async function HydrateClientDemoPage() {
  // 1. Prefetch data on the server using `api.prefetch()`
  await api.product.getProducts.prefetch();
  // You can prefetch multiple queries for the same page:
  // await api.product.getProductById.prefetch({ id: "p1" });

  console.log("Server prefetched products for HydrateClient demo...");

  return (
    <main className="flex min-h-screen flex-col items-center justify-center p-8 bg-gray-900 text-white">
      <h1 className="text-4xl font-bold mb-8">
        HydrateClient Demo (Recommended)
      </h1>
      {/* 2. HydrateClient component handles the data transfer */}
      <HydrateClient>
        <ProductListHydrate />{" "}
        {/* Client component that uses `api.product.getProducts.useQuery()` */}
      </HydrateClient>
    </main>
  );
}
```

```tsx
// src/app/hydrate-client-demo/_components/ProductListHydrate.tsx (Client Component)
"use client";

import { api } from "~/trpc/react"; // Client-side tRPC hooks

export default function ProductListHydrate() {
  // `useQuery` automatically picks up pre-fetched data from the global QueryClient's cache.
  // Due to staleTime in query-client.ts (default: 30s), it won't refetch immediately.
  const {
    data: products,
    isLoading,
    isError,
    error,
  } = api.product.getProducts.useQuery();

  if (isLoading) return <p>Loading products...</p>;
  if (isError) return <p className="text-red-500">Error: {error.message}</p>;

  return (
    <div className="bg-gray-800 p-6 rounded-lg shadow-md w-full max-w-2xl">
      <h2 className="text-2xl font-semibold mb-4">
        Products (Client Interactive)
      </h2>
      <ul className="space-y-3">
        {products.map((product) => (
          <li
            key={product.id}
            className="bg-gray-700 p-4 rounded-md flex justify-between items-center"
          >
            <div>
              <p className="font-bold text-lg">{product.name}</p>
              <p className="text-sm text-gray-400">{product.description}</p>
            </div>
            <p className="text-xl font-semibold">${product.price.toFixed(2)}</p>
          </li>
        ))}
      </ul>
      <p className="mt-4 text-sm text-gray-400">
        Data provided via `HydrateClient` and stored in global QueryClient
        cache. Will refetch after default `staleTime` (30 seconds in
        `query-client.ts`).
      </p>
    </div>
  );
}
```

### Why `HydrateClient` is Generally Better than `initialData`

1.  **Centralized Cache Management:**

    - **`HydrateClient`**: Dehydrates/hydrates data into the **global `QueryClient` cache**. This means if multiple `useQuery` calls across different components (or even different pages, if configured) need the same data, they can all benefit from the single cached entry. This promotes data consistency and reduces redundant network requests.
    - **`initialData`**: Provides data directly to a _specific_ `useQuery` instance. This data is not automatically added to the global cache unless the `useQuery` hook itself performs a background re-fetch after becoming stale. Other `useQuery` instances for the same data won't benefit from this `initialData` immediately.

2.  **Seamless Invalidation:**

    - **`HydrateClient`**: Since data is in the global cache, you can easily invalidate it from anywhere (e.g., after a mutation using `api.useContext().someRouter.someQuery.invalidate()`). This will trigger a re-fetch for _all_ components consuming that query.
    - **`initialData`**: Invalidation is less direct. You'd need to explicitly `invalidate` the query key, but the benefit of the `initialData` is that it's already there. If you invalidate, TanStack Query will simply treat the data as stale and potentially re-fetch based on its configuration. The main point is that `HydrateClient` ensures the _initial server-provided state_ is part of the regular cache flow, making invalidation more intuitive.

3.  **Less Prop Drilling:**

    - **`HydrateClient`**: You simply call `api.someRouter.someQuery.prefetch()` on the server and use `api.someRouter.someQuery.useQuery()` on the client. You don't need to pass the actual data down through component props.
    - **`initialData`**: Requires explicitly passing the fetched data down as a prop, which can lead to prop drilling if the Client Component is nested several layers deep.

4.  **Automatic `staleTime` and `gcTime` Management:**
    - **`HydrateClient`**: The `staleTime` and `gcTime` (garbage collection time) configured in `src/trpc/query-client.ts` apply automatically to the hydrated data, ensuring efficient cache behavior without manual per-hook configuration.
    - **`initialData`**: You _must_ manually configure a `staleTime` for each `useQuery` call using `initialData` to prevent immediate re-fetching.

**In summary, `HydrateClient` (powered by `createHydrationHelpers`) provides a more robust, integrated, and idiomatic way to manage server-prefetched data within TanStack Query's global caching system, simplifying data flow and enabling powerful features like automatic invalidation.**

---

## Deep Dive into Data Fetching Patterns (Server/Client Component Interaction)

Let's explore the three common patterns for fetching data in a Next.js App Router application with tRPC, detailing when and how to use each.

### 1. Pure Server Component: Fetch Normally with tRPC

**Concept:** This pattern is for data that is only needed for the initial server render and does not require any client-side interactivity, re-fetching, or mutations. The data is fetched and consumed entirely within a Server Component, and no JavaScript is shipped to the client for this specific data interaction.

**When to use:**

- Displaying static content (e.g., blog post content, product descriptions that don't change).
- Rendering lists that don't need client-side filtering/sorting (or if filtering/sorting is done via full page reloads/server actions).
- SEO-critical content where you want the fastest possible load time and no reliance on JavaScript.
- Data that is very slow to fetch and you want to ensure it blocks rendering until available (if suspense is not used).

**How it works:**

- Import `api` from `~/trpc/server`.
- Call `await api.yourRouter.yourQuery.query()` directly within your `async` Server Component.

**Code Snippet:**

```tsx
// src/app/pure-server-demo/page.tsx (Pure Server Component)
// This file runs only on the server, so 'use client' is absent.
import { api } from "~/trpc/server"; // Import the server-side tRPC caller

export default async function PureServerDemoPage() {
  console.log("Server rendering PureServerDemoPage...");
  // 1. Fetch data directly using the server-side tRPC caller
  const products = await api.product.getProducts.query(); // No TanStack Query client-side involvement here

  return (
    <main className="flex min-h-screen flex-col items-center justify-center p-8 bg-gray-900 text-white">
      <h1 className="text-4xl font-bold mb-8">Pure Server Component Demo</h1>
      <div className="bg-gray-800 p-6 rounded-lg shadow-md w-full max-w-2xl">
        <h2 className="text-2xl font-semibold mb-4">
          Products (Server-Only Rendered)
        </h2>
        {products.length === 0 ? (
          <p>No products available.</p>
        ) : (
          <ul className="space-y-3">
            {products.map((product) => (
              <li key={product.id} className="bg-gray-700 p-4 rounded-md">
                <p className="font-bold text-lg">{product.name}</p>
                <p className="text-sm text-gray-400">{product.description}</p>
                <p className="text-xl font-semibold">
                  ${product.price.toFixed(2)}
                </p>
              </li>
            ))}
          </ul>
        )}
        <p className="mt-4 text-sm text-gray-400">
          This data was fetched and rendered entirely on the server. No
          client-side JavaScript for this data.
        </p>
      </div>
    </main>
  );
}
```

**Deep Dive:**

- **No Client-Side Footprint:** The key advantage here is that no JavaScript related to this data fetching is sent to the client. This reduces bundle size and improves initial page load performance, especially on slower networks.
- **Blocking Render:** The `await` keyword means the component will wait for the data to be fetched before sending the HTML to the client. This can be good for SEO (full content immediately available) but can introduce a delay if the data fetch is slow.
- **Limited Interactivity:** This pattern is not suitable if you need to re-fetch data based on user input, perform mutations, or manage complex client-side caching.

### 2. Server + Client Interaction: Prefetch on Server, Hydrate Client

**Concept:** This is the most common and powerful pattern for building interactive applications with SSR. Data is initially fetched on the server, embedded in the HTML, and then seamlessly handed off to TanStack Query on the client for further management (caching, re-fetching, mutations).

**When to use:**

- Any data that needs to be displayed immediately on load but also requires client-side interactivity (e.g., live search, filtering, infinite scrolling, data that can be updated or deleted).
- Providing a fast initial load (SSR) while retaining the benefits of a robust client-side data layer.

**How it works:**

- **Server Component (`page.tsx`):**
  - Import `api` and `HydrateClient` from `~/trpc/server`.
  - Use `await api.yourRouter.yourQuery.prefetch()` to fetch data on the server and prepare it for hydration.
  - Render `HydrateClient` wrapping your Client Components.
- **Client Component:**
  - Import `api` from `~/trpc/react`.
  - Use `api.yourRouter.yourQuery.useQuery()` as usual. TanStack Query will find the pre-fetched data in its cache.

**Code Snippet:**

**(Refer to the `src/app/hydrate-client-demo/page.tsx` and `src/app/hydrate-client-demo/_components/ProductListHydrate.tsx` examples above. That's the perfect demonstration of this pattern.)**

**Deep Dive:**

- **Best of Both Worlds:** Provides immediate content for SEO and perceived performance (SSR) while enabling full client-side interactivity, data synchronization, and optimistic UI updates through TanStack Query.
- **Efficient Hydration:** The `staleTime` configured in `src/trpc/query-client.ts` prevents an immediate re-fetch on the client, ensuring the server-provided data is used first.
- **Global Cache:** Data is put into TanStack Query's global cache, making it available to other `useQuery` calls with the same key and allowing easy invalidation.
- **No Prop Drilling (for primary data):** The data doesn't need to be passed down as props to the Client Component; `HydrateClient` handles the transfer via the QueryClient.

### 3. Server Component Deep Nesting: Passing Data via Props (Limited Depth)

**Concept:** This pattern involves a Server Component fetching data and passing it down as props to a Client Component, which might then pass it further down to another Client Component. This is _not_ for initial data hydration into TanStack Query's cache but rather for direct consumption of fetched data by nested Client Components that might not need the full power of `useQuery` for that specific data.

**When to use:**

- When a Server Component fetches data that a deeply nested Client Component needs, but that Client Component doesn't require `useQuery`'s caching, re-fetching, or mutation capabilities for _that specific data_.
- For configuration data, display-only data that is static, or props that are only used to trigger rendering logic (e.g., `productId` to fetch details in a sub-component).
- When the prop passing chain is limited to 1-2 levels deep to avoid excessive prop drilling.

**How it works:**

- **Server Component:** Fetches data using the server-side tRPC caller and passes it as a prop.
- **Client Component (Parent):** Receives the prop and simply passes it down to its child Client Component.
- **Client Component (Child):** Receives the prop and renders the data.

**Code Snippet:**

```tsx
// src/app/prop-drilling-demo/page.tsx (Server Component)
import { api } from "~/trpc/server";
import TopLevelClientComponent from "./_components/TopLevelClientComponent";

export default async function PropDrillingDemoPage() {
  console.log("Server rendering PropDrillingDemoPage...");
  // Fetch some product data on the server
  const product = await api.product.getProductById.query({ id: "p1" });

  return (
    <main className="flex min-h-screen flex-col items-center justify-center p-8 bg-gray-900 text-white">
      <h1 className="text-4xl font-bold mb-8">Prop Drilling Demo</h1>
      {product ? (
        // Pass the fetched product as a prop to a top-level Client Component
        <TopLevelClientComponent productData={product} />
      ) : (
        <p>Product not found.</p>
      )}
    </main>
  );
}
```

```tsx
// src/app/prop-drilling-demo/_components/TopLevelClientComponent.tsx (Client Component - Level 1)
"use client";
import { type RouterOutputs } from "~/trpc/react";
import NestedClientComponent from "./NestedClientComponent";

type Product = RouterOutputs["product"]["getProductById"];

interface TopLevelClientComponentProps {
  productData: Product; // Receives data as a prop
}

export default function TopLevelClientComponent({
  productData,
}: TopLevelClientComponentProps) {
  return (
    <div className="bg-gray-800 p-6 rounded-lg shadow-md w-full max-w-2xl mb-4">
      <h2 className="text-2xl font-semibold mb-4">
        Top Level Client Component
      </h2>
      <p className="mb-2">Product Name: {productData?.name}</p>
      {/* Passes the same prop down to a nested Client Component */}
      <NestedClientComponent product={productData} />
    </div>
  );
}
```

```tsx
// src/app/prop-drilling-demo/_components/NestedClientComponent.tsx (Client Component - Level 2)
"use client";
import { type RouterOutputs } from "~/trpc/react";

type Product = RouterOutputs["product"]["getProductById"];

interface NestedClientComponentProps {
  product: Product; // Receives data as a prop
}

export default function NestedClientComponent({
  product,
}: NestedClientComponentProps) {
  return (
    <div className="bg-gray-700 p-4 rounded-md">
      <h3 className="text-xl font-semibold mb-2">Nested Client Component</h3>
      <p>Product Description: {product?.description}</p>
      <p>Product Price: ${product?.price.toFixed(2)}</p>
      <p className="mt-4 text-sm text-gray-400">
        Data passed down via props. Not managed by TanStack Query.
      </p>
    </div>
  );
}
```

**Deep Dive:**

- **Simplicity for Non-Interactive Data:** For data that's merely for display and doesn't require complex caching or re-fetching logic on the client, this can be simpler than setting up `useQuery` hooks.
- **Prop Drilling Concerns:** If nesting becomes deep (3+ levels), this pattern leads to "prop drilling," making components less reusable and harder to maintain. In such cases, consider:
  - **Context API:** For truly global client-side state not managed by TanStack Query.
  - **Re-fetching in Child:** If the nested component _does_ need interactivity, it might be better to re-fetch the data in the nested Client Component using `api.useQuery()`, potentially with `initialData` or `HydrateClient` (if you can arrange the hydration boundary).
  - **Flattening Component Tree:** Refactor your component structure to reduce nesting.
- **No TanStack Query Benefits:** This data is not automatically cached by TanStack Query, nor can it be easily invalidated or re-fetched using TanStack Query's mechanisms. If the data needs to be dynamic on the client, this is not the ideal approach.

---

**Conclusion:**

The T3 Stack's integration of tRPC and TanStack Query with Next.js App Router provides flexible and powerful data fetching capabilities.

- Use **Pure Server Components** for static, display-only content that benefits from minimal client-side JavaScript.
- Embrace the **Server + Client with `HydrateClient` pattern** for virtually all interactive data, as it offers the best balance of SSR benefits and full client-side data management with TanStack Query. This is the recommended default.
- Reserve **prop passing (limited depth)** for simple, non-interactive data that doesn't need to live in the TanStack Query cache.
- Avoid `initialData` for `useQuery` when using `createHydrationHelpers` as `HydrateClient` provides a more robust and idiomatic solution for global cache management.

By strategically choosing the right pattern for each piece of data, you can build highly performant, scalable, and delightful applications with the T3 Stack.

# NextAuth Flow Deep Dive (with Database Strategy)

This document clarifies the exact flow of data in NextAuth.js when using the **database session strategy** with the Prisma Adapter, which is the default for the T3 Stack.

## The Core Concept: The Two Layers of Session Management

To understand the flow, you must think of NextAuth as having two layers on the server:

1.  **The Internal Token Layer (`jwt` callback):** Regardless of your strategy, NextAuth uses an internal, short-lived JWT on the server. This token acts as a "data packet" that carries information between the different stages of the authentication process. The `jwt` callback is your hook to add custom data (like a user's `role` or provider `accessToken`) into this packet.
2.  **The External Session Layer (`session` callback & strategy):** This layer determines how the session is stored and exposed to your app.
    - With the **`database` strategy**, NextAuth takes the internal data packet, uses it to create a session record in your database, and sends a simple, secure reference token to the client in a cookie.
    - The `session` callback then builds the final session object that your app (`auth()` and `useSession()`) will see.

**The key takeaway:** You use the `jwt` callback to _prepare the data efficiently_, and the `session` callback to _present that data_ to your application.

## The Recommended Config (`src/server/auth.ts`)

This configuration demonstrates the **best practice** for adding a custom `role` to your session for optimal performance.

```ts
import { PrismaAdapter } from "@auth/prisma-adapter";
import {
  type DefaultSession,
  type NextAuthConfig,
  type User as NextAuthUser,
} from "next-auth";
import GoogleProvider from "next-auth/providers/google";
import { db } from "@/server/db";
import GitHubProvider from "next-auth/providers/github";

// Assuming you have this in your prisma schema
type UserRole = "USER" | "ADMIN";

/**
 * Module augmentation for `next-auth` types. Allows us to add custom properties to the `session`
 * object and keep type safety.
 */
declare module "next-auth" {
  interface Session extends DefaultSession {
    user: {
      id: string;
      role: UserRole; // Add your custom property here
    } & DefaultSession["user"];
  }

  // We also need to add the role to the User model so the adapter can read it
  interface User extends NextAuthUser {
    role: UserRole;
  }
}

/**
 * Options for NextAuth.js used to configure adapters, providers, callbacks, etc.
 */
export const authConfig = {
  providers: [
    // ... your providers
  ],
  adapter: PrismaAdapter(db),
  session: {
    strategy: "database",
  },
  callbacks: {
    /**
     * The JWT callback is called first. It's the ideal place to perform actions
     * that should only happen once per sign-in, like adding the user's role to the token.
     * The data you add to the `token` here will be available in the `session` callback.
     */
    jwt: async ({ token, user }) => {
      // On initial sign-in, the `user` object is available
      if (user) {
        token.id = user.id;
        token.role = user.role; // The `user` object comes from the adapter, so it has the role
      }
      return token;
    },

    /**
     * The session callback is called after the JWT callback. It receives the token
     * from the `jwt` callback and uses it to build the final session object that is
     * returned to the client.
     */
    session: ({ session, token }) => {
      // `token` contains the data we added in the `jwt` callback
      if (session.user && token.id) {
        session.user.id = token.id as string;
        session.user.role = token.role as UserRole;
      }
      return session;
    },
  },
} satisfies NextAuthConfig;
```

My point was, you DO NOT NEED it, you can use it but can skip it in db strategy and directly make the session have all the properties a user model has - is it true? Also, what is the order of the callbacks when they run and which ones run per request, and which ones run only once and when? I need to add this info to the original doc you gave, lemme give it to you and can you address all my concerns in QnA form and add it to the following doc you gave?

# NextAuth Flow Deep Dive (with Database Strategy)

This document clarifies the exact flow of data in NextAuth.js when using the **database session strategy** with the Prisma Adapter, which is the default for the T3 Stack.

## The Core Concept: The Two Layers of Session Management

To understand the flow, you must think of NextAuth as having two layers on the server:

1.  **The Internal Token Layer (`jwt` callback):** Regardless of your strategy, NextAuth uses an internal, short-lived JWT on the server. This token acts as a "data packet" that carries information between the different stages of the authentication process. The `jwt` callback is your hook to add custom data (like a user's `role` or provider `accessToken`) into this packet.
2.  **The External Session Layer (`session` callback & strategy):** This layer determines how the session is stored and exposed to your app.
    - With the **`database` strategy**, NextAuth takes the internal data packet, uses it to create a session record in your database, and sends a simple, secure reference token to the client in a cookie.
    - The `session` callback then builds the final session object that your app (`auth()` and `useSession()`) will see.

**The key takeaway:** You use the `jwt` callback to _prepare the data_, and the `session` callback to _present that data_ to your application.

## The Corrected Config (`src/server/auth.ts`)

This configuration demonstrates how to correctly add a custom `role` to your session.

```ts
import { PrismaAdapter } from "@auth/prisma-adapter";
import {
  type DefaultSession,
  type NextAuthConfig,
  type User as NextAuthUser,
} from "next-auth";
import GoogleProvider from "next-auth/providers/google";
import { db } from "@/server/db";
import GitHubProvider from "next-auth/providers/github";

// Assuming you have this in your prisma schema
type UserRole = "USER" | "ADMIN";

/**
 * Module augmentation for `next-auth` types. Allows us to add custom properties to the `session`
 * object and keep type safety.
 */
declare module "next-auth" {
  interface Session extends DefaultSession {
    user: {
      id: string;
      role: UserRole; // Add your custom property here
    } & DefaultSession["user"];
  }

  // We also need to add the role to the User model
  interface User extends NextAuthUser {
    role: UserRole;
  }
}

/**
 * Options for NextAuth.js used to configure adapters, providers, callbacks, etc.
 */
export const authConfig = {
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    }),
    GitHubProvider({
      clientId: process.env.GITHUB_CLIENT_ID,
      clientSecret: process.env.GITHUB_CLIENT_SECRET,
    }),
  ],
  adapter: PrismaAdapter(db),
  session: {
    strategy: "database", // This is the default with an adapter
  },
  callbacks: {
    /**
     * The JWT callback is called first. It's the ideal place to perform actions
     * that should only happen once per sign-in, like fetching the user's role from the database.
     * The data you add to the `token` here will be available in the `session` callback.
     */
    jwt: async ({ token, user }) => {
      // On initial sign-in, the `user` object is available
      if (user) {
        token.id = user.id;
        token.role = user.role; // The `user` object comes from the adapter, so it has the role
      }
      return token;
    },

    /**
     * The session callback is called after the JWT callback. It receives the token
     * from the `jwt` callback and uses it to build the final session object that is
     * returned to the client.
     */
    session: ({ session, token }) => {
      // `token` contains the data we added in the `jwt` callback
      if (session.user && token.id) {
        session.user.id = token.id as string;
        session.user.role = token.role as UserRole;
      }
      return session;
    },
  },
} satisfies NextAuthConfig;
```

## The Authentication Flow (Database Strategy)

#### Step 1: OAuth & Adapter Interaction

The user is redirected to the OAuth provider (e.g., Google). After authenticating, they are sent back to your app. NextAuth's `PrismaAdapter` then does the following:

- Finds the user in your `User` table based on their email.
- If the user doesn't exist, it creates a new `User` record.
- It creates or updates an `Account` record, linking the user to the provider and storing tokens like `access_token` and `refresh_token` in the database.

#### Step 2: `signIn` Callback (The Gatekeeper)

- **When it runs:** After the adapter has done its work, but before the session is created.
- **Purpose:** To decide if a user is allowed to sign in. You can check for a verified email, a specific domain, or if their account is active. You generally **do not** modify session data here.
- **Returns:** `true` to allow sign-in, `false` or a redirect path (`"/banned"`) to deny it.

#### Step 3: `jwt` Callback (The Data Enricher)

- **When it runs:** After `signIn` is successful.
- **Purpose:** This is the most important callback for customizing session data. Its job is to enrich the **internal server-side token**.
- **On Initial Sign-In:** The `user` object (from your database, via the adapter) is available. You take its properties (`id`, `role`, etc.) and add them to the `token`. This is efficient because it happens only once.
- **For Provider Tokens:** This is also where you would add the provider's `access_token` from the `account` object to the `token` and implement the logic to refresh it if it expires.

#### Step 4: `session` Callback (The Data Exposer)

- **When it runs:** After the `jwt` callback, every time a session is accessed (`auth()`, `useSession()`).
- **Purpose:** To construct the final session object that will be sent to the client and exposed in your server code. It acts as a "filter," ensuring you don't accidentally leak sensitive data from the internal token to the frontend.
- **How it works:** It receives the enriched `token` from the `jwt` callback. You simply transfer the properties you need (`id`, `role`) from the `token` to the `session.user` object. **You should avoid making database calls here** for performance reasons; the data should already be prepared in the `token`.

### Summary of Data Flow

| Stage                | What Happens                                                        | Key Callback |
| :------------------- | :------------------------------------------------------------------ | :----------- |
| **Authentication**   | User logs in via provider; DB records are created/updated.          | -            |
| **Authorization**    | Decide if the user is allowed to proceed.                           | `signIn()`   |
| **Data Enrichment**  | Fetch custom data (`role`) ONCE and add it to an internal token.    | `jwt()`      |
| **Session Creation** | A `Session` record is created in the database.                      | -            |
| **Data Exposure**    | Shape the final session object for the app from the internal token. | `session()`  |

By following this model, you get a performant, secure, and scalable authentication system where custom data is fetched efficiently and is available with full type safety throughout your application.

## Callback Execution Order & Frequency

Understanding _when_ each callback runs is crucial for writing efficient code.

| Callback      | When It Runs                                                                                                                                                             | How Often                                            | Purpose                                                  |
| :------------ | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------- | :------------------------------------------------------- |
| **`signIn`**  | On sign-in attempt (before session creation).                                                                                                                            | **Once per sign-in.**                                | **Authorize:** Allow or deny the sign-in.                |
| **`jwt`**     | After a successful `signIn`. Also runs when a session is checked and `strategy` is "jwt". With `strategy: "database"`, it runs on sign-in to help create the DB session. | **Once per sign-in.**                                | **Enrich:** Add custom data to the internal token.       |
| **`session`** | When a session is accessed by the client or server (`useSession`, `auth()`).                                                                                             | **On every session check (per request/navigation).** | **Expose:** Build the final session object for your app. |

---

## Q&A: Clarifying Common Points of Confusion

### Q1: Can I add custom user fields to the session _without_ using the `jwt` callback in the database strategy?

**Yes, you technically can, but you shouldn't.**

It is possible to skip the `jwt` callback and put your logic directly in the `session` callback like this:

```ts
// TECHNICALLY POSSIBLE, BUT INEFFICIENT - NOT RECOMMENDED
callbacks: {
  session: async ({ session, user }) => {
    // The `user` object comes from the Prisma adapter and has your custom fields
    session.user.role = user.role;
    return session;
  },
}
```

**Why is this bad?**
The `session` callback runs on **every single request** that checks for a session. This implementation forces a **database query on every request** to get the `user.role`. This can severely slow down your application and overload your database.

The recommended approach (using the `jwt` callback) queries the database **only once** at sign-in and then re-uses that data from the in-memory token for all subsequent session checks, which is far more performant.

### Q2: So, if the `jwt` callback is optional, why does this document recommend it so strongly?

Because it represents the **performance and scalability best practice**.

- **Performance:** It prevents redundant database calls.
- **Scalability:** Your app will handle more traffic without slowing down.
- **Functionality:** It is the **only** place where you can implement logic to refresh expired OAuth provider tokens (like a Google `access_token`). You cannot do this in the `session` callback.

Think of it this way: skipping the `jwt` callback is an easy but inefficient shortcut. Using it is the professional, robust solution.

### Q3: What is the exact order of operations for a user signing in for the first time?

1.  **OAuth -> Adapter:** The user authenticates with Google. The Prisma Adapter creates the `User` and `Account` records in your database.
2.  **`signIn` callback runs:** Receives the newly created `user`. It returns `true` to allow the process to continue.
3.  **`jwt` callback runs:** Receives the same `user` object. It adds custom properties (`id`, `role`) to the `token`.
4.  **Database Session Creation:** NextAuth takes the user ID (`token.sub`), creates a new record in the `Session` table, and links it to the user.
5.  **Cookie is set:** A secure, http-only cookie containing the session token is sent to the user's browser. The sign-in is now complete.

### Q4: What happens on a subsequent request (e.g., navigating to another page)?

1.  **Cookie is sent:** The browser sends the session token cookie with the request.
2.  **Database Session Lookup:** NextAuth uses the token from the cookie to look up the session in your `Session` table.
3.  **`session` callback runs:** It receives a `session` object and a `user` object (fetched from the DB via the session link). If you were using the recommended `jwt` approach, it would also receive the `token` to build the final session object without another DB call.
4.  **Session Returned:** The final session object is returned to your `auth()` call or `useSession()` hook.

# Why use the `jwt` callback and NOT just the session callback?

### The Short Answer

When using the database strategy, the `user` object passed as a parameter to the `session` callback is the **result of a mandatory database query that NextAuth performs _before_ your callback is ever called.**

---

### The Journey of a Session Check: Where the DB Queries Happen

Imagine a user is logged in and navigates to their dashboard. Your `app/dashboard/page.tsx` calls `await auth()`. Here is the step-by-step journey that happens on the server, revealing the hidden database queries.

**Scenario: You are NOT using the `jwt` callback, only the `session` callback.**

1.  **Browser Sends Cookie:** The browser sends the request to your server, including the secure, http-only `next-auth.session-token` cookie. This cookie contains a simple, random string (e.g., `abc-123`).

2.  **NextAuth Receives Token:** NextAuth gets this token (`abc-123`). It knows nothing about the user yet, only this token.

3.  **DATABASE QUERY #1 (The Session Query):** NextAuth must now find out who this token belongs to. It performs a database query against your `Session` table.

    - **Conceptually:** `SELECT * FROM "Session" WHERE "sessionToken" = 'abc-123' LIMIT 1;`
    - **Result:** It gets back a session record containing the `userId` (e.g., `user-xyz`) and an `expires` date. It checks if the session is valid and not expired.

4.  **NextAuth Prepares for the `session` Callback:** The `session` callback's signature is `session({ session, user })`. NextAuth sees that it needs to provide a `user` object. It currently only has the `userId` (`user-xyz`).

5.  **DATABASE QUERY #2 (The User Query):** To fulfill the contract of its own callback, NextAuth **must** perform a second database query against your `User` table to get the full user object.

    - **Conceptually:** `SELECT * FROM "User" WHERE "id" = 'user-xyz' LIMIT 1;`
    - **Result:** It gets back the full user record: `{ id: 'user-xyz', name: '...', email: '...', role: 'ADMIN', ... }`.

6.  **Your `session` Callback is Finally Called:** Now that NextAuth has all the necessary data, it calls your function:
    ```ts
    // This function only runs AFTER two database queries have already succeeded.
    session: ({ session, user }) => {
      // The `user` object here is the result from Query #2.
      // Accessing `user.role` is now an instant, in-memory operation.
      session.user.role = user.role;
      return session;
    },
    ```

The slowness isn't in the line `session.user.role = user.role;`. The slowness is in the **two database round-trips that were required just to be able to call your function with the correct parameters.** This process happens on **every single request** that needs authentication.

### How the `jwt` Callback Prevents This

1.  **Sign-in (Once):**

    - User signs in.
    - The `jwt` callback runs **once**.
    - It queries the DB **once** to get the role.
    - It creates an internal token like `{ id: 'user-xyz', role: 'ADMIN' }`.
    - A session is created in the database.

2.  **Subsequent Request (e.g., navigating to dashboard):**
    - **Browser Sends Cookie:** Same as before.
    - **DATABASE QUERY #1 (The Session Query):** Same as before. NextAuth queries the `Session` table to validate the token and get the `userId`. This is unavoidable.
    - **NextAuth Prepares for Callbacks:** NextAuth sees it has the `userId`. It knows it needs to run the `jwt` and `session` callbacks.
    - **The `jwt` callback is called:** NextAuth re-builds the internal token. Crucially, it uses the `userId` from the session to populate the token. It does **not** need to re-query the `User` table for the role, because that logic is inside the `if (user)` block which only runs at sign-in.
    - **The `session` callback is called:**
      ```ts
      session: ({ session, token }) => {
        // The `token` is already populated with the role from sign-in.
        // This is an instant, in-memory operation.
        session.user.role = token.role;
        return session;
      };
      ```
    - In this flow, there is **NO second database query for the `User` object**.

### Summary Table: Database Hits Per Request

| Approach                     | DB Query #1 (`Session` table) | DB Query #2 (`User` table) | Total Queries per Request |
| :--------------------------- | :---------------------------- | :------------------------- | :------------------------ |
| **`session`-Only (Slow)**    | ✅ Yes                        | ✅ Yes                     | **2**                     |
| **`jwt` + `session` (Fast)** | ✅ Yes                        | ❌ **No**                  | **1**                     |

By using the `jwt` callback as a caching layer, you effectively **halve the number of database queries** on every authenticated request, which is a massive performance win.

# Why the second query though?

### The Decisive Factor: The `user` Parameter Itself

The second database query (`User` table) is triggered **specifically because the `user` object is a required parameter for the `session` callback in that configuration.**

#### The Causality Chain

When you define your `session` callback, you are telling NextAuth what data you expect to receive.

**Scenario 1: The Slow Path**

You write this code:

```ts
callbacks: {
  session: ({ session, user }) => { // You are REQUESTING the `user` object
    session.user.role = user.role;
    return session;
  },
}
```

1.  NextAuth prepares to call your function.
2.  It looks at the function's signature and sees that you have requested the `user` parameter.
3.  NextAuth says, "I have a contract to fulfill. I _must_ provide a complete `user` object from the database to this function."
4.  It has already done Query #1 (on the `Session` table) to get the `userId`.
5.  To fulfill the contract, it is now **forced** to perform Query #2 (on the `User` table) using that `userId`.
6.  Only after Query #2 is complete can it call your function with both the `session` and the fully populated `user` objects.

In this model, **the function signature _causes_ the database query.**

---

**Scenario 2: The Fast Path (The `jwt` solution)**

You write this code:

```ts
callbacks: {
  jwt: async ({ token, user }) => {
    // This runs ONCE at sign-in, caching the role in the token
    if (user) {
      token.role = user.role;
    }
    return token;
  },
  session: ({ session, token }) => { // You are NOT requesting the `user` object
    // You are requesting the `token` instead
    session.user.role = token.role;
    return session;
  },
}
```

1.  NextAuth prepares to call your `session` function.
2.  It looks at the function's signature and sees you have requested the `token` parameter, **not** the `user` parameter.
3.  NextAuth says, "I have a contract to fulfill. I must provide a `token` object."
4.  It has already done Query #1 (on the `Session` table) to get the `userId`.
5.  It uses the `userId` to reconstruct the internal `token` (which was originally created by the `jwt` callback at sign-in). This is a fast, in-memory operation.
6.  It has now fulfilled its contract. **It has no reason to perform Query #2 because you never asked for the `user` object.**
7.  It calls your function with the `session` and `token` objects.

In this model, your function signature **prevents** the second database query.

### "that's what is being saved by jwt, right?"

**Precisely.**

The `jwt` callback's primary job in the database strategy is to act as a **caching mechanism**. It runs once at sign-in to do the expensive work (like querying the `User` table for custom fields). It then "saves" or "caches" the result of that work into the internal token.

This allows you to design a `session` callback that relies on the cheap, in-memory `token` instead of the expensive, database-queried `user` object, thereby eliminating the extra database hit on every request.

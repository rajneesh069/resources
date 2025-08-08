# Next Auth Flow Deep Dive

- [Simplified Flow Diagram](./next_auth_flow_diagram.png)

## The config:

```ts
import { PrismaAdapter } from "@auth/prisma-adapter";
import { type DefaultSession, type NextAuthConfig } from "next-auth";
import GoogleProvider from "next-auth/providers/google";
import { db } from "@/server/db";
import GitHubProvider from "next-auth/providers/github";

/**
 * Module augmentation for `next-auth` types. Allows us to add custom properties to the `session`
 * object and keep type safety.
 *
 * @see https://next-auth.js.org/getting-started/typescript#module-augmentation
 */
declare module "next-auth" {
  interface Session extends DefaultSession {
    user: {
      id: string;
      // ...other properties
      // role: UserRole;
    } & DefaultSession["user"];
  }

  // interface User {
  //   // ...other properties
  //   // role: UserRole;
  // }
}

/**
 * Options for NextAuth.js used to configure adapters, providers, callbacks, etc.
 *
 * @see https://next-auth.js.org/configuration/options
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
  callbacks: {
    session: ({ session, user }) => {
      console.log("SESSION RUNNING\n\n\n");
      return {
        ...session,
        user: {
          ...session.user,
          id: user.id,
        },
      };
    },
  },
} satisfies NextAuthConfig;
```

Let's understand the exact order and working of the NextAuth.js callbacks, specifically focusing on how they handle `accessToken`, `refreshToken`, and the user ID from an OAuth provider (like Google or GitHub), in the context of your setup with a Prisma Adapter.

This is the most critical flow to understand for custom fields and token refreshing!

### Scenario: User logs in via OAuth Provider (e.g., Google) for the first time or subsequent times.

Here's the step-by-step process:

1.  **OAuth Provider Redirection & Callback:**

    - User clicks "Sign in with Google."
    - NextAuth.js redirects to Google.
    - User authenticates with Google and grants permissions.
    - Google redirects back to your NextAuth.js `/api/auth/callback/google` endpoint.
    - Google sends `code` (and potentially `state`) to NextAuth.js.

2.  **NextAuth.js Internal Processing (Authentication Phase):**

    - NextAuth.js exchanges the `code` for `accessToken`, `refreshToken`, `expires_at`, `id_token` from Google's token endpoint.
    - It also uses the `accessToken` to fetch the user's `profile` (e.g., `name`, `email`, `image`) from Google's user info endpoint.
    - **Adapter Interaction (PrismaAdapter):**
      - NextAuth.js uses the `profile` information to check if the user already exists in your `User` table (via `db.user.findUnique` typically).
      - If not, it creates a new `User` record in your database.
      - It then creates/updates an `Account` record in your database, storing the `userId` linked to this `User`, along with `provider`, `providerAccountId`, and crucially, the `accessToken`, `refreshToken`, `expires_at`, `id_token` received from Google.

3.  **`signIn` Callback (Execution Order: 1st User-Defined Callback):**

    - **Arguments:** You get `user` (the object from your `User` model in the database, with its `id`, `name`, `email`, `image`, and any other custom fields like `createdAt`, `updatedAt`, `role`), `account` (the full `Account` object from the database, containing all provider tokens), `profile` (raw data from provider), etc.
    - **Purpose:** This is your initial gatekeeper. You can inspect the `user` or `account` to decide if this sign-in attempt should be allowed (`return true`) or denied (`return false` or `return "/error-page"`).
    - **Provider Tokens:** You see `account.access_token`, `account.refresh_token` here, but you generally **don't manipulate them here**. This callback is for authorization (`allow/deny`), not data transformation for the session.

    ```typescript
    // Example:
    async signIn({ user, account, profile }) {
      // Check if user has a specific role or email domain
      if (user && account?.provider === "google") {
        // You can access account.access_token here, but it's not where you put it in session
        // console.log("Google access token during signIn:", account.access_token);
        // const dbUser = await db.user.findUnique({ where: { id: user.id }, select: { role: true }});
        // if (dbUser?.role === "BLOCKED") return false;
      }
      return true; // Proceed with sign-in
    },
    ```

4.  **`jwt` Callback (Execution Order: 2nd User-Defined Callback):**

    - **Arguments:**
      - `token`: The internal JWT. On the _very first call_ after sign-in, this is initially an empty/minimal JWT. On subsequent calls (session refresh), it contains the data from the previous `jwt` callback execution.
      - `user`: The `User` object from your database. (Available _only_ on initial sign-in).
      - `account`: The `Account` object from your database, containing `access_token`, `refresh_token`, etc. (Available _only_ on initial sign-in).
    - **Purpose:** This is the _most crucial_ callback for enriching the internal JWT that carries your session data.
    - **How it works with Provider Tokens and Custom Fields:**

      - **Initial Sign-in (`if (user && account)` block):**

        - This block executes only once when the user successfully signs in.
        - You take `user.id` and assign it to `token.id` (or `token.sub`).
        - You query your database (e.g., `db.user.findUnique`) to fetch any _custom fields_ like `createdAt`, `updatedAt`, `role` from your `User` model, and add them to `token` (e.g., `token.createdAt`, `token.updatedAt`, `token.role`).
        - You take the provider's `access_token`, `refresh_token`, `expires_at` from the `account` object and add them to `token` (e.g., `token.accessToken`, `token.refreshToken`, `token.accessTokenExpires`). This is vital for later refreshing.

      - **Subsequent Calls (Session Refreshes):**
        - `user` and `account` are _not_ available here.
        - The `token` object contains the data that was last returned by this `jwt` callback.
        - This is where you implement the `refreshAccessToken` logic:
          - Check `if (token.accessTokenExpires && Date.now() > token.accessTokenExpires)`.
          - If expired, use `token.refreshToken` to make an API call to Google to get a _new_ `accessToken` and `expires_in`.
          - Update `token.accessToken` and `token.accessTokenExpires` with the new values.
          - Return the updated `token`.
        - If the access token is still valid, simply `return token;`.

    ```typescript
    // Example:
    async jwt({ token, user, account }) {
      // 1. Initial Sign-in
      if (account && user) {
        token.id = user.id; // From DB user
        token.accessToken = account.access_token; // From DB account
        token.refreshToken = account.refresh_token; // From DB account
        token.accessTokenExpires = account.expires_at * 1000; // From DB account (to ms)

        // Fetch custom fields from DB
        const dbUser = await db.user.findUnique({
          where: { id: user.id },
          select: { createdAt: true, updatedAt: true, role: true },
        });
        if (dbUser) {
          token.createdAt = dbUser.createdAt.getTime(); // Store as timestamp
          token.updatedAt = dbUser.updatedAt.getTime();
          token.role = dbUser.role;
        }
      }
      // 2. Subsequent calls (token refresh)
      // Check if access token is expired, and if so, refresh it
      if (token.accessTokenExpires && Date.now() > token.accessTokenExpires) {
        // Assume refreshAccessToken is a function defined elsewhere
        // that takes token.refreshToken and returns a new token with updated access_token, expiry, etc.
        console.log("Refreshing expired token...");
        return refreshAccessToken(token);
      }
      // If token is not expired, just return it
      return token;
    },
    ```

5.  **`session` Callback (Execution Order: 3rd User-Defined Callback):**

    - **Arguments:**
      - `session`: The default session object, initially populated with `name`, `email`, `image` from the database `User` object (because you have an adapter).
      - `user`: The `User` object directly from your database (if using `database` strategy).
      - `token`: The **internal JWT returned from the `jwt` callback** (containing all the custom fields and provider tokens you added/refreshed).
    - **Purpose:** This is where you construct the final `session` object that will be exposed to your client-side (`useSession()`) and server-side (`getServerSession()`) code.
    - **How it works:** You take the custom data from the `token` (which you've already ensured is up-to-date and complete in the `jwt` callback) and transfer it to the `session.user` object.

    ```typescript
    // Example:
    async session({ session, user, token }) {
      if (session.user && token) {
        // Transfer 'id' from token (or user)
        session.user.id = user?.id || token.id; // Use user.id if available (direct from DB), else token.id

        // Transfer custom fields from token (converted back from timestamp if needed)
        if (token.createdAt) session.user.createdAt = new Date(token.createdAt);
        if (token.updatedAt) session.user.updatedAt = new Date(token.updatedAt);
        if (token.role) session.user.role = token.role;

        // If you need the actual provider access token on the client for API calls
        if (token.accessToken) session.user.accessToken = token.accessToken;
      }
      // This is the object available to useSession() and getServerSession()
      return session;
    },
    ```

6.  **Client-Side Session Availability:**
    - The `session` object returned by the `session` callback is now available to your frontend (via `useSession()` hook in `next-auth/react`) and server-side logic (via `auth()` or `getServerSession()` in API routes/`getServerSideProps`).

### Summary of Data Flow:

- **OAuth Provider:** Gives you `access_token`, `refreshToken`, `id_token`, `expires_at`, `profile`.
- **PrismaAdapter:** Stores these (along with `userId`) in your `Account` model in the database. Also creates/updates `User` records.
- **`signIn` Callback:** Sees the `user` (from DB) and `account` (from DB, including tokens). Decides allow/deny. No data modification for session here.
- **`jwt` Callback:**
  - **Initial:** Gets `user` (DB) and `account` (DB). Pulls `id`, custom user fields (from `user`), and provider tokens (from `account`). **Adds all of this to the `token` (internal JWT).**
  - **Subsequent:** Receives the `token` (from previous `jwt` run). **Refreshes provider tokens if expired and updates `token`**. Returns the `token`.
- **`session` Callback:** Receives the `token` (the rich, internal JWT) and `user` (from DB). **Constructs the final `session` object for the client from `token` (and `user`)**.

This detailed flow clarifies why the `jwt` callback is indispensable for custom fields and provider token management, even when your main session strategy is `database` with Prisma.

That's a very insightful question and points to a common point of confusion when first working with NextAuth.js, especially with database adapters!

The short answer is: **Even with the database strategy and PrismaAdapter, NextAuth.js still uses an internal JWT, and the `jwt` callback is your primary means of modifying that internal token before it's used to create the session.**

Let's break down why:

# The Two "Layers" of Session Management in NextAuth.js

NextAuth.js conceptually operates with two layers of session management:

1.  **The Internal JWT Layer:**

    - Regardless of your chosen `session.strategy` (`database` or `jwt`), NextAuth.js _always_ creates and uses an internal **JSON Web Token (JWT)** on the server.
    - This JWT is never directly exposed to the client when using the `database` strategy.
    - Its purpose is to **carry essential session data during the server-side authentication flow** from the point of authentication to the point where the session is written to the database (or a client-side cookie if using the `jwt` strategy).
    - The `jwt` callback directly interacts with and manipulates this internal JWT.

2.  **The External Session Layer (based on `session.strategy`):**
    - This layer determines how the _actual user session_ (what gets sent to the client via `useSession()` or `getServerSession()`) is managed.
    - **`session.strategy: "database"` (your current setup):**
      - After the `jwt` callback runs and the internal JWT is created/modified, NextAuth.js uses the data from this JWT to:
        - Create/update `Session` records in your database (via PrismaAdapter).
        - Set a simple `next-auth.session-token` cookie on the client. This cookie **does not contain the full session data**; it only contains a lookup ID that points to the session record in your database.
        - The `session` callback then pulls data from the database user and/or the internal JWT to build the final `session` object sent to the client.
    - **`session.strategy: "jwt"`:**
      - The internal JWT _itself_ becomes the client-side session cookie. No database `Session` records are created by NextAuth.js for session management.
      - The `session` callback then directly uses the internal JWT to build the `session` object for the client.

### Why You Need the `jwt` Callback with the Database Strategy:

1.  **Data Flow for Custom Fields:**

    - When a user logs in, NextAuth.js receives `user` and `account` objects from the provider.
    - The `jwt` callback is the **first place** where you can intercept this `user` and `account` data and `await` additional data from your database (like `createdAt`, `updatedAt`, `role`, or even a Google `accessToken`).
    - You then add this fetched data to the `token` (the internal JWT).
    - This enriched `token` is then passed to the `session` callback.

2.  **Populating `session.user.id`:**

    - Even though the database strategy handles `userId` in the `Session` model, the `DefaultSession` interface doesn't automatically populate `session.user.id`.
    - The `jwt` callback is the standard way to ensure `token.id = user.id` is set. Then, the `session` callback can pick `session.user.id = token.id` to ensure it's available for the client. This provides a consistent way to add `id` regardless of the session strategy.

3.  **Handling Provider Tokens (e.g., Google `refresh_token`):**

    - NextAuth.js stores provider-specific tokens (`access_token`, `refresh_token`, `expires_at`) in your `Account` model when using a database adapter.
    - However, NextAuth.js **does not automatically use the `refresh_token` to get new `access_token`s** when the original one expires.
    - The `jwt` callback is the _correct_ place to implement the logic for refreshing provider access tokens. You check if `token.accessTokenExpires` is in the past, use `token.refreshToken` to get a new `accessToken`, and then update `token.accessToken` and `token.accessTokenExpires`.
    - Without this `jwt` callback logic, your Google `access_token` would expire after an hour, and you wouldn't be able to make further API calls without the user re-authenticating.

4.  **Consistency and Reusability:**
    - By putting the data enrichment logic in the `jwt` callback, you centralize how your user data flows into the session.
    - If you ever decide to switch your `session.strategy` from `database` to `jwt`, your `jwt` callback logic for adding custom fields or handling token refreshes would remain largely the same, making your configuration more portable.

### Analogy:

Think of it like this:

- **`signIn` callback:** The bouncer at the club door. Decides who gets in at all.
- **`jwt` callback:** The "preparation room" behind the bouncer. This is where you get all the necessary personal belongings (user ID, roles, custom data, fresh provider tokens) ready and pack them into a special "internal passport" (the internal JWT).
- **`session` callback:** The "presenter" of the session. It takes the "internal passport" (the internal JWT from the `jwt` callback) and/or directly queries the database user record to construct the final "ID card" (the session object) that the client gets to hold.
- **`session.strategy: "database"`:** The "ID card" has a secret number on it that points to a locked safe (your database) containing all your full details. The client only sees the number, the server goes to the safe.
- **`session.strategy: "jwt"`:** The "ID card" _is_ the full "internal passport" itself, encrypted and given directly to the client.

Even if your "ID card" just points to a safe (database strategy), you still need that "preparation room" (`jwt` callback) to gather and prepare all the right details to put into the safe in the first place, or to get fresh supplies for your API calls.

Therefore, you absolutely need to implement the `jwt` callback to effectively manage custom fields and provider access token refreshing, even when using the database session strategy with PrismaAdapter.

# Anatomy of JWT

Let's dive into JWTs and the `sub` (subject) claim! This is fundamental to understanding how identities are managed in token-based authentication.

### What is a JWT (JSON Web Token)?

A JWT is a compact, URL-safe means of representing claims to be transferred between two parties. The claims in a JWT are encoded as a JSON object that is digitally signed.

A JWT typically consists of three parts, separated by dots (`.`):

1.  **Header:**

    - Contains the token type (JWT) and the signing algorithm used (e.g., HMAC SHA256 or RSA).
    - Example: `{"alg": "HS256", "typ": "JWT"}`

2.  **Payload (Claims):**

    - This is the core of the JWT, a JSON object that contains the "claims" (statements about an entity, typically the user, and additional data).
    - There are three types of claims:
      - **Registered Claims:** Pre-defined, non-mandatory but recommended claims (e.g., `iss`, `exp`, `sub`, `aud`).
      - **Public Claims:** Claims defined by users for public use, often registered in the IANA JWT Registry, or defined with collision-resistant names.
      - **Private Claims:** Custom claims created by you that are neither registered nor public. Used for sharing information between parties who agree on their meaning.
    - Example: `{"sub": "1234567890", "name": "John Doe", "admin": true}`

3.  **Signature:**
    - Created by taking the encoded header, the encoded payload, a secret (or a private key), and the algorithm specified in the header, and then signing them.
    - This signature is used to verify that the sender of the JWT is who it says it is and that the message hasn't been tampered with.

A full JWT looks like: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ`

### What is `sub` (Subject Claim)?

`sub` stands for **"subject"**. It is a **Registered Claim** in the JWT specification (RFC 7519).

- **Purpose:** The `sub` claim identifies the **principal** (the entity or user) that the JWT is about. In most authentication systems, this will be the unique identifier for the user.
- **Uniqueness:** It is typically a **unique, non-reassignable, and non-sensitive identifier** for the user. Examples include a database `user.id`, a unique username, or an email address (if it's guaranteed to be unique and immutable).
- **Non-sensitive:** While it identifies the user, it generally shouldn't contain sensitive PII (Personally Identifiable Information) directly, as the payload is only _encoded_, not _encrypted_ (unless a separate encryption step is applied to the entire JWT). The signature ensures integrity, not confidentiality.
- **Relationship to NextAuth.js:** In NextAuth.js, when a user successfully authenticates and you're using an adapter, the `sub` claim in the internal JWT is almost always set to the `id` of the user from your database (e.g., `user.id` from your Prisma `User` model).

### How `sub` Works in Practice (with NextAuth.js):

1.  **User Authentication:** A user signs in using Google.
2.  **User Record Created/Found:** NextAuth.js (via PrismaAdapter) either creates a new `User` record in your database or finds an existing one. Let's say this user has `id: "clx123abc"`.
3.  **Internal JWT Generation (Default `jwt` Callback):**
    - If you don't define your own `jwt` callback, NextAuth.js's default `jwt` callback creates an internal JWT.
    - It takes the `user.id` (`"clx123abc"`) and sets it as the `sub` claim in the JWT's payload: `{"sub": "clx123abc"}`.
    - This JWT is then signed.
4.  **Session Creation (Database Strategy):**
    - NextAuth.js then uses this internal JWT (primarily its `sub` claim) to look up or create a `Session` record in your database, linking it to the user with `id = "clx123abc"`.
    - A simple session token cookie is sent to the client, which is just an ID pointing back to the database session.
5.  **Subsequent Requests:**
    - On subsequent requests, the client sends this simple session token cookie.
    - NextAuth.js uses this token to retrieve the full `Session` record from your database.
    - From this `Session` record, it knows the `userId` (e.g., `"clx123abc"`).
    - It then re-generates an internal JWT (again, with `sub` set to `"clx123abc"`). This JWT is passed to your `session` callback (as the `token` argument).

### Why the `sub` is so important:

- **Identity Assertion:** It's the core piece of information asserting "who" the token belongs to.
- **Database Lookup:** With a database adapter, the `sub` (which maps to `user.id`) is the key used to fetch the full user profile from your database for further processing or for populating the `session` object.
- **Authorization:** In server-side authorization checks, you'll often extract the `sub` from the internal JWT (or `session.user.id` derived from it) to determine permissions.
- **Statelessness (for JWT strategy):** If you were using the `jwt` session strategy (where the JWT itself is the session cookie), the `sub` and other claims within that JWT would be the _only_ information needed to identify the user without a database lookup for the session itself.

**Note**: In the current setup, even without you explicitly defining the `jwt` callback, the default behavior ensures that the `user.id` from your database is used as the `sub` in the internal JWT, which is then used to manage your database sessions. When you explicitly define the `jwt` callback, you gain control over _what else_ beyond `sub` gets added to that internal token, allowing you to include custom fields and manage provider-specific tokens.

# Why JWT callback is needed when adding custom fields to other callbacks, why can't just session modification work?

This is a great question because it probes the boundaries of how NextAuth.js's internal mechanisms work!

Let's break down why this approach **will NOT fully work as you intend** for `createdAt` and `updatedAt` fields.

### The Proposed Approach:

1.  **Modify `declare module "next-auth"`:**

    ```typescript
    declare module "next-auth" {
      interface Session extends DefaultSession {
        user: {
          id: string;
          createdAt?: Date; // Added here
          updatedAt?: Date; // Added here
        } & DefaultSession["user"];
      }
    }
    // No declare module "next-auth/jwt" modification
    ```

2.  **Modify `session` callback ONLY:**
    ```typescript
    callbacks: {
      session: ({ session, user }) => {
        console.log("SESSION RUNNING\n\n\n");
        return {
          ...session,
          user: {
            ...session.user,
            id: user.id,
            createdAt: user.createdAt, // <-- Attempt to add this directly from `user`
            updatedAt: user.updatedAt, // <-- Attempt to add this directly from `user`
          },
        };
      },
    },
    ```
    (Note: This assumes `user.createdAt` and `user.updatedAt` are directly available in the `user` object passed to the `session` callback.)

### Why this will **NOT** fully work for `createdAt` and `updatedAt`:

The core issue lies in **what data is actually available in the `user` argument of the `session` callback** when using the `PrismaAdapter` (database strategy).

1.  **The `user` object in `session` callback (with Database Strategy):**
    When using a database adapter and the `database` session strategy, the `user` object passed to the `session` callback is indeed the `User` record fetched directly from your database based on the `userId` in the session token.

    So, conceptually, `user.createdAt` and `user.updatedAt` _should_ be available on this `user` object, as these fields exist in your `User` model.

2.  **The Role of the `jwt` callback (Even for Database Strategy):**

    - **The Default `jwt` Callback:** When you _don't_ define your own `jwt` callback, NextAuth.js provides a default one. This default callback primarily populates the `sub` (subject, which is `user.id`) of the internal JWT. It _does not_ go out of its way to fetch all arbitrary fields from your `User` model (`createdAt`, `updatedAt`, `role`, etc.) and put them into this internal JWT.
    - **Why the internal JWT matters:** While the `session` callback _does_ receive the `user` object directly from the database, it _also_ receives the `token` (the internal JWT). NextAuth.js's internal logic often relies on this `token` for consistency, especially when constructing the final `session` object. The `session` callback often takes properties from _either_ `token` or `user` depending on what's available and the strategy.

3.  **Potential for Data Stale-ness and Inconsistency (The Bigger Problem):**
    - If you _only_ rely on fetching `user` data in the `session` callback:
      - The `user` object here represents the _current state_ of the user in the database.
      - While this might seem fine for `createdAt` and `updatedAt` (which don't change often), it means every time a session is checked or refreshed, NextAuth.js might be doing an extra database query just to get these fields for the session.
      - More critically, this approach breaks down for fields that need to be derived or frequently refreshed, like **provider access tokens (e.g., Google's)** or complex roles based on multiple factors. The `jwt` callback is the designated place for that dynamic logic.

### The Correct and Recommended Approach (as discussed previously):

The recommended pattern is to use the `jwt` callback to _prepare_ all the necessary custom data into the internal `token` (JWT). Then, the `session` callback takes this `token` and just transfers its contents to the final `session` object.

**Steps Revisited for Clarity:**

1.  **Prisma Schema:** Ensure `createdAt`, `updatedAt` (and any other fields like `role`) exist in your `User` model.
2.  **Augment `next-auth/jwt` type:**
    ```typescript
    declare module "next-auth/jwt" {
      interface JWT {
        id: string;
        createdAt?: number; // As timestamp
        updatedAt?: number; // As timestamp
        // role?: "USER" | "ADMIN";
        // accessToken?: string;
      }
    }
    ```
3.  **Augment `next-auth` (`Session`) type:**
    ```typescript
    declare module "next-auth" {
      interface Session extends DefaultSession {
        user: {
          id: string;
          createdAt?: Date; // As Date object for client
          updatedAt?: Date; // As Date object for client
          // role?: "USER" | "ADMIN";
          // accessToken?: string;
        } & DefaultSession["user"];
      }
    }
    ```
4.  **Implement/Modify `jwt` callback:**
    ```typescript
    callbacks: {
      async jwt({ token, user, account }) {
        if (user) { // On initial sign-in
          token.id = user.id;
          // Fetch from DB once and add to internal JWT
          const dbUser = await db.user.findUnique({
            where: { id: user.id },
            select: { createdAt: true, updatedAt: true /*, role: true */ },
          });
          if (dbUser) {
            token.createdAt = dbUser.createdAt.getTime();
            token.updatedAt = dbUser.updatedAt.getTime();
            // token.role = dbUser.role;
          }
          // Add provider tokens if applicable (Google etc.)
          if (account?.provider === "google") { /* ... */ }
        }
        // Handle refreshing provider tokens here if they expire
        // ... refreshAccessToken logic ...
        return token;
      },
      // ...
    }
    ```
5.  **Implement/Modify `session` callback:**
    ```typescript
    callbacks: {
      // ... jwt callback ...
      async session({ session, token }) { // Notice 'token' is key here
        if (session.user && token) {
          session.user.id = token.id;
          if (token.createdAt) session.user.createdAt = new Date(token.createdAt);
          if (token.updatedAt) session.user.updatedAt = new Date(token.updatedAt);
          // if (token.role) session.user.role = token.role;
          // if (token.accessToken) session.user.accessToken = token.accessToken;
        }
        return session;
      },
    },
    ```

### Why the recommended approach is better:

- **Consistency:** The `jwt` callback always creates the consistent "source of truth" for what data should be in the session.
- **Performance:** You fetch custom fields from the database once per session creation/refresh in the `jwt` callback. If you relied solely on the `session` callback's `user` object and it was fetching data every time, it could lead to more frequent database queries than necessary.
- **Token Refreshing:** The `jwt` callback is the _only_ place where you can correctly implement the logic for refreshing OAuth provider access tokens (e.g., Google). This is not possible in the `session` callback.
- **Flexibility:** If you ever switch your `session.strategy` to `jwt`, your data enrichment logic in the `jwt` callback is already perfectly set up.

So, while `user.createdAt` might technically be available in the `session` callback's `user` argument when using an adapter, it's generally **not the robust or recommended way** to manage custom fields or complex session logic. Always use the `jwt` callback to prepare the data for the session.

You've hit on a critical point for understanding NextAuth.js's default behavior!

# What's happening by default without a `jwt` callback?

When you _don't_ explicitly define the `jwt` callback, NextAuth.js provides a **default `jwt` callback** for you. This default behavior depends on whether you have an adapter configured:

1.  **With an Adapter (Your Case - PrismaAdapter):**

    - The default `jwt` callback, upon successful sign-in, will primarily add the `user.id` (from your database user object) to the internal JWT (`token.sub` which means "subject" of the token). It doesn't typically add much else by default like `name`, `email`, or `image` _from the database user_ to the internal JWT, unless they are implicitly part of `user.id`'s lookup.
    - Crucially, it **does not automatically transfer custom fields** like `createdAt`, `updatedAt`, or `role` from your database `User` model into this internal JWT.
    - It also **does not automatically transfer provider-specific tokens** like `access_token` or `refresh_token` from the `account` object into the internal JWT, nor does it handle refreshing them. These are still stored in your `Account` model in the database, but they aren't part of the internal JWT that the `session` callback receives.

2.  **Without an Adapter (and `session.strategy: "jwt"`):**
    - If you had no adapter, the default `jwt` callback would typically add `user.id`, `user.name`, `user.email`, and `user.image` from the initial provider response directly into the internal JWT, which then becomes the client-side session.

### Why your current `session` callback works for `id`, `name`, `email`, `image`:

Your current `session` callback is defined as:

```typescript
callbacks: {
  session: ({ session, user }) => {
    console.log("SESSION RUNNING\n\n\n");
    return {
      ...session,
      user: {
        ...session.user,
        id: user.id, // <-- Here's the key!
      },
    };
  },
},
```

- When you use a **database adapter** (like PrismaAdapter), the `session` callback receives a `user` argument that is populated directly from your **database's `User` model**.
- This `user` object already contains `id`, `name`, `email`, and `image` (because these fields are part of NextAuth.js's standard `User` interface, which is mapped by the adapter to your `User` model).
- So, your current `session` callback is explicitly taking `user.id` (from the database `User` object) and adding it to `session.user`.
- The `session.user` also implicitly gets `name`, `email`, and `image` from `DefaultSession["user"]`, which NextAuth.js populates automatically when using an adapter, directly from the `user` object.

Therefore, `id`, `name`, `email`, `image` are available to `useSession()` and `getServerSession()` because:

- `id` is explicitly added by your `session` callback from the `user` (database) object.
- `name`, `email`, `image` are implicitly added by NextAuth.js to `session.user` from the `user` (database) object due to the default `DefaultSession["user"]` mapping.

### For adding _more_ fields (like `role`, `createdAt`, `updatedAt`):

**Yes, you absolutely need to be explicit and define the `jwt` callback (and potentially adjust the `session` callback as well).**

Here's why and how:

1.  **`createdAt` and `updatedAt`:** These are custom fields that are _not_ part of the standard `DefaultSession` or `User` interface that NextAuth.js expects. They exist in _your specific_ Prisma `User` model.

    - **The Problem:** The default `jwt` callback won't know to pull these fields from your database and put them into the internal JWT.
    - **The Solution:** You need to implement the `jwt` callback to `await db.user.findUnique(...)` and explicitly add `createdAt` and `updatedAt` to the `token` object.

2.  **`role` (if you add it):** Similar to `createdAt` and `updatedAt`, `role` is a custom field.

    - **The Problem:** The default `jwt` callback won't include it.
    - **The Solution:** You need to implement the `jwt` callback to fetch `role` from the database and add it to the `token` object.

3.  **Provider Access/Refresh Tokens (e.g., Google):**
    - **The Problem:** The default `jwt` callback does _not_ put the `access_token` or `refresh_token` from the `account` object into the internal JWT, nor does it handle refreshing them. While these are stored in your database `Account` model, they aren't readily available for the `session` callback from the `token` argument, and more importantly, they won't be automatically refreshed.
    - **The Solution:** You _must_ implement the `jwt` callback to store these tokens in the `token` object and to add the logic for refreshing the `access_token` when it expires.

### The Flow with Custom Fields:

1.  **User signs in:** NextAuth.js authenticates with Google/GitHub.
2.  **Default `signIn` callback (implicitly `true`):** Allows the sign-in.
3.  **Your Custom `jwt` callback (explicitly defined):**
    - Receives `user` (from DB) and `account` (from provider).
    - You explicitly query `db.user.findUnique` to get `createdAt`, `updatedAt`, `role` from your database.
    - You add `user.id`, `createdAt`, `updatedAt`, `role`, and provider tokens (like Google's `access_token`, `refresh_token`) to the `token` (internal JWT).
    - This is also where you'd implement the `refreshAccessToken` logic if the Google `access_token` has expired.
    - Returns the _enriched_ `token`.
4.  **Your Custom `session` callback:**
    - Receives the _enriched_ `token` from the `jwt` callback, and the `user` object directly from the database (via PrismaAdapter).
    - You explicitly transfer `token.id`, `token.createdAt`, `token.updatedAt`, `token.role`, and `token.accessToken` (if needed) to `session.user`.
    - Returns the `session` object ready for the client.

**In summary:** While NextAuth.js provides some sensible defaults, for any fields beyond the very basic `id`, `name`, `email`, `image` (especially those specific to your database schema or provider tokens), you need to take control of the `jwt` and `session` callbacks. The `jwt` callback is where you enrich the data, and the `session` callback is where you expose that enriched data to your frontend.

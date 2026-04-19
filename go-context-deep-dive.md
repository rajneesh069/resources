# Go Tutorial — Questions & Answers

A learning record from a deep-dive session covering `context`, Gin, pointers, receivers, maps, slices, and addressability. Each section is self-contained so you can skim by topic.

---

## Q1. Is the project using `context` properly — and is there a conflict between Gin's context and Go's `context`?

### What's done correctly

- **Repository layer signatures** all take `ctx context.Context` as the first parameter — idiomatic Go.
- **pgx calls** all use context-aware methods (`pool.Query(ctx, …)`, `pool.QueryRow(ctx, …)`, `pool.Exec(ctx, …)`).
- **Timeouts** — every repo function wraps the incoming `ctx` with `context.WithTimeout(ctx, 5*time.Second)` and `defer cancel()`.
- **Startup** uses `context.Background()` for `pgxpool.NewWithConfig` and `pool.Ping` — the correct root for one-time init.
- **Todo handlers** correctly pass `c.Request.Context()` to repo calls.
- **No goroutines** spawned from handlers (yet), so no risk of a goroutine outliving a request with a stale `*gin.Context`.

### Where the conflict is — 3 bugs

`*gin.Context` implements `context.Context`, so passing `c` where a `context.Context` is expected **compiles silently** but is wrong. Three places passed the raw `c`:

- `internal/handlers/user_handler.go:44` — `repository.CreateUser(c, pool, …)` inside `Register`
- `internal/handlers/user_handler.go:78` — `repository.GetUserByEmail(c, pool, …)` inside `Login`
- `internal/handlers/todo_handler.go:125` — `repository.GetTodoByID(c, pool, …)` inside `UpdateTodo`

### Why it matters

1. **Cancellation semantics differ.** `c.Request.Context()` is cancelled when the client disconnects or the request aborts — that propagates into pgx and cancels the in-flight query. `*gin.Context` is not cancelled with those same semantics.
2. **`*gin.Context` is pooled and reused** via `sync.Pool`. Once the handler returns, Gin resets it for the next request. Any code that outlives the handler and reads from `c` is a data race / stale read.
3. **`Value()` lookup differs.** `c.Value(key)` checks Gin's keys first, then falls through to the request context. `c.Request.Context().Value(key)` only walks the standard context chain.
4. **Your `WithTimeout` partially hides this** — your repo wraps ctx in a 5s timeout, so timeouts still fire on those paths. But you lose **client-disconnect cancellation**, which is the whole point of threading request context.

**Rule of thumb**: never pass `c` (the `*gin.Context`) to anything that takes `context.Context`. Always pass `c.Request.Context()`.

---

## Q2. What is the source of this "footgun" claim? Link it.

Authoritative sources:

- [Context and Cancellation — Gin Web Framework](https://gin-gonic.com/en/docs/server-config/context/) — tells you explicitly to pass `c.Request.Context()` (not `c`) to downstream calls.
- [Goroutines inside a middleware — Gin](https://gin-gonic.com/en/docs/middleware/goroutines-inside-a-middleware/) — the canonical "use `c.Copy()`, don't use the original `c`" guidance.
- [gin package — pkg.go.dev](https://pkg.go.dev/github.com/gin-gonic/gin) — shows `Copy()` and the interface methods.
- [gin/context.go at master — github](https://github.com/gin-gonic/gin/blob/master/context.go) — the actual source.
- [Issue #1118 — Data race when not calling Copy()](https://github.com/gin-gonic/gin/issues/1118) — real bug report showing the race.
- [Issue #3554 — Context Copy does not detach request.context](https://github.com/gin-gonic/gin/issues/3554) — subtle interaction if you're spawning goroutines with copied contexts.
- [Issue #1317 — Goroutine within gin handler](https://github.com/gin-gonic/gin/issues/1317)
- [Issue #4117 — gin.Context data race in trivial situation](https://github.com/gin-gonic/gin/issues/4117)

Start with the first two; the issues are where you see the footgun biting people in real code.

---

## Q3. Why is it a footgun — teach me from scratch.

### What is `context.Context`?

An interface with four methods. Any type that implements them **is** a `context.Context`:

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}    // closed when cancelled
    Err() error                // why it was cancelled
    Value(key any) any         // lookup request-scoped values
}
```

It answers three needs:

1. **"Stop, I don't need the result anymore"** — cancellation.
2. **"You have at most N seconds"** — deadline/timeout.
3. **"Here's some request-scoped data to carry along"** — values (user id, trace id).

Why it matters: if a client hangs up mid-request, you want the DB query they triggered to stop. The signal travels: HTTP server cancels the request context → you passed that context to `pool.QueryRow(ctx, …)` → pgx sees `Done()` close → pgx tells Postgres to cancel the query. **If you break the chain by passing the wrong context, cancellation doesn't flow.**

### What is `*gin.Context`?

Gin's object for "everything about this request." It holds:

- the `*http.Request` and `http.ResponseWriter`
- a key-value map for middleware data (`c.Set("user_id", …)` / `c.Get("user_id")`)
- helpers: `c.JSON(...)`, `c.Param("id")`, `c.BindJSON(...)`, etc.

It's **not** the same as `context.Context`. It's Gin-specific. But — here's the trap — it **also implements the four `context.Context` methods**, so Go lets you pass `c` wherever `context.Context` is expected.

Inside any handler, there are two context-y things:

| Thing | Type | What it is |
|---|---|---|
| `c` | `*gin.Context` | Gin's request object. Lives only during the handler. |
| `c.Request.Context()` | `context.Context` | The standard Go request context tied to the HTTP request. |

### The actual mechanism — why it breaks

**(a) Gin pools `*gin.Context` objects**

When a request arrives, Gin doesn't allocate a fresh `*gin.Context` — it grabs one from a `sync.Pool` and fills it. When your handler returns, Gin **resets** that object (wipes keys, nils Request) and puts it back for the next request. `c` is a reusable clipboard: after the handler returns, the clipboard is wiped and handed to someone else. If any code keeps a reference to `c` past the handler's return, it's reading someone else's paperwork.

**(b) Cancellation semantics differ**

When the HTTP client disconnects, `net/http` closes the request's `Done` channel — on `c.Request.Context()`. `*gin.Context.Done()` returns that same channel in some versions but has had bugs (see issue #3554). The documented, stable, intended way to get the request's cancel signal is `c.Request.Context()`.

**(c) `Value()` lookup is different**

With `c.Request.Context()`: walks the standard Go context chain.
With `c`: checks Gin's internal KV map first, then falls back. Setting `c.Set("user_id", …)` means `c.Value("user_id")` finds it, but `c.Request.Context().Value("user_id")` does not.

**(d) Why it "works" in your code right now**

Your repo layer wraps the incoming context:

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
```

Even if you accidentally pass `c`, the new `ctx` is a derived context that fires cancellation on timeout. So queries still time out. But you've lost the "client disconnected → cancel the query" propagation.

### The rule

> **`c` is Gin's scratchpad. `c.Request.Context()` is the real request context. Anytime a function asks for `context.Context`, give it `c.Request.Context()`.** The only exception: when you spawn a goroutine that needs Gin keys after the handler returns, use `c.Copy()`.

---

## Q4. So I was passing `c` instead of `c.Request.Context()` — that's the DB cancel risk, right?

Exactly right. Specifically:

- **Client disconnects / browser tab closes** → `net/http` closes `c.Request.Context().Done()`. If you pass `c.Request.Context()` to pgx, pgx cancels the in-flight query. If you pass `c`, that signal may not propagate — you keep burning DB work for a client who's already gone.
- **Server shutdown** (graceful) — same story.
- **Your own 5-second timeout** still works either way, because `context.WithTimeout(ctx, 5s)` creates a *new* context whose `Done()` fires on the timer regardless of the parent.

Fixing the three call sites restores the "client hung up → stop the query" behavior. Everything else about your context usage (threading, wrap with timeout, `defer cancel()`) is already correct.

---

## Q5. What's the race condition, and when do I use `c.Copy()` vs `c`?

### What "race condition" actually means

Two goroutines touching the same memory at the same time, with at least one writing. Result is **undefined**: corrupted data, panics, or silently wrong answers.

With `*gin.Context`, the shared memory is the pooled `c` object. Timeline:

```
T=0ms   Request A arrives. Gin pulls c from the pool, fills it with A's stuff.
T=1ms   Handler A runs. Does: go doWork(c). Returns.
T=2ms   Gin resets c: wipes Keys, nils Request, puts c back in the pool.
T=3ms   Request B arrives. Gin grabs THE SAME c, fills it with B's stuff.
T=4ms   Goroutine from step 1 wakes up, calls c.GetString("user_id").
        → Reads B's user_id. Or reads a half-written field, gets garbage.
```

Catch it in tests with `go test -race`.

### What `c.Copy()` does

Looking at Gin's source:

```go
func (c *Context) Copy() *Context {
    cp := Context{...}
    cp.Keys = map[string]any{}
    for k, v := range c.Keys {
        cp.Keys[k] = v
    }
    // handlers set to nil, index set to abortIndex
    return &cp
}
```

- One struct allocation on the heap.
- One map allocation for `Keys`, shallow copy of entries.
- Does NOT copy `Request`, `Writer`, `engine` — those stay as shared pointers.
- The copy is **not returned to the pool** — it lives as long as your goroutine holds it.

Concrete cost: a few hundred bytes + map rehash. Negligible unless you're copying in a hot loop millions of times.

### When you actually need `c.Copy()`

Only when you **spawn a goroutine from a handler AND the goroutine reads Gin-specific data** (keys, params, etc.) after the handler returns:

```go
func SomeHandler(c *gin.Context) {
    cCp := c.Copy()                     // snapshot for the goroutine

    go func() {
        userID := cCp.GetString("user_id")   // safe — reads the snapshot
        // slow work...
    }()

    c.JSON(202, gin.H{"status": "queued"})
}
```

### The trap inside the trap

Go's `net/http` cancels the request's context the moment your handler returns. So:

```go
func SomeHandler(c *gin.Context) {
    ctx := c.Request.Context()

    go func() {
        // Handler returns ~immediately after the 'go' statement.
        // Two ns later, ctx is cancelled. Query never runs.
        repository.DoSlowThing(ctx, ...)
    }()

    c.JSON(202, gin.H{"status": "queued"})
}
```

### Decision table for goroutines spawned from handlers

| Goroutine's job | What to pass |
|---|---|
| Work tied to the request — must die if client hangs up | `c.Request.Context()` (rare, because the handler usually returns first) |
| Background work that should outlive the request (email, audit log, enqueue) | `context.Background()`, usually wrapped in `context.WithTimeout(...)` |
| Needs Gin keys / params inside the goroutine | `c.Copy()` for the Gin bits, plus a context of your choice |

Realistic example:

```go
func CreateOrder(c *gin.Context) {
    userID := c.GetString("user_id")
    cCp := c.Copy()

    order, err := repository.CreateOrder(c.Request.Context(), pool, userID, ...)
    if err != nil { return }

    go func() {
        ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()
        email := cCp.GetString("email")
        _ = mailer.SendOrderConfirmation(ctx, email, order.ID)
    }()

    c.JSON(201, order)
}
```

---

## Q6. What is `context.Background()` and the full package — the unknown unknowns?

### Mental model: contexts form a tree

Every context has a parent. You build a tree:

```
context.Background()         ← root (created at program start)
  │
  ├── c.Request.Context()    ← net/http made this when the request arrived
  │     │
  │     └── ctx, cancel := context.WithTimeout(req.Ctx, 5s)  ← your repo
  │
  └── (other requests' contexts, goroutines, etc.)
```

Three rules:

1. **Cancellation flows down.** Cancel a node, every descendant cancels too.
2. **Deadlines are the minimum of the chain.** You can only tighten, never loosen.
3. **Values shadow down.** `ctx.Value(key)` walks upward through parents.

### The two roots

**`context.Background()`** — empty, never-cancelled, no-deadline, no-values. Use at: `main()`, startup/shutdown not tied to any request, background goroutines that must outlive the request that spawned them.

**`context.TODO()`** — identical at runtime; intent differs. Use when "I know I should plumb a real context but haven't figured out which." Linters flag `TODO`s as tech debt.

### The `With*` family (children)

**`context.WithCancel(parent) (ctx, cancel)`** — manual cancellation:

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
go worker(ctx)
if userPressedStop { cancel() }
```

**`context.WithTimeout(parent, duration) (ctx, cancel)`** — sugar for `WithDeadline(parent, time.Now().Add(duration))`:

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
```

**`context.WithDeadline(parent, time.Time) (ctx, cancel)`** — absolute wall-clock deadline:

```go
ctx, cancel := context.WithDeadline(parent, maintenanceWindowStart)
defer cancel()
```

**`context.WithValue(parent, key, value) ctx`** — odd one out: no `CancelFunc`. Use a custom unexported type as the key:

```go
type ctxKey int
const userIDKey ctxKey = iota
ctx := context.WithValue(parent, userIDKey, "u-123")
```

Don't use `WithValue` for regular function args. Reserve it for request-scoped data crossing many layers without every intermediate caring (trace IDs, auth principals, request IDs, locale).

### `context.CancelFunc`

A type, not a function:

```go
type CancelFunc func()
```

Rules:

1. **Always call it eventually.** Forgetting leaks (a timer goroutine for `WithTimeout`, a registration on the parent for `WithCancel`). `go vet` catches many forgotten cancels.
2. **Calling cancel multiple times is fine** — that's why `defer cancel()` is safe even if you also cancel early.

### Inspecting a context

```go
ctx.Done()      // <-chan struct{} — closes on cancel/timeout
ctx.Err()       // nil if alive, else context.Canceled or context.DeadlineExceeded
ctx.Deadline()  // (time.Time, bool)
ctx.Value(key)  // walks chain upward
```

Canonical detection pattern:

```go
select {
case <-ctx.Done():
    return ctx.Err()
case result := <-workChan:
    return result
}
```

### Lesser-known / newer stuff

**`context.Cause(ctx) error` (Go 1.20+)** — returns the original cancellation cause, not a generic error.

**`context.WithCancelCause(parent) (ctx, CancelCauseFunc)` (Go 1.20+)** — the cancel takes an error:

```go
ctx, cancel := context.WithCancelCause(parent)
defer cancel(nil)
if user.Banned {
    cancel(errors.New("user banned mid-request"))
}
// later:
log.Println(context.Cause(ctx))  // "user banned..."
```

**`context.WithDeadlineCause` / `WithTimeoutCause` (Go 1.21+)** — attach custom error as cause when deadline fires.

**`context.WithoutCancel(parent) ctx` (Go 1.21+)** — keeps values, drops cancellation. For background work needing request-scoped values (trace IDs) that shouldn't die with the request:

```go
go func() {
    bgCtx := context.WithoutCancel(reqCtx)
    bgCtx, cancel := context.WithTimeout(bgCtx, 30*time.Second)
    defer cancel()
    sendEmail(bgCtx, ...)
}()
```

**`context.AfterFunc(ctx, func()) (stop func() bool)` (Go 1.21+)** — run a callback when ctx cancels, no manual select:

```go
stop := context.AfterFunc(ctx, func() {
    conn.Close()
})
defer stop()
```

### Cheat sheet

| You want to… | Use |
|---|---|
| Start a context from nothing | `context.Background()` |
| Mark "I'll figure out later" | `context.TODO()` |
| Cancel manually | `WithCancel` |
| Cancel after a duration | `WithTimeout` |
| Cancel at a specific time | `WithDeadline` |
| Pass a value down | `WithValue` (typed key) |
| Know why cancelled | `WithCancelCause` + `context.Cause` |
| Carry values but not cancellation | `WithoutCancel` (1.21+) |
| Run code on cancel | `AfterFunc` (1.21+) |
| Detect cancellation | `<-ctx.Done()` then `ctx.Err()` |
| Know the deadline | `ctx.Deadline()` |

### Five common mistakes

1. Forgetting `defer cancel()`.
2. Using a `string` as a `WithValue` key.
3. Storing a context in a struct field.
4. Passing `nil` as a context (panics — use `context.TODO()`).
5. Using the request context for background work (dies when handler returns).

---

## Q7. Is `c.Copy()` too memory-heavy? Why do we pass library structs as pointers?

### Is `c.Copy()` heavy?

Not really. Cost = one struct allocation (a few hundred bytes) + one shallow map copy. Negligible compared to JSON parsing, DB calls, and HTTP responses you're already doing.

The "`c.Copy()` is expensive, avoid it" framing is wrong. Real rule: don't `Copy()` needlessly, but when you genuinely need it for a goroutine, the cost is trivial.

### Why pointers for library structs — the five real reasons

**(a) Pointer-receiver methods**

Most of Gin's methods are on `*Context`, not `Context`:

```go
func (c *Context) JSON(code int, obj any) { ... }
func (c *Context) Set(key string, value any) { ... }
```

You can call pointer-receiver methods on a pointer but not on a plain value (unless it's addressable, with restrictions). **Main reason.**

**(b) Mutation must persist**

`c.Set("user_id", "u-123")` writes to `c.Keys`. A value copy would mutate the copy; the caller's data stays unchanged. Pass-by-pointer is the only way middleware can mutate something a later handler sees.

**(c) Structs containing mutexes / channels must not be copied**

`gin.Context` has a `sync.RWMutex` to protect the Keys map. **Copying a `sync.Mutex` is a bug** — two mutexes each protecting a different copy, so two goroutines can "lock" simultaneously. `go vet copylocks` catches this. Same for `sync.WaitGroup`, `sync.Once`, channels, `atomic.Int64`.

**(d) Identity matters**

Some types represent a handle to shared state (DB pool, HTTP request, open file). Two copies still "point to" the same pool, but state tracking goes haywire. `*http.Request`, `*pgxpool.Pool`, `*os.File`, `*sync.Mutex` — these *are* handles. Copying them is meaningless or broken.

**(e) Copy cost**

For big structs in hot paths, copying every call is wasteful. But usually the least important reason — other reasons force your hand first.

### Notes on `c.Copy()` specifically

`c.Copy()` returns a `*gin.Context` — a pointer to a fresh struct. You're still passing pointers; you're pointing at a safe allocation instead of the pooled one about to be recycled.

The decision isn't "pointer or copy?" — it's "share the pooled pointer or point to a snapshot?" Share inside the handler. Snapshot when crossing into a goroutine.

### Library convention

When a library's constructor returns `*Foo`, that's the author saying: "this is a handle. Share it. Don't dereference it to a value. Store `*Foo`, not `Foo`."

| Type | Why pointer |
|---|---|
| `*gin.Context` | pooled, has mutex, pointer-receiver methods |
| `*gin.Engine` | has mutex, pointer-receiver methods |
| `*http.Request` | stateful body reader, identity matters |
| `*pgxpool.Pool` | connection pool handle |
| `*sql.DB` | same |
| `*zap.Logger` | internal state, pointer-receiver methods |

When a library returns a **value** (`time.Time`, `uuid.UUID`), it's usually a small, immutable, copyable thing — opposite signal.

---

## Q8. Full teaching on pointer vs value receivers.

### What a receiver is

A method is a function with a receiver — a special first argument:

```go
type Todo struct {
    Title     string
    Completed bool
}

func markDone(t Todo) { ... }           // plain function
func (t Todo) IsDone() bool { ... }     // method, value receiver
func (t *Todo) MarkDone() { ... }       // method, pointer receiver
```

### Runtime behavior

**Value receiver** — Go **copies** the struct into `t`. Mutations are on the copy, invisible to the caller.

**Pointer receiver** — Go passes a **pointer**. Mutations affect the caller's struct.

### Calling rules — addressability

Go auto-inserts `&` and `*` where it can:

```go
var t Todo
t.MarkDone()  // Go rewrites to (&t).MarkDone() — works because t is addressable
```

Addressable = has a memory location Go can take the address of. Locals, struct fields, array elements are all addressable.

Not addressable:

```go
getTodo().MarkDone()   // ERROR: cannot take address of function return

todos := map[string]Todo{"a": {Title: "x"}}
todos["a"].MarkDone()  // ERROR: cannot take address of map value

var t interface{ IsDone() bool } = Todo{}
t.MarkDone()           // ERROR: MarkDone isn't in the interface's method set
```

**Summary**:
- **Value receiver** methods can be called on anything.
- **Pointer receiver** methods need a pointer or an addressable value.

### Interface satisfaction — the asymmetric method sets

| Type | Method set includes |
|---|---|
| `T` (value) | methods with value receivers |
| `*T` (pointer) | methods with value receivers AND pointer receivers |

```go
type Marker interface {
    MarkDone()  // pointer-receiver method on Todo
}

var t Todo = Todo{}
var m Marker = t    // ❌ COMPILE ERROR — Todo's method set lacks MarkDone
var m Marker = &t   // ✅ works
```

In Gin's case, the `Context` interface methods are defined on `*gin.Context`. Only `*gin.Context` satisfies `context.Context` — not `gin.Context` as a value. That's why your code always uses `c *gin.Context`.

### When to pick which

**Pointer receiver when:**

1. The method mutates the receiver. **Decisive alone.**
2. Struct is large.
3. Struct contains `sync.Mutex`, `sync.WaitGroup`, atomic, etc. (`go vet copylocks` enforces.)
4. Struct contains slices/maps/channels where callers expect shared behavior.
5. You want to call the method on a nil pointer (legal if no deref).

**Value receiver when:**

1. Type is small (fits in a couple of words).
2. Type is "value-like" — instances with the same fields are interchangeable (`time.Time`, `uuid.UUID`, `image.Point`, `netip.Addr`).
3. Methods don't mutate.

**Consistency rule**: if any method needs a pointer receiver, **give every method a pointer receiver**. Mixing creates method-set surprises.

### Standard library examples

| Type | Why |
|---|---|
| `time.Time` | value receivers — small, immutable. `t.Add(d)` returns a new Time. |
| `bytes.Buffer` | pointer receivers — mutable, internal state. |
| `sync.Mutex` | pointer receivers — copying defeats the lock. |
| `strings.Builder` | pointer receivers — mutable byte slice; vet detects copies at runtime. |

### Two footguns

**Footgun 1 — value-receiver method that looks like it mutates:**

```go
func (t Todo) SetTitle(s string) { t.Title = s }   // silently useless

var t Todo
t.SetTitle("new")
fmt.Println(t.Title)   // "" (mutation was on a copy)
```

No compile error. Just does nothing.

**Footgun 2 — map values aren't addressable:**

```go
todos := map[uint]Todo{}
todos[1] = Todo{Title: "x"}
todos[1].MarkComplete()   // ❌ won't compile if pointer receiver
```

Fix: `map[uint]*Todo`, or extract/mutate/put back:

```go
t := todos[1]
t.MarkComplete()
todos[1] = t
```

---

## Q9. For small fields on my repository functions, do I need pointers?

**No. Values are correct.**

### What's being passed

```go
func CreateTodo(ctx context.Context, pool *pgxpool.Pool, title string, completed bool, userId string) (*models.Todo, error)
```

- `title` — `string` is 16 bytes (pointer + length; bytes aren't copied).
- `completed` — `bool` is 1 byte.
- `userId` — `string`, 16 bytes.

These are tiny. Copying is one or two register moves. A pointer would be 8 bytes + an extra heap indirection on read — often *slower*.

### Why value is correct for primitives

1. **Cost**: tiny primitives are cheap; strings are already reference-typed under the hood.
2. **No mutation needed**: your repo functions don't write to `title`, just read it.
3. **No `nil` to handle**: `string` is always some string. `*string` adds a nil branch for no benefit.
4. **Honest signatures**: `title string` says "give me a title." `*string` says "give me a title, or don't."

### The one case where `*string` IS right — you already did it

`internal/handlers/todo_handler.go:138-144`:

```go
if input.Title != nil {
    title = *input.Title
}
if input.Completed != nil {
    completed = *input.Completed
}
```

`UpdateTodoInput` uses pointers because of PATCH semantics — three states on update:

| Client sent | Meaning | `string` | `*string` |
|---|---|---|---|
| `{"title": "new"}` | Update to "new" | ✅ | ✅ |
| `{"title": ""}` | Update to empty | ✅ | ✅ |
| `{}` (absent) | Don't touch | ❌ indistinguishable from `""` | ✅ nil |

Pointers carry a third state ("absent"). That's the legitimate use.

### The rule

Use a pointer for a primitive parameter only if **one** is true:

- You need mutation (rare — usually return a new value instead).
- You need `nil` to mean "absent/optional" (PATCH inputs, sparse config).

Otherwise, pass by value.

### Bridge to receivers

The rules for receivers and regular parameters are **the same** — receivers are just a special first parameter. Value vs pointer uses the same five factors (mutation, size, mutexes, identity, optional-nil).

---

## Q10. What's special about maps that we store pointers in them?

### The rule: map values aren't addressable

```go
m := map[string]Todo{"a": {}}
p := &m["a"]      // ❌ compile error: cannot take address
```

This is the root of every map-related mutation weirdness.

### Why — how maps are implemented

A Go map is a hash table: an array of "buckets" with up to 8 key/value slots each. When the map grows past a load factor, Go allocates a bigger bucket array and **rehashes everything into it**. Entries physically move in memory.

If Go let you take `&m["a"]`, insert a new key, trigger a resize, the old bucket array gets freed, and your pointer is dangling. Go's designers picked: **you can't take the address of a map value.** Compile-time check, no runtime surprises.

Slices don't have this — their elements live in a contiguous backing array whose element positions don't shift.

### What this prevents

```go
m["a"].Title = "new"   // ❌ cannot assign to struct field in map
m["a"].Rename("new")   // ❌ cannot call pointer-receiver method
p := &m["a"]           // ❌ cannot take address
```

### What you CAN still do

```go
t := m["a"]            // ✅ copies the value out
m["a"] = Todo{...}     // ✅ replaces the entire value (special case)
delete(m, "a")         // ✅
for k, v := range m { ... }   // ✅ v is a copy each iteration
```

You can always replace whole values; you just can't mutate in place.

### Two fixes

**(a) Extract, mutate, put back:**

```go
t := m["a"]           // copy out
t.Title = "new"       // mutate the copy
m["a"] = t            // copy back
```

Works, idiomatic; costs two map lookups and a full struct copy.

**(b) Store pointers — `map[K]*V`:**

```go
m := map[string]*Todo{"a": {Title: "old"}}
m["a"].Title = "new"   // ✅ works
m["a"].Rename("new2")  // ✅ works
```

Why this compiles: `m["a"]` returns a `*Todo`. Go auto-dereferences for field access: `(*m["a"]).Title`. That's dereferencing a pointer, not taking the address of a map slot.

The struct the pointer points to lives on the heap at a stable address. The map only moves the 8-byte pointer during rehash.

### Trade-offs

**`map[K]V`**: simpler, no nil to worry about, each lookup returns a copy, no extra GC pressure. Best for small structs, read-heavy, caches you rebuild rather than edit.

**`map[K]*V`**: in-place mutation works, cheap reads (pointer copy), supports pointer-receiver methods, shared pointee. Best for larger structs, mutable state, objects with identity.

### Range loop variable (pre Go 1.22)

```go
m := map[string]*Todo{}
todos := []Todo{{Title: "a"}, {Title: "b"}}
for _, t := range todos {
    m[t.Title] = &t        // ⚠️ Go < 1.22 bug — all entries alias the same t
}
```

Go 1.22+ fixed it (per-iteration variable). Before that: `t := t` inside the loop.

---

## Q11. How is slice reallocation handled? What does "stable address" mean?

### Slice internals

```go
type slice struct {
    array unsafe.Pointer  // pointer to the first element
    len   int             // current number of elements
    cap   int             // capacity of the backing array
}
```

24-byte header pointing into a separate contiguous backing array.

```
  slice header           backing array
  ┌─────────────┐        ┌────┬────┬────┬────┬────┬────┐
  │ array ──────┼──────► │ 10 │ 20 │ 30 │    │    │    │
  │ len = 3     │        └────┴────┴────┴────┴────┴────┘
  │ cap = 6     │          [0]  [1]  [2]  [3]  [4]  [5]
  └─────────────┘
```

Element `i` is at `array + i * sizeof(element)`. Stable while this backing array exists.

### What `append` does

**Case A: `len < cap` — grow in place**

```
before: len=3, cap=6            after: append(s, 40)
┌────┬────┬────┬────┬────┬────┐ ┌────┬────┬────┬────┬────┬────┐
│ 10 │ 20 │ 30 │    │    │    │ │ 10 │ 20 │ 30 │ 40 │    │    │
└────┴────┴────┴────┴────┴────┘ └────┴────┴────┴────┴────┴────┘
```

Writes to slot `[3]`, bumps `len`. **No memory moves.** `&s[0]` through `&s[2]` point at literally the same bytes.

**Case B: `len == cap` — reallocate**

1. Allocate a new, bigger backing array (~2× for small slices, ~1.25× once large).
2. **Copy** old elements into the new array.
3. Write the appended element.
4. Return a new slice header pointing to the new array.

The runtime does NOT mutate the old array. The old array stays alive as long as something references it (say, your `p := &s[0]` from before).

### What "stable address" actually means

> **While a given backing array exists, the address of element `i` in that array never changes.**

The runtime never says "I'm moving element 0 from X to Y within the same array." In-place relocation doesn't happen. Either it writes into the existing array (A) or allocates a new one elsewhere (B).

### The gotcha after reallocation

```go
s := []int{10, 20, 30}    // backing array A, cap=3
p := &s[0]                // p points into A
*p = 99                   // A now holds [99, 20, 30]

s = append(s, 40)         // cap was full → reallocate
                          // s now points to array B: [99, 20, 30, 40]
                          // p STILL points into array A
*p = 77                   // writes into array A, which is detached
fmt.Println(s[0])         // 99 — not 77
fmt.Println(*p)           // 77
```

`p` isn't dangling — A is kept alive by `p`. But `p` and `s[0]` now look at **different memory**. The pointer's target is stable; its relationship with the slice isn't.

### Contrast with maps

Maps only have case B, and worse: rehashing actively **copies entries** from old bucket array to new as a normal operation. Go uses incremental **evacuation** — the map can be split between old and new buckets during resize. No per-entry stable address.

Go's design chose flat value storage with rehash-in-place for speed. The price: you don't get to take the address. Opt into stable pointers manually via `map[K]*V`.

### Why not auto-box?

Some languages (Python dicts) do. Costs: per-entry heap allocation, GC pressure, extra pointer deref on every read. Go pushes the decision to you — very Go.

### TL;DR

- Slice elements sit in a flat contiguous backing array. Addresses stable **for as long as that specific array exists**.
- `append` either writes within current array (unchanged) or allocates a new one (old one left alone; new one has different addresses).
- Pointers taken before reallocation continue to work but reference the *old* array.
- Maps physically relocate entries during rehash, so no stable addresses.

---

## Q12. Go deeper on addressability.

### Formal definition (Go spec)

Addressable expressions are exactly these six forms:

1. A variable (local, package-level, parameter, named return).
2. A pointer dereference `*p`.
3. A slice index `s[i]`.
4. An array index `a[i]`, but only if `a` itself is addressable.
5. A field selector `x.f`, but only if `x` is addressable (or a pointer).
6. A composite literal, as a special exception, **only** when immediately prefixed with `&`.

`&x` is legal iff `x` is addressable. Assignment `x = y` is legal iff `x` is addressable (with a special exception for map index).

### Why the concept exists — memory stability

"If I took a pointer to this expression, would that pointer remain meaningful?"

The six forms describe locations where Go can guarantee a stable address for as long as the containing object exists:

- Variable: lives in a stack frame or heap allocation.
- `*p`: by definition, the location `p` points to.
- `s[i]`: lives in the backing array, stable.
- `x.f`: lives at `&x + offset(f)`.

Non-addressable cases: transients in registers, hash buckets that rehash, compile-time constants with no runtime location.

### The non-addressable cases, with physical reasons

**(a) Map index `m[k]`** — hash rehashes.

```go
m["a"] = Point{1, 2}   // ✅ special-cased runtime call
m["a"].X = 5           // ❌
m["a"].Move()          // ❌ with pointer receiver
p := &m["a"]           // ❌
```

**(b) Function/method return values** — exist for an expression's worth of time, no persistent storage:

```go
func getPoint() Point { return Point{1, 2} }
getPoint().X = 5   // ❌
p := &getPoint()   // ❌

// Fix:
p := getPoint()
p.X = 5            // ✅
```

**(c) Composite literals (mostly)** — the `&T{...}` exception is purely ergonomic:

```go
Point{1, 2}.X = 5     // ❌
&Point{1, 2}          // ✅ exception
&Point{1, 2}.X        // ❌ exception only covers the literal itself
```

**(d) String indexing `s[i]`** — strings are immutable:

```go
s := "hello"
c := s[0]          // ✅ reads 'h'
s[0] = 'H'         // ❌
p := &s[0]         // ❌

// Fix:
b := []byte(s)
b[0] = 'H'         // ✅
```

**(e) Expression results** — `a + b`, `!x`, conversions — transient:

```go
(a + b).X = 5          // ❌
&(a + b)               // ❌
```

**(f) Constants** — no runtime location:

```go
const Pi = 3.14
p := &Pi               // ❌
```

**(g) Interface-held values** — the surprise:

```go
type Counter struct{ n int }
func (c *Counter) Inc() { c.n++ }
type Incer interface{ Inc() }

var i Incer = &Counter{}  // ✅ wrapped a pointer
var i2 Incer = Counter{}  // ❌ COMPILE ERROR — Counter doesn't satisfy Incer
```

The interface variable itself is addressable; the **value stored inside** isn't. To call `Inc()` on a `Counter`, Go would need to take its address, but interface-stored values may be boxed on the heap or stored inline — no stable address. The type system cuts it: interfaces with pointer-receiver methods can only be satisfied by pointers.

**(h) Method value / function value** — `&obj.Method` can't be taken.

### The addressable cases, with mechanics

**Variables** — stack slot or heap (escape analysis decides).

**Pointer deref** `*p` — trivially addressable; `&*p == p`.

**Slice index** `s[i]` — `s.array + i * elemSize`, stable for life of backing array.

**Array index** — but only if array is addressable:

```go
var arr [3]int
p := &arr[0]            // ✅

p := &[3]int{1,2,3}[0]  // ❌ literal not addressable without &
```

**Struct field selector** — inherits from base:

```go
var o Outer
&o.inner.n             // ✅ o addressable → o.inner addressable → o.inner.n addressable

getOuter().inner.n = 5  // ❌ chain fails at root

p := getPointerToOuter()
p.inner.n = 5             // ✅ (*p) is pointer deref — always addressable
```

### The map assignment exception

```go
m[k] = v       // ✅ LEGAL even though m[k] isn't addressable
```

Compiler treats this specially — compiled as a call to the map's store runtime, not a memory write. That's why whole-value assignments work but field assignments don't.

### Method call rules — auto-& and auto-*

**Auto-&** (take address for pointer-receiver method):

```go
func (t *Todo) Mark() { t.Done = true }

var t Todo
t.Mark()           // compiled as (&t).Mark() — legal, t is addressable

getTodo().Mark()   // ❌ getTodo() not addressable
m[k].Mark()        // ❌ map value not addressable
```

**Auto-*** (deref for value-receiver method) — always works.

Auto-address only works on addressable receivers. That asymmetry is the entire source of "I can't call this method" confusion.

| Expression | Value receiver | Pointer receiver |
|---|---|---|
| Variable `t` | ✅ | ✅ (auto-&) |
| Pointer `p` | ✅ (auto-*) | ✅ |
| Map value `m[k]` | ✅ | ❌ |
| Return value `f()` | ✅ | ❌ |
| Slice index `s[i]` | ✅ | ✅ (auto-&) |
| Interface holding value type | ✅ if in method set | ❌ |

### Diagnosing "cannot take address" errors

Checklist:

1. Is the LHS a bare variable? If no, must fit one of six forms.
2. Slice index? Fine.
3. Field chain from addressable root? Fine.
4. Map index? Extract/mutate/put back, or `map[K]*V`.
5. Function return? Assign to a variable first.
6. Composite literal? Use `&T{...}` wholesale.
7. Interface holding value type? Wrap a `*T` instead.
8. String index? Convert to `[]byte`.

### Quick exercise

```go
type P struct{ X int }
var (
    a  P
    pa *P = &P{}
    ar [3]P
    s  []P = []P{{}}
    m  map[string]P = map[string]P{"k": {}}
    mp map[string]*P = map[string]*P{"k": {}}
)

a.X = 1               // (1) ✅ variable
pa.X = 1              // (2) ✅ pointer deref
ar[0].X = 1           // (3) ✅ array index (ar addressable)
s[0].X = 1            // (4) ✅ slice index
m["k"].X = 1          // (5) ❌ map value not addressable
mp["k"].X = 1         // (6) ✅ (*mp["k"]).X — pointer deref addressable
P{}.X = 1             // (7) ❌ literal without &
(&P{}).X = 1          // (8) ✅ composite literal exception
```

### Why this design

Go could make everything addressable (heap everything, like some other languages). It didn't because:

- Stack allocation is fast.
- Map/array inline layout is fast.
- Value semantics are useful.
- Immutable strings are a real guarantee.

You opt into pointer semantics explicitly. Addressability is the language's mechanism for saying "can't guarantee a stable pointer here, so I won't let you take one."

---

## Q13. Final takeaway / rules of thumb for writing Go?

Seven rules that prevent 90% of Go correctness bugs:

### 1. Follow the context trail religiously

Every I/O or blocking function takes `ctx context.Context` as first parameter named `ctx`. In Gin handlers, the source is always `c.Request.Context()` — never raw `c`.

### 2. `defer cancel()` every time

`WithCancel`, `WithTimeout`, `WithDeadline`, `WithCancelCause` all return a `CancelFunc`. The next line is always `defer cancel()`.

### 3. Pick your context source deliberately

| Use case | Context |
|---|---|
| Synchronous I/O in a handler | `c.Request.Context()` |
| Goroutine outliving the request | `context.Background()` + `WithTimeout` |
| Startup / shutdown | `context.Background()` |
| Unfinished plumbing | `context.TODO()` |

### 4. Pointer receivers for types that *do*; value receivers for types that *are*

Pointer: mutates, holds mutex, manages connection, represents identity.
Value: small, immutable, value-like (`time.Time`, `uuid.UUID`, `Point`).

Be consistent across the type — all pointer or all value.

### 5. Value-passing for primitive parameters; pointer only for real optionality

`string`, `bool`, small structs → value.
`*T` → optional (PATCH `nil` means "not provided") or when sharing/mutation required.

### 6. Addressability governs what you can mutate and point to

| Scenario | Rule |
|---|---|
| `m[k].Field = x` fails | map values aren't addressable |
| `f().Field = x` fails | return values aren't addressable |
| `var i Iface = T{}` rejects pointer method | use `*T` |
| `s[0] = 'x'` on string fails | strings immutable |
| `&T{...}` works, `T{...}.Field = x` doesn't | `&` on literals is a special exception |

### 7. Interfaces are an asymmetric method-set contract

`T`'s method set = value-receiver methods only.
`*T`'s method set = both.
Interface with any pointer-receiver method → only `*T` satisfies. In practice: store pointers in interface variables.

### The meta-rule

Across all of these, the theme is **respect lifecycle and memory layout**. Context is lifecycle. Pointer vs value is memory layout. Addressability is the compile-time check stopping you from violating either.

When confused, ask:

1. **What is the lifetime of this thing?** (When does it stop being valid?)
2. **Where does it live in memory?** (Stack, heap, map bucket, inside an interface?)
3. **Who else can see it?** (Is the pointer shared? Is the pool recycling it?)

Most Go bugs come from a mismatch between your mental model and reality. The seven rules are shortcuts for getting those three answers right.

---

## Q14. After slice reallocation, if I use the old pointer, why doesn't it crash — and what bugs does it cause?

### Why it doesn't crash — memory-level

Go's GC tracks every live pointer. When:

```go
s := []int{10, 20, 30}  // backing array A
p := &s[0]              // A is now referenced by both s and p
s = append(s, 40)       // reallocate; s points to B, A still referenced by p
```

After reallocation, the GC sees `p` references A. A stays alive, not freed. Reading `*p` returns valid bytes. Writing `*p = 99` writes to valid memory.

Different from C:

```c
int *p = &arr[0];
free(arr);
*p = 99;       // undefined behavior — use-after-free
```

In Go, that never happens. **No crash, no corruption.**

The bug is semantic: your mental model says "`p` is a pointer to `s[0]`" — but after reallocation, it isn't. It's a pointer to a ghost array that `s` no longer knows about.

### Minimal walkthrough

```go
s := make([]int, 1, 1)
s[0] = 10
p := &s[0]

fmt.Println(*p, s[0])   // 10 10 — agree

s = append(s, 20)       // reallocate. s → new array. p → old array.

*p = 999
fmt.Println(*p, s[0])   // 999 10 — disagree!

s[0] = 777
fmt.Println(*p, s[0])   // 999 777 — still disagree
```

Two variables, two universes. Neither is "wrong" in isolation; the bug is you intended them to coordinate and they don't.

### Real-world bug patterns

**Pattern A — cached pointer goes stale:**

```go
type UserService struct {
    defaultUser *User
}

func NewService(users []User) *UserService {
    return &UserService{defaultUser: &users[0]}
}

users := []User{{Name: "Alice"}}
svc := NewService(users)
users = append(users, User{Name: "Bob"})   // may reallocate
users[0].Name = "Alice Updated"

fmt.Println(svc.defaultUser.Name)          // "Alice" — stale!
```

Small test data stays in original array, hiding the bug. Production data triggers reallocation, breaks the cache.

**Pattern B — element addresses in a map while building a slice:**

```go
type Event struct { ID int; Data string }

events := make([]Event, 0)   // no preallocated cap
byID := map[int]*Event{}

for i := 0; i < 1000; i++ {
    events = append(events, Event{ID: i, Data: "..."})
    byID[i] = &events[len(events)-1]  // pointer into current array
}

events[500].Data = "patched"
fmt.Println(byID[500].Data)           // probably "..." — stale
```

Every reallocation copies elements to a new array, leaving `byID`'s pointers scattered across several orphaned arrays. Fix: store indices, or preallocate capacity, or store values in the map.

**Pattern C — concurrent read/write with append:**

```go
type Log struct{ entries []Entry }

func (l *Log) LastPtr() *Entry { return &l.entries[len(l.entries)-1] }
func (l *Log) Append(e Entry) { l.entries = append(l.entries, e) }

// Goroutine 1:
p := log.LastPtr()
go readFromP(p)

// Goroutine 2:
log.Append(newEntry)  // reallocates
```

Beyond the data race (its own problem), g1 reads through `p` from the old array while g2 writes into the new array.

**Pattern D — two slices sharing a backing array (the inverse):**

```go
s := []int{1, 2, 3, 4, 5}   // cap 5
s2 := s[:3]                 // shares backing; len=3, cap=5
s2 = append(s2, 99)         // writes at index 3 of shared array
                            // → overwrites s[3]!
fmt.Println(s)              // [1 2 3 99 5]
```

A function returning a sub-slice can have its caller append and silently mutate the source. Nasty.

**Pattern E — returning `&slice[i]` as an API:**

```go
func (c *Cache) Get(id int) *Entry {
    for i := range c.items {
        if c.items[i].ID == id {
            return &c.items[i]  // dangerous
        }
    }
    return nil
}
```

Cache grows, callers keep reading from the old array. Better: return a copy, return an index, or `map[int]*Entry` where each entry is individually heap-allocated.

### Why this hurts so much in practice

1. **Silent.** No panic, no error, race detector doesn't always catch it.
2. **Capacity-dependent.** Only manifests when `append` actually reallocates — small test data hides it.
3. **Invisible in review.** `&s[0]` looks harmless, `append(s, x)` looks harmless. The bug is in the interaction across lines that may be far apart.

### Four defensive rules

1. **Don't save `&slice[i]` across code that might `append` to the slice.** Copy the element out (`elem := s[i]`) or use indices.
2. **For stable per-element pointers, allocate each element individually**: `[]*Elem` or `map[K]*Elem`. Each heap-allocated, stable.
3. **If you return a sub-slice or an element, document whether the caller can append.** `strings.Builder`/`bytes.Buffer` handle this by returning copies on `.String()`/`.Bytes()`.
4. **Run `go test -race`**. Catches the concurrent cases, not the single-threaded ones.

### The one-line rule

> **`append` breaks the contract between your saved pointer and the slice.** Treat `&s[i]` as only valid until the next `append(s, ...)`. If you can't guarantee that, don't take the address.

---

## Meta-rule from the session

For any Go correctness question, ask:

1. **Lifetime** — when does this thing stop being valid?
2. **Memory layout** — where does it live, and does its address stay put?
3. **Visibility** — who else holds a reference, and what can they do with it?

Most bugs come from answering one of these three wrong.

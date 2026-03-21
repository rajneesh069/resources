# Code

```python
import asyncio


async def main():
    print("Brewing something....")
    await asyncio.sleep(2)
    print("Did I reach here?!")
    print("Brewed stuff")


def some_task():
    print("I am here!")
    for _ in range(100_000):
        pass


asyncio.run(main())
```

## Question: I wanted to see something like JS does, like moving onto the sync stuff while the async stuff finishes, is that behavior mimiced in python or not? If yes, how to achieve it?

# 🟦 JavaScript world:

JS is single-threaded with an event loop.

When we `await somethingAsync()`, JS pauses that async function, puts it in the event loop’s "pending tasks" queue, and immediately returns control to run other synchronous code.

Example:

```ts
async function main() {
  console.log("Brewing...");
  await new Promise((r) => setTimeout(r, 2000));
  console.log("Brewed!");
}

main();
console.log("Sync stuff running while brewing...");
```

```
Output:

Brewing...
Sync stuff running while brewing...
Brewed!
```

So JS always lets sync code after the async call run right away.

# 🟩 Python world:

Python’s asyncio is `cooperative concurrency`.

When we `await asyncio.sleep(2)`, you’re telling the event loop:

`“I’m done for now, let someone else run.”`

BUT — unlike JS, asyncio.run(main()) only runs the async code we passed in. It won’t “fall back” to your synchronous some_task() unless we explicitly tell the event loop to juggle multiple tasks.

The code:

```python
async def main():
    print("Brewing something....")
    await asyncio.sleep(2) # gives control back to loop
    print("Did I reach here?!")
    print("Brewed stuff")
```

will just pause for 2s and then continue — but nothing else is scheduled, so it looks like "nothing happens" during that time.

## ✅ How to mimic JS behavior in Python?

If we want some_task() (sync function) to run while your async sleep is happening, we need to schedule it as a task in the event loop. For example:

```python
import asyncio

async def main():
print("Brewing something....")
task1 = asyncio.create_task(brew()) # async task
task2 = asyncio.to_thread(some_task) # run sync function in thread
await asyncio.gather(task1, task2)

async def brew():
    await asyncio.sleep(2)
    print("Brewed stuff")

def some_task():
    print("I am here!")
    for _ in range(100_000_000):
        pass
    print("Done with sync loop")

asyncio.run(main())
```

```
Output (roughly):
Brewing something....
I am here!
Done with sync loop
Brewed stuff
```

🔑 Key takeaways:

In JS, sync code after an async call always runs immediately because the function unwinds back to the event loop.

In Python, `asyncio.run(main())` locks into that async coroutine. If we want concurrency, we must explicitly schedule other work (asyncio.create_task, asyncio.to_thread, etc.).

`👉 Yes, we can mimic JS’s behavior in Python, but we need to explicitly tell the event loop to run other things (unlike JS where it just “happens”).`

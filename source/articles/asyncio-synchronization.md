# Locks, semaphores, events and conditions in asyncio

**Covers:** asyncio.Lock, asyncio.Semaphore, asyncio.Event, asyncio.Condition

## The same ideas, without blocking the loop {#overview}
Asyncio tasks share one thread, and they switch only at `await`, so many race conditions cannot happen. But a task can still pause in the middle of an update, and another can run then. For those cases, asyncio has its own locks and signals. They are not interchangeable with the threading ones: the thread versions block the whole thread, and would freeze the event loop.

## asyncio.Lock {#asyncio-lock}
`asyncio.Lock` allows one task at a time into a section. A task that finds it taken awaits, and the loop runs others meanwhile. You use it with `async with lock:`. It protects data that is changed across an `await`.

## asyncio.Semaphore {#asyncio-semaphore}
`asyncio.Semaphore` allows up to N tasks into a section at once. It is the usual way to limit how many requests are in flight at the same time.

## asyncio.Event {#asyncio-event}
`asyncio.Event` is a flag that tasks can await. `await event.wait()` pauses until another task calls `event.set()`. `clear()` lowers it again.

## asyncio.Condition {#asyncio-condition}
`asyncio.Condition` combines a lock with waiting for a change. A task holds the lock, checks a state, and if it is not ready, awaits `condition.wait()`, which releases the lock until another task calls `notify()` or `notify_all()`.

## A worked example {#example}
An update that spans an `await` needs a lock:

```python
import asyncio

async def main():
    lock = asyncio.Lock()
    total = 0

    async def add():
        nonlocal total
        async with lock:
            current = total
            await asyncio.sleep(0.001)
            total = current + 1

    await asyncio.gather(*(add() for _ in range(10)))
    print(total)  # 10

asyncio.run(main())
```

Each task reads the total, pauses at the `await`, then writes. Without the lock, all ten would read 0 and write 1, and the total would be 1. With the lock, they take turns, and the total is 10. The pause inside the update is what makes it unsafe even in a single thread.

Limiting concurrency with a semaphore:

```python
import asyncio

async def main():
    sem = asyncio.Semaphore(3)
    active = 0
    peak = 0

    async def job():
        nonlocal active, peak
        async with sem:
            active += 1
            peak = max(peak, active)
            await asyncio.sleep(0.01)
            active -= 1

    await asyncio.gather(*(job() for _ in range(10)))
    print(peak)  # 3

asyncio.run(main())
```

Ten jobs, but never more than three at the same moment. For 10 jobs of 0.01 seconds with 3 at a time, you need 4 rounds, about 0.04 seconds in all.

An event as a start signal:

```python
import asyncio

async def main():
    go = asyncio.Event()
    log = []

    async def waiter(n):
        await go.wait()
        log.append(n)

    tasks = [asyncio.create_task(waiter(n)) for n in range(3)]
    await asyncio.sleep(0.01)
    print(log)  # []
    go.set()
    await asyncio.gather(*tasks)
    print(sorted(log))  # [0, 1, 2]

asyncio.run(main())
```

Nothing is logged until `set()`, then all three proceed.

A condition, waiting for a state:

```text
import asyncio

async def main():
    cond = asyncio.Condition()
    items = []

    async def consumer():
        async with cond:
            await cond.wait_for(lambda: len(items) > 0)
            return items.pop()

    async def producer():
        async with cond:
            items.append("job")
            cond.notify()

    task = asyncio.create_task(consumer())
    await asyncio.sleep(0.01)
    await producer()
    print(await task)  # job

asyncio.run(main())
```

The consumer waits until the list has an item. The producer adds one and notifies, and the consumer wakes with the lock held.

When no lock is needed: if a section has no `await` inside, no other task can run in the middle, so it is already safe. A lock is needed only across awaits.

| Class | Admits | Use it for |
|---|---|---|
| `asyncio.Lock` | One task | Updates across an `await` |
| `asyncio.Semaphore` | N tasks | Limit of simultaneous jobs |
| `asyncio.Event` | Signal | Start or stop |
| `asyncio.Condition` | Wait for a state | Producer-consumer logic |

## A common mistake {#mistake}
Using a `threading.Lock` in a coroutine. It blocks the thread, so the loop freezes. Use the asyncio version with `async with`. Another is adding a lock around code that has no `await`, which costs time and protects nothing.

## Key points {#key-points}
- Asyncio has its own `Lock`, `Semaphore`, `Event` and `Condition`, used with `await` and `async with`.
- A lock is needed only when an update spans an `await`.
- A semaphore limits how many tasks run a section together.
- Never use the threading versions inside coroutines.

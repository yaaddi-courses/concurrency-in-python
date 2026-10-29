# Asyncio queues

**Covers:** asyncio.Queue, LifoQueue, PriorityQueue, LIFO, task_done

## Handing work between tasks {#overview}
Producers make jobs and consumers handle them. A queue between them smooths the difference in speed and gives a natural place to limit the backlog. Asyncio has its own queues, whose waiting calls are awaited, so a task that waits does not stop the loop.

## asyncio.Queue {#asyncio-queue}
`asyncio.Queue` is a first-in, first-out queue for tasks. `await queue.put(item)` adds an item and waits if the queue has a maximum size and is full. `await queue.get()` removes the next item and waits if it is empty. It is not thread-safe: use it only from tasks on one loop.

## LifoQueue {#lifoqueue}
`LifoQueue` is a queue in which the newest item leaves first, like a stack of plates. It suits depth-first work, where the most recently found job should be done next.

## PriorityQueue {#priorityqueue}
`PriorityQueue` hands out the smallest item first. Items are usually tuples such as `(priority, job)`, where a lower number means more urgent. It suits work with different urgency.

## LIFO {#lifo}
LIFO means last in, first out. The most recently added item is the first to leave. It is the opposite of FIFO, which the standard queue uses.

## task_done {#task-done}
`task_done()` tells the queue that an item taken with `get()` has been fully handled. `await queue.join()` waits until every item has been marked done, so you know that all the work is finished, and not merely taken.

## A worked example {#example}
A producer, a consumer and `join()`:

```python
import asyncio

async def main():
    queue = asyncio.Queue()
    results = []

    async def consumer():
        while True:
            item = await queue.get()
            results.append(item * 2)
            queue.task_done()

    worker = asyncio.create_task(consumer())
    for n in range(4):
        await queue.put(n)
    await queue.join()
    worker.cancel()
    print(results)  # [0, 2, 4, 6]

asyncio.run(main())
```

The consumer loops for ever. `join()` returned after all four items were marked done by `task_done()`. Then the worker was cancelled, because it was waiting for more. Without `task_done()`, `join()` would never return.

Order in the three kinds:

```python
import asyncio

async def drain(queue):
    out = []
    while not queue.empty():
        out.append(queue.get_nowait())
    return out

async def main():
    fifo, lifo, prio = asyncio.Queue(), asyncio.LifoQueue(), asyncio.PriorityQueue()
    for item in (3, 1, 2):
        await fifo.put(item)
        await lifo.put(item)
        await prio.put(item)
    print(await drain(fifo))  # [3, 1, 2]
    print(await drain(lifo))  # [2, 1, 3]
    print(await drain(prio))  # [1, 2, 3]

asyncio.run(main())
```

Items went in as 3, 1, 2. The normal queue returned them in the same order. The LIFO queue returned the newest first. The priority queue returned them sorted.

A maximum size creates backpressure:

```python
import asyncio

async def main():
    queue = asyncio.Queue(maxsize=2)
    await queue.put(1)
    await queue.put(2)
    print(queue.full())  # True
    try:
        queue.put_nowait(3)
    except asyncio.QueueFull:
        print("producer must wait")  # producer must wait

asyncio.run(main())
```

With `await queue.put(3)`, the producer would pause until a consumer made room. That keeps memory bounded when the producer is faster.

Count: a producer that makes 1,000 jobs a second and a consumer that handles 100 a second. With `maxsize=50`, the producer is slowed to 100 a second, and at most 50 jobs wait. Without the limit, the backlog grows by 900 a second.

| Queue | Leaves first | Typical use |
|---|---|---|
| `Queue` | Oldest | Ordinary jobs |
| `LifoQueue` | Newest | Depth-first search |
| `PriorityQueue` | Smallest | Urgent jobs first |

A queue also helps when consumers are slow in bursts. If a consumer is stuck for a second, the producer keeps adding, and the backlog absorbs the gap, as long as memory allows. With a limit set, it absorbs only up to the limit, and after that the producer waits. Pick the limit from what you can afford to hold: 200 jobs of 10 KB each is only 2 MB.

## A common mistake {#mistake}
Forgetting `task_done()`, so `join()` waits for ever. Call it once for each `get()`, even when handling failed. Another is using a standard `queue.Queue` in a coroutine, which blocks the loop.

## Key points {#key-points}
- `asyncio.Queue` is FIFO. `LifoQueue` gives the newest first and `PriorityQueue` the smallest.
- `task_done()` after each item, then `await queue.join()` waits for all.
- A `maxsize` gives backpressure.
- Use asyncio queues only inside one event loop.

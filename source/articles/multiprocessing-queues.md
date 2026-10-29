# Queues between processes

**Covers:** multiprocessing.Queue, FIFO, get(), put(), get_nowait(), empty()

## A safe channel across process boundaries {#overview}
Processes do not share variables, so a worker cannot append to the parent's list. The simplest way to move data between them is a queue, which works like the one for threads but is built to cross process boundaries. Items are sent through a pipe underneath, and converted to bytes on the way.

## multiprocessing.Queue {#multiprocessing-queue}
`multiprocessing.Queue` is a queue that several processes can use at once. One process puts items in, another takes them out. It does its own locking. Each item must be picklable, because it is turned into bytes in one process and rebuilt in the other, so what arrives is a copy, not the same object.

## FIFO {#fifo}
FIFO stands for first in, first out. Items leave the queue in the order they arrived, like people in a line. The oldest item is always taken first.

## put() {#put}
`put()` adds an item to the queue, waiting if the queue has a maximum size and is full. Because the item is sent by a background thread, it may not be visible to `get()` the very instant after `put()` returns.

## get() {#get}
`get()` removes and returns the next item, waiting if the queue is empty. With a `timeout`, it raises `queue.Empty` if nothing arrives in time.

## get_nowait() {#get-nowait}
`get_nowait()` returns the next item at once, or raises `queue.Empty` if there is none. It never waits.

## empty() {#empty}
`empty()` returns `True` if the queue has no items right now. The answer may be stale a moment later, since another process can put something in at any time. So it is a hint, not a guarantee, and should not decide what your program does.

## A worked example {#example}
Workers send results to the parent.

```text
import multiprocessing as mp

def square(n, out):
    out.put((n, n * n))

if __name__ == "__main__":
    out = mp.Queue()
    procs = [mp.Process(target=square, args=(n, out)) for n in range(3)]
    for p in procs:
        p.start()
    results = [out.get() for _ in range(3)]
    for p in procs:
        p.join()
    print(sorted(results))  # [(0, 0), (1, 1), (2, 4)]
```

The parent calls `get()` three times, and each call waits for one result. Only then does it join. That order matters: a process that has put items in a queue will not exit until those items are taken out, so joining before getting can freeze the program.

FIFO order for one producer:

```text
import multiprocessing as mp

def producer(q):
    for i in range(3):
        q.put(i)

if __name__ == "__main__":
    q = mp.Queue()
    p = mp.Process(target=producer, args=(q,))
    p.start()
    items = [q.get() for _ in range(3)]
    p.join()
    print(items)  # [0, 1, 2]
```

The items come out in the order they went in. With several producers, items from different producers can interleave, but each producer's own items stay in order.

Without waiting:

```text
import multiprocessing as mp
import queue

if __name__ == "__main__":
    q = mp.Queue()
    try:
        q.get_nowait()
    except queue.Empty:
        print("nothing yet")  # nothing yet
```

Why not rely on `empty()`? Suppose a consumer loops `while not q.empty(): q.get()`. Between the test and the `get()`, another consumer may take the last item, and the first one waits for ever. A safer loop uses `get(timeout=...)` and handles `queue.Empty`, or a stop marker described in a later article.

Count the cost of copying. Sending a list of 1 million numbers means converting it to bytes (about 8 MB), sending it, and rebuilding it. That can take tens of milliseconds. So send small messages, or file names and ranges, and not huge objects.

| Call | Waits? | If nothing is there |
|---|---|---|
| `get()` | Yes | Waits |
| `get(timeout=1)` | Up to 1 s | Raises `queue.Empty` |
| `get_nowait()` | No | Raises `queue.Empty` |
| `empty()` | No | Returns `True`, maybe wrongly a moment later |

## A common mistake {#mistake}
Joining a process before reading everything it put in the queue, which freezes both. Read first. Another is passing an unpicklable object, such as an open file, and getting an error in the background thread that sends it.

## Key points {#key-points}
- A `multiprocessing.Queue` moves copies of picklable items between processes in FIFO order.
- `get()` waits, `get_nowait()` does not, and `empty()` can be out of date at once.
- Take items out before joining the producer.
- Send small messages.

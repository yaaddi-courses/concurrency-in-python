# Semaphores: limiting how many threads enter

**Covers:** Semaphore, threading.Semaphore, BoundedSemaphore, ValueError, OverflowError, IndexError

## A lock that lets in more than one {#overview}
A lock admits one thread. Sometimes you want a limit above one: at most five downloads at once, or at most three connections to a database. A semaphore is a counter that does exactly that.

## Semaphore {#semaphore}
A semaphore is a counter that decides how many threads may enter a section at the same time. In Python it is the class `threading.Semaphore`, described next.

## threading.Semaphore {#threading-semaphore}
A `threading.Semaphore` holds a counter, set when you create it. `acquire()` lowers the counter by one, and waits if it is zero. `release()` raises it by one and wakes a waiter. At most that many threads can be inside the protected block at the same time. With a counter of one, it behaves like a lock.

## BoundedSemaphore {#boundedsemaphore}
A `BoundedSemaphore` works the same way but refuses to go above its starting counter. If you release more often than you acquired, it raises a `ValueError`. That turns a silent counting mistake into an error you can see.

## ValueError {#valueerror}
`ValueError` is raised when an argument has the right type but is not acceptable. A `BoundedSemaphore` raises it when released too many times. Creating a semaphore with a negative counter raises it too.

## OverflowError {#overflowerror}
`OverflowError` is raised when a number is too large for what it is used for. For example, a counter above the system's limits cannot be represented. You rarely see it with semaphores, but it can appear when an enormous number is passed as the count.

## IndexError {#indexerror}
`IndexError` is raised when you use a position that does not exist in a list, such as the fifth item of a list that has three. It appears in thread code when several workers take items from a shared list, and one empties it between another's check and its removal.

## A worked example {#example}
At most two workers inside the block at once, out of five.

```python
import threading
import time

sem = threading.Semaphore(2)
lock = threading.Lock()
inside = 0
peak = 0

def worker():
    global inside, peak
    with sem:
        with lock:
            inside += 1
            peak = max(peak, inside)
        time.sleep(0.05)
        with lock:
            inside -= 1

threads = [threading.Thread(target=worker) for _ in range(5)]
for t in threads:
    t.start()
for t in threads:
    t.join()
print(peak)  # 2
```

Five threads wanted in, but the highest number inside together was 2. The other three waited their turn. With a plain lock, the peak would be 1, and with no limit, 5.

Count the time: five jobs of 0.05 seconds each, two at a time, take three rounds: 2 + 2 + 1, so about 0.15 seconds. One at a time takes 0.25 seconds, and all at once 0.05 seconds. The semaphore picks a point in between, which protects whatever resource has a limit.

A bounded one catches a bug:

```python
import threading

sem = threading.BoundedSemaphore(1)
sem.acquire()
sem.release()
try:
    sem.release()
except ValueError:
    print("released too often")  # released too often
```

A plain `Semaphore` would have silently raised its counter to 2, and from then on admitted two threads where one was meant.

The two other errors show up around shared lists. `IndexError`:

```python
items = [1, 2, 3]
try:
    items[5]
except IndexError:
    print("no such position")  # no such position
```

and `OverflowError`:

```python
try:
    float(10 ** 400)
except OverflowError:
    print("too large")  # too large
```

| Class | Can the counter exceed its start? | Extra release |
|---|---|---|
| `Semaphore` | Yes | Counts up silently |
| `BoundedSemaphore` | No | Raises `ValueError` |

## A common mistake {#mistake}
Using a plain `Semaphore` and releasing in an error path that was not preceded by an acquire. Use `with`, which pairs them, or a bounded one to catch the mistake. Another is a semaphore of 100 that protects a resource that only handles 10, so the limit protects nothing.

## Key points {#key-points}
- A semaphore admits up to N threads at a time.
- `BoundedSemaphore` raises `ValueError` if released too often.
- Use `with sem:` so each acquire is paired with a release.
- `IndexError` and `OverflowError` are errors from the list and number operations around threaded code.

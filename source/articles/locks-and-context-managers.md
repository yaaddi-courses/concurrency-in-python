# Locks: acquire, release and the with statement

**Covers:** Lock, threading.Lock, acquire(), release(), __enter__, __exit__, RLock, threading.RLock

## Taking turns on shared data {#overview}
A lock is the tool that fixes a race condition. It lets one thread at a time into a critical section, and makes the others wait at the door. This article covers the basic lock, and the way to use it safely.

## Lock {#lock}
A lock is an object that only one thread can hold at a time, used to protect shared data. In Python it is the class `threading.Lock`, described next.

## threading.Lock {#threading-lock}
A `threading.Lock` is a mutex you create with `threading.Lock()`. Only one thread can hold it at a time. Any other thread that tries to take it waits until the holder gives it back. Wrapping the shared update in a lock removes the race.

## acquire() {#acquire}
`acquire()` takes the lock, waiting if another thread already holds it. It can take a `timeout`, after which it gives up and returns `False`, or `blocking=False` to try without waiting at all.

## release() {#release}
`release()` gives the lock back, so another thread can take it. Releasing a lock that is not held raises a `RuntimeError`. Forgetting to release is the classic mistake, since every other thread then waits for ever.

## __enter__ {#enter}
`__enter__` is the method that Python calls at the start of a `with` block. For a lock, it calls `acquire()`.

## __exit__ {#exit}
`__exit__` is the method that Python calls when the `with` block ends, whether normally or by an error. For a lock, it calls `release()`. That guarantee is why `with lock:` is the recommended form.

## threading.RLock {#threading-rlock}
`threading.RLock` is the full name of the reentrant lock below. Use it in place of `threading.Lock` whenever the same thread may need to take the lock more than once.

## RLock {#rlock}
An `RLock`, a reentrant lock, can be taken again by the thread that already holds it. It counts how many times it was taken, and is free only after the same number of releases. A plain lock would make such a thread wait for itself for ever. It is needed when one locked function calls another that takes the same lock.

## A worked example {#example}
The counter from the previous article, now safe:

```python
import threading

counter = 0
lock = threading.Lock()

def add():
    global counter
    for _ in range(50_000):
        with lock:
            counter += 1

threads = [threading.Thread(target=add) for _ in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()
print(counter)  # 200000
```

The result is always exactly 200,000. The `with lock:` block calls `__enter__`, which takes the lock, and `__exit__`, which releases it even if an error happens inside. The cost: the four threads now take turns at the counter, so the program is slower than the broken one.

The same thing, written by hand:

```python
import threading

lock = threading.Lock()
lock.acquire()
try:
    print("inside")  # inside
finally:
    lock.release()
print(lock.locked())  # False
```

The `try` and `finally` do what `with` does for you. Use `with`.

A timeout on `acquire()`:

```python
import threading

lock = threading.Lock()
lock.acquire()
print(lock.acquire(timeout=0.1))  # False
lock.release()
```

The lock was already held, so the second `acquire` waited 0.1 seconds and gave up with `False`.

A lock held twice by the same thread needs an `RLock`:

```python
import threading

rlock = threading.RLock()

def outer():
    with rlock:
        return inner()

def inner():
    with rlock:
        return "ok"

print(outer())  # ok
```

With a plain `Lock`, `inner` would wait for a lock that its own thread holds, and the program would stop for ever.

| Tool | Use it for |
|---|---|
| `Lock` | One thread at a time |
| `RLock` | Same thread may take it again |

## A common mistake {#mistake}
Taking a lock with `acquire()` and forgetting to release it when an error happens. Use `with`. Another is holding a lock while doing something slow, such as a network call, which blocks every other thread.

## Key points {#key-points}
- Protect the whole critical section with one lock, using `with lock:`.
- `with` calls `__enter__` and `__exit__`, so the lock is always released.
- Use an `RLock` when the same thread may take the lock twice.

# Thread, lock and timer method names that only look real

**Covers:** .begin(), .execute(), .status(), .unlock(), .off(), .decrement(), .take(), .abort()

## Why wrong answers look so reasonable {#overview}
Every concurrency tool in Python has a small, fixed set of method names. When you are unsure of one, your mind suggests a name from plain English or from another language: "begin" for starting, "unlock" for a lock, "abort" for stopping. These guesses sound right, and that is exactly why they appear as wrong answers. Each name below does not exist on the object in question. Next to each one you will find the real name, so you can tell the two apart at a glance.

## .begin() {#begin}
`.begin()` is not a method of a thread. A thread is started with `start()`, which makes the new thread call its `run()` method. You do not call `run()` yourself to start a thread: that would run the work in your own thread and nothing would happen in parallel.

## .execute() {#execute}
`.execute()` is not a thread method either. The name is common in database libraries, where a cursor executes a query. For a thread, the word for "go" is `start()`.

## .status() {#status}
`.status()` does not exist on a thread. To ask whether a thread is still working without waiting for it, use `is_alive()`. It returns `True` while the thread runs and `False` once it has finished. By contrast, `join()` waits until the thread is done, so it blocks.

## .unlock() {#unlock}
`.unlock()` is not a lock method. A lock is taken with `acquire()` and given back with `release()`. In most code you avoid both and use a `with lock:` block, which releases the lock for you even when an error occurs. The pair "lock" and "unlock" belongs to other languages and to everyday speech.

## .off() {#off}
`.off()` does not exist on an event. An event holds a flag. `set()` raises it, `clear()` lowers it back to false, `is_set()` reads it, and `wait()` pauses until it is raised. There is no on or off, only set and clear.

## .decrement() {#decrement}
`.decrement()` is not a method of a semaphore. A semaphore keeps a counter. Taking a place lowers the counter, and the real method for that is `acquire()`. If the counter is already at zero, `acquire()` waits until another thread calls `release()`, which raises the counter again.

## .take() {#take}
`.take()` is another invented name for the same job. Nothing is "taken" from a semaphore by this name. Use `acquire()`, preferably in a `with sem:` block.

## .abort() {#abort}
`.abort()` does not exist on a timer. A timer waits for a delay and then runs a function. If you change your mind before it fires, call `cancel()`. After it has fired, cancelling has no effect.

## A worked example {#example}
Python can tell you which names exist. The built-in `hasattr` returns `True` for a real name and `False` for an invented one.

```python
import threading

t = threading.Thread(target=print)
print(hasattr(t, "start"))  # True
print(hasattr(t, "begin"))  # False
print(hasattr(t, "execute"))  # False
print(hasattr(t, "is_alive"))  # True
print(hasattr(t, "status"))  # False
```

Only the real names are found. The same test works for the other objects.

```python
import threading

lock = threading.Lock()
event = threading.Event()
sem = threading.Semaphore(2)
timer = threading.Timer(60, print)
print(hasattr(lock, "release"), hasattr(lock, "unlock"))  # True False
print(hasattr(event, "clear"), hasattr(event, "off"))  # True False
print(hasattr(sem, "acquire"), hasattr(sem, "decrement"), hasattr(sem, "take"))  # True False False
print(hasattr(timer, "cancel"), hasattr(timer, "abort"))  # True False
```

Now see the real methods working together. The event flag goes up and down, and the semaphore counter changes with each `acquire` and `release`.

```python
import threading

event = threading.Event()
event.set()
print(event.is_set())  # True
event.clear()
print(event.is_set())  # False

sem = threading.Semaphore(2)
print(sem.acquire(blocking=False))  # True
print(sem.acquire(blocking=False))  # True
print(sem.acquire(blocking=False))  # False
sem.release()
print(sem.acquire(blocking=False))  # True
```

A semaphore started at 2 allows two takers. The third attempt fails at once, because we asked it not to block. After one `release()` there is a place again.

A thread that is polled instead of joined:

```python
import threading
import time

t = threading.Thread(target=time.sleep, args=(0.2,))
t.start()
print(t.is_alive())  # True
t.join()
print(t.is_alive())  # False
```

The thread was alive while it slept and not alive after `join()` returned.

## A common mistake {#mistake}
The usual slip is to trust a name that feels natural. Calling `t.begin()` raises an `AttributeError`, and it happens when that line runs, which may be deep in a long program. Another slip is to call `t.run()` because it exists. It exists, but it runs the work in the current thread, so you get no concurrency at all. When unsure, look the name up in the documentation or test it with `hasattr`.

## Key points {#key-points}
- A thread starts with `start()` and is polled with `is_alive()`. It is waited for with `join()`.
- A lock uses `acquire()` and `release()`. A `with` block does both safely.
- An event is raised with `set()` and lowered with `clear()`.
- A semaphore uses `acquire()` and `release()`, and a timer is stopped with `cancel()`.
- Names like `begin`, `execute`, `status`, `unlock`, `off`, `decrement`, `take` and `abort` do not exist on these objects.

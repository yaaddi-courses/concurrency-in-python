# Events: a flag that threads wait for

**Covers:** Event, threading.Event, set(), clear(), is_set(), wait()

## Telling other threads that something happened {#overview}
A lock decides who may touch data. An event answers a different question: has the thing I need happened yet? One thread announces it, and any number of others wait for the announcement. It is the simplest way to say "start now", "stop now" or "the data is ready".

## Event {#event}
An event is a simple object that lets one thread signal others that something happened. It holds a flag. In Python it is the class `threading.Event`, described next.

## threading.Event {#threading-event}
A `threading.Event` is a simple flag that one thread sets and other threads wait for. It starts out cleared. Threads that call `wait()` sleep until the flag is set. Many threads can wait on the same event, and one `set()` wakes all of them.

## set() {#set}
`set()` raises the flag, which is true from then on, and wakes every thread that is waiting for it.

## clear() {#clear}
`clear()` lowers the flag again, so that the next `wait()` blocks. It makes an event reusable, for a signal that happens many times.

## is_set() {#is-set}
`is_set()` returns `True` if the flag is currently raised. It never waits. Use it to check, for example in a loop, whether you have been asked to stop.

## wait() {#wait}
`wait()` blocks until the event is set. With a `timeout` in seconds, it gives up after that time and returns `False`. It returns `True` when the flag was set.

## A worked example {#example}
A start signal: three workers are ready, and wait for a go.

```python
import threading

go = threading.Event()
log = []

def worker(n):
    go.wait()
    log.append(n)

threads = [threading.Thread(target=worker, args=(n,)) for n in range(3)]
for t in threads:
    t.start()
print(go.is_set())  # False
go.set()
for t in threads:
    t.join()
print(sorted(log))  # [0, 1, 2]
print(go.is_set())  # True
```

All three workers sleep on `go.wait()`. Nothing is logged until `go.set()`, and then all three wake at once. One signal released three waiting threads.

A stop signal, the most common use. The worker checks the flag in its loop:

```python
import threading
import time

stop = threading.Event()
ticks = []

def worker():
    while not stop.is_set():
        ticks.append(1)
        time.sleep(0.01)

t = threading.Thread(target=worker)
t.start()
time.sleep(0.1)
stop.set()
t.join()
print(len(ticks) > 0)  # True
```

The worker runs until told to stop, and then ends cleanly. This is far safer than trying to kill a thread, which Python does not allow.

A better version waits on the event instead of sleeping, so it reacts at once:

```python
import threading

stop = threading.Event()
stop.set()
print(stop.wait(timeout=1))  # True
stop.clear()
print(stop.wait(timeout=0.1))  # False
```

With the flag set, `wait` returns `True` immediately. After `clear()`, it waits the full 0.1 seconds and returns `False` on timeout.

Count the difference for a worker that must react to a stop within 0.01 seconds. Polling with `sleep(0.01)` wakes the thread 100 times a second even when nothing happens. Waiting on the event wakes it zero times until the event is set, and then immediately.

| Method | Does | Waits? |
|---|---|---|
| `set()` | Raises the flag, wakes waiters | No |
| `clear()` | Lowers the flag | No |
| `is_set()` | Reads the flag | No |
| `wait(timeout)` | Blocks until the flag is set | Yes |

## A common mistake {#mistake}
Using `wait()` without a timeout in a thread that must be able to stop. If nobody ever sets the event, the thread waits for ever. Give it a timeout and check again. Another is expecting `set()` to wake only one waiter. It wakes all of them.

## Key points {#key-points}
- An event is a flag. `set()` raises it, `clear()` lowers it, `is_set()` reads it.
- `wait()` sleeps until the flag is set, or until the timeout.
- Use an event to signal start or stop to many threads.
- Always consider a timeout on `wait()`.

# Condition variables and the producer-consumer pattern

**Covers:** threading.Condition, notify(), notify_all(), notifyAll(), spurious wakeup, producer-consumer pattern

## Waiting for something more specific than a flag {#overview}
An event says that something happened. Often, a thread needs to wait for a particular state of shared data: "until there is at least one item in the list". A condition variable combines a lock with a waiting room for exactly that.

## threading.Condition {#threading-condition}
A `threading.Condition` ties a lock to the ability to wait and be woken. A thread takes the lock with `with cond:`, checks the state, and if it is not ready, calls `wait()`. That releases the lock and sleeps. Another thread changes the state, and calls `notify()`. The sleeper wakes and takes the lock again before it continues.

## notify() {#notify}
`notify()` wakes one thread that is waiting on the condition. Which one is not specified. It must be called while holding the lock.

## notify_all() {#notify-all}
`notify_all()` wakes every thread that is waiting on the condition. Use it when a change might help several waiters, or when you are unsure which one can proceed.

## notifyAll() {#notifyall}
`notifyAll()` is an older spelling of `notify_all()` that still works, so you will see it in old code. Write the underscore form in anything new.

## Spurious wakeup {#spurious-wakeup}
A spurious wakeup is a wake-up of a waiting thread when no one has notified it, or when the state it waits for is still not true. It is a known property of this kind of waiting on all systems. The cure is to always check the state again after waking, in a `while` loop, and not in a single `if`.

## Producer-consumer pattern {#producer-consumer-pattern}
The producer-consumer pattern splits work between threads that make items, the producers, and threads that use them, the consumers, linked by a shared buffer. The buffer smooths out differences in speed. A condition, or a queue, is how the consumer waits when the buffer is empty.

## A worked example {#example}
A buffer shared by a producer and a consumer.

```python
import threading

items = []
cond = threading.Condition()
got = []

def consumer():
    with cond:
        while not items:
            cond.wait()
        got.append(items.pop(0))

def producer():
    with cond:
        items.append("job")
        cond.notify()

c = threading.Thread(target=consumer)
c.start()
p = threading.Thread(target=producer)
p.start()
c.join()
p.join()
print(got)  # ['job']
```

The consumer takes the lock, finds the list empty and waits, which releases the lock. The producer takes the lock, adds a job and calls `notify()`. The consumer wakes, takes the lock again and finds the job.

The `while not items` is the important part. If the consumer used `if not items`, a spurious wakeup, or another consumer that took the item first, would let it continue with an empty list, and `pop(0)` would fail. The `while` checks again after every wake-up.

Several consumers, one batch of work. Here `notify_all()` is the safer choice:

```python
import threading

state = {"ready": False}
cond = threading.Condition()
seen = []

def waiter(n):
    with cond:
        while not state["ready"]:
            cond.wait()
        seen.append(n)

threads = [threading.Thread(target=waiter, args=(n,)) for n in range(3)]
for t in threads:
    t.start()
with cond:
    state["ready"] = True
    cond.notify_all()
for t in threads:
    t.join()
print(sorted(seen))  # [0, 1, 2]
```

With `notify()`, only one of the three would wake, and the other two would wait for ever. `notify_all()` wakes all, and each checks the state.

Count the costs: a consumer that polls the list every 10 ms wakes 100 times a second to find nothing. A consumer waiting on a condition wakes only when told.

| Tool | Wakes | Use it when |
|---|---|---|
| `notify()` | One waiter | Any one waiter can use the change |
| `notify_all()` | All waiters | Several may proceed, or you are unsure |

For a plain buffer, a `Queue` already does all of this, so it is the first choice. A condition is for cases that a queue cannot express, such as waiting for a state that is more than "there is an item".

## A common mistake {#mistake}
Using `if` instead of `while` around `wait()`. It works in simple tests and fails under load. Another is calling `notify()` without holding the lock, which raises an error.

## Key points {#key-points}
- A condition is a lock plus waiting. `wait()` releases the lock, and wakes holding it again.
- Always re-check the state in a `while` loop, because of spurious wakeups.
- `notify()` wakes one waiter and `notify_all()` wakes all.
- A queue is the simpler tool for plain producer-consumer work.

# Timers, thread-local data and barriers

**Covers:** threading.Timer, threading.local, Barrier, fan-out\, fan-in

## Three small helpers from the threading module {#overview}
Besides locks, the `threading` module has a few helpers for situations that come up again and again: running something later, keeping data private to one thread, and making threads meet at a fixed point.

## threading.Timer {#threading-timer}
`threading.Timer` runs a function once, after a delay, in its own thread. You can call `cancel()` before it fires. It is a handy way to schedule a delayed action without sleeping in the main flow.

## threading.local {#threading-local}
`threading.local` creates an object whose attributes are separate for every thread. Each thread sees only its own copy, so no lock is needed. It suits per-thread data such as a database connection.

## Barrier {#barrier}
A `Barrier` makes a fixed number of threads wait for each other. Each one calls `wait()`, and none continues until all of them have arrived. It suits steps that must start together.

## Fan-out, fan-in {#fan-out-fan-in}
Fan-out, fan-in is a pattern: start a whole batch of threads or tasks at once (fan out), then wait for all of them and gather their results (fan in). A barrier is the same idea inside the threads. The pattern gives the best use of waiting time, since all the jobs overlap. Starting each thread and joining it at once would remove the gain.

## A worked example {#example}
A timer and thread-local data.

```python
import threading

fired = []
timer = threading.Timer(0.1, lambda: fired.append("fired"))
timer.start()
timer.join()
print(fired)  # ['fired']

local = threading.local()
local.name = "main"

def show():
    print(hasattr(local, "name"))  # False

t = threading.Thread(target=show)
t.start()
t.join()
```

The thread does not see the main thread's `name`, so `hasattr` is `False`.

A barrier for three threads:

```python
import threading

barrier = threading.Barrier(3)
order = []

def step(n):
    order.append(f"before {n}")
    barrier.wait()
    order.append(f"after {n}")

threads = [threading.Thread(target=step, args=(n,)) for n in range(3)]
for t in threads:
    t.start()
for t in threads:
    t.join()
print(all(x.startswith("before") for x in order[:3]))  # True
```

All three "before" lines come first, since nobody passes the barrier until all three have arrived.

A timer can also be cancelled before it fires, which is how you build a timeout that can be called off:

```python
import threading

fired = []
timer = threading.Timer(0.5, lambda: fired.append("late"))
timer.start()
timer.cancel()
timer.join()
print(fired)  # []
```

Cancelled in time, the function never runs. Count what the barrier costs: if three workers take 1, 2 and 3 seconds to reach it, all three wait for the slowest, so each pass through the barrier costs 3 seconds, not 1 or 2. That is the price of keeping them in step. A common use is a simulation where every worker must finish step one before any starts step two.

Thread-local data is the right tool when each thread needs its own database connection or similar context. Passing it through every function call works too, but it clutters the code. With `threading.local`, each thread reads `local.connection` and always gets its own, with no lock and no chance of two threads sharing one by mistake.

| Helper | Use it for |
|---|---|
| `Timer` | Run once after a delay |
| `local` | Separate data per thread |
| `Barrier` | Make threads wait for each other |

A small example of why each exists. Suppose you build a download tool. A `Timer` shows a warning if a transfer takes longer than 30 seconds, and you cancel it when the transfer ends. Each download thread keeps its own connection in a `threading.local`, so two downloads never share one by accident. And a `Barrier` makes the 4 threads start their timing together, so the comparison between them is fair. None of these needs a lock of your own.

## A common mistake {#mistake}
Using a module-level variable for per-thread data and getting mixed-up results. Use `threading.local`. Another is creating a `Barrier` for three threads and starting only two: the two wait for ever.

## Key points {#key-points}
- `Timer` delays one call and can be cancelled before it fires.
- `threading.local` gives each thread its own attributes.
- A `Barrier` releases its threads only when all have arrived.

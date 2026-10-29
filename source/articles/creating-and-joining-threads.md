# Creating threads and waiting for them

**Covers:** Thread, Threading, threading.Thread, start(), join(), is_alive(), False, True

## A thread is a path of execution {#overview}
A program normally follows one path through its code. A thread is an extra path inside the same program, sharing its memory. Python's `threading` module creates and manages them. The pattern is always the same: make a thread, start it, and wait for it to finish.

## Thread {#thread}
A thread is a separate stream of execution that shares one process's memory with the other threads. All threads of a program can read and change the same variables, which is both the strength and the danger. `Thread`, with a capital T, is also the name of the class that represents one.

## Threading {#threading}
Threading means running several threads inside one process, so that tasks overlap, especially while they wait on input or output. It is the oldest and simplest way to get concurrency in Python.

## threading.Thread {#threading-thread}
`threading.Thread` is the class from the standard library for creating a new thread. You give it a `target`, the function to run, and optional `args` and `kwargs` for that function. Creating the object does not start anything yet.

## start() {#start}
`start()` begins running the thread. After it returns, the new thread executes the target function alongside your main code. Calling `start()` twice on the same thread object raises a `RuntimeError`.

## join() {#join}
`join()` makes the caller wait until the thread has finished. Without it, the main code may continue, and even end, while the worker is still busy. You can pass a timeout, in seconds, to wait at most that long.

## is_alive() {#is-alive}
`is_alive()` returns `True` while the thread is running, and `False` once it has finished. It is how you check, without waiting, whether a thread is done.

## True and False {#booleans}
`True` and `False` are Python's two boolean values, written with capital letters. Many thread functions return them, such as `is_alive()`.

## A worked example {#example}
```python
import threading
import time

def worker(name, delay):
    time.sleep(delay)
    print(f"{name} done")

t = threading.Thread(target=worker, args=("A", 0.2))
print(t.is_alive())  # False
t.start()
print(t.is_alive())  # True
t.join()
print(t.is_alive())  # False
# A done
```

Before `start()`, the thread exists but is not running, so `is_alive()` is `False`. After `start()`, it runs the 0.2-second wait, so it is `True`. After `join()` returns, the work is over, so it is `False` again. The line "A done" is printed in between, by the worker.

What if you forget `join()`? The main code reaches its end at once, but Python still waits for every running thread before it exits. The difference shows when you need the result: without `join()`, you may read it too early.

Starting several threads and waiting for all of them:

```python
import threading

results = []

def square(n):
    results.append(n * n)

threads = [threading.Thread(target=square, args=(n,)) for n in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()
print(sorted(results))  # [0, 1, 4, 9]
```

All four are started first, then all four are joined. Joining each right after starting it would run them one at a time, and lose the point of using threads. The `sorted` call is there because threads may finish in any order.

Count the saving with waits: 4 workers that each wait 1 second take 4 seconds in a row, but about 1 second when started together, since the waits overlap.

Reusing a thread object fails:

```python
import threading

t = threading.Thread(target=lambda: None)
t.start()
t.join()
try:
    t.start()
except RuntimeError:
    print("a thread can start only once")  # a thread can start only once
```

Make a new `Thread` for each run.

| Call | Does | Returns |
|---|---|---|
| `Thread(target=f)` | Creates, does not run | The thread object |
| `start()` | Begins running `f` | `None` |
| `join()` | Waits for the end | `None` |
| `is_alive()` | Checks if running | `True` or `False` |

## A common mistake {#mistake}
Calling the function instead of passing it: `Thread(target=worker("A", 1))` runs `worker` right away in the calling code and passes its result, `None`, as the target. Write `target=worker, args=("A", 1)`. Another is joining inside the same loop that starts each thread, which makes the threads run in turn.

## Key points {#key-points}
- Create a `threading.Thread` with a target, then `start()` it, then `join()` it.
- Start all threads before joining any of them.
- `is_alive()` is `True` only while the thread runs.
- A thread object can be started only once.

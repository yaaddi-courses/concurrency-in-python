# Naming threads, passing results and handling errors

**Covers:** IDs, threading.current_thread(), threading.enumerate(), threading.active_count(), threading.excepthook, Queue, concurrent.futures, print()

## Threads need names and a way to talk {#overview}
Once a program has a dozen threads, two questions come up all the time: which thread printed that? and how does a worker give its answer back? Python has small tools for both. A thread cannot return a result the way a function does, so results travel through a shared list or a queue.

## IDs {#ids}
IDs are identifiers: unique names or numbers that pick out one thing. Every thread has a name, which you can set, and a number, available as `ident`. They are what you put in a log line to see which thread did what. A good name such as `fetch-3` saves a lot of time when debugging.

## threading.current_thread() {#current-thread}
`threading.current_thread()` returns the `Thread` object of the thread that is running the code. Its `name` attribute gives the name. Called in the main script, it returns the main thread, named `MainThread`.

## threading.enumerate() {#enumerate}
`threading.enumerate()` gives you the list of all running threads, the main one included. Treat it as a photograph: a thread may start or end a moment after you look.

## threading.active_count() {#active-count}
`threading.active_count()` returns how many threads are alive right now. It equals the length of the list from `enumerate()`. A number that keeps growing is a sign that threads are started and never finish.

## threading.excepthook {#excepthook}
`threading.excepthook` is a function that Python calls when an exception escapes a thread's target. By default it prints the traceback, and the thread ends, while the rest of the program carries on. You can replace it to log the error or to alert someone. An uncaught error in a worker never reaches the main thread by itself.

## Queue {#queue}
A `Queue`, from the `queue` module, is a thread-safe first-in, first-out container for passing items between threads. One thread calls `put()` and another calls `get()`. It does its own locking, so you do not need to add any.

## concurrent.futures {#concurrent-futures}
`concurrent.futures` is the standard-library module with ready-made pools of threads and processes. It handles starting workers, handing out work and collecting results, so you write less code than with raw threads. Later articles cover it in detail.

## print() {#print}
`print()` writes text to the screen. Called from several threads at once, the text of two calls can mix on one line, since each call may write in pieces. For tidy output, build the full line first, or send lines through one thread.

## A worked example {#example}
Names and counts:

```python
import threading
import time

def work():
    time.sleep(0.2)

t = threading.Thread(target=work, name="fetch-1")
t.start()
print(threading.current_thread().name)             # MainThread
print(threading.active_count())                    # 2
print(sorted(x.name for x in threading.enumerate()))  # ['MainThread', 'fetch-1']
t.join()
print(threading.active_count())                    # 1
```

While `fetch-1` sleeps, two threads are alive. After the join, only the main thread remains.

Giving results back through a queue:

```python
import queue
import threading

q = queue.Queue()

def worker(n):
    q.put((n, n * n))

threads = [threading.Thread(target=worker, args=(n,)) for n in range(3)]
for t in threads:
    t.start()
for t in threads:
    t.join()
results = [q.get() for _ in range(3)]
print(sorted(results))  # [(0, 0), (1, 1), (2, 4)]
```

Each worker puts a pair in the queue. The main thread takes three items out. The queue needs no lock of our own.

Catching an error with `excepthook`:

```python
import threading

errors = []

def hook(args):
    errors.append(type(args.exc_value).__name__)

threading.excepthook = hook

def bad():
    raise ZeroDivisionError("boom")

t = threading.Thread(target=bad)
t.start()
t.join()
print(errors)  # ['ZeroDivisionError']
```

Without the hook, Python would print a traceback to the screen and the program would go on as if nothing was wrong. With it, the error is recorded, and the main code can check `errors` after the join.

Count the benefit of names: with 20 workers, a log line `fetch-12 failed` tells you at once where to look. A line `Thread-19 (worker) failed` still helps, but generic names give no hint about the job.

| Tool | Gives you |
|---|---|
| `current_thread()` | The running thread, and its name |
| `enumerate()` | A list of live threads |
| `active_count()` | How many are alive |
| `Queue` | A safe channel for results |
| `excepthook` | A place to catch errors from workers |

## A common mistake {#mistake}
Expecting an error in a worker to stop the program, or to show up in the main thread. It does not. Install an `excepthook`, or catch the error inside the worker and put it in a queue. Another is reading the thread list and acting on it as if it could not change.

## Key points {#key-points}
- Name your threads. `current_thread().name` shows the name.
- `enumerate()` and `active_count()` give a snapshot of live threads.
- A `Queue` carries results between threads safely.
- Errors in a worker are caught with `threading.excepthook`, or inside the worker.

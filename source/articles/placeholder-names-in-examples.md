# Placeholder names in examples

**Covers:** critical_work(), state_changed(), process(), my_func, counter.get_lock(), save_critical_data(), sync_entry(), async_work()

## Names the author made up {#overview}
Some names in the course are not part of Python at all. They are stand-ins that the author picked so an example could be read: a function that does "the important work", a coroutine that does "the asynchronous work". Seeing them in a question does not mean you must remember them. What matters is the role they play in the code around them. One name in this group, `counter.get_lock()`, is real, and it is explained last.

## critical_work() {#critical-work}
`critical_work()` stands for any code that must not run in two places at once. In `with lock: critical_work()` the lock guarantees that only one thread is inside the call at a time. You would replace it with your own function, for example the one that updates a shared total.

## state_changed() {#state-changed}
`state_changed()` stands for the code that changes the data a condition protects. A condition is used in a block that holds its lock. The code inside changes the state, and then calls `notify_all()` to wake the waiting threads so that each can check the new state.

## process() {#process}
`process()` is not a function you call to create a process. A process is an object. You create a `multiprocessing.Process` with a target function and then call its `start()` method. A name like `process()` would suggest an ordinary function call, which only runs code in the current process.

## my_func {#my-func}
`my_func` stands for the function that a thread or a process runs. It is given as `target=my_func`, with no brackets. Writing `target=my_func()` would call the function at once and pass its result, which is a common slip.

## save_critical_data() {#save-critical-data}
`save_critical_data()` stands for a coroutine whose work must finish, even if the code waiting for it is cancelled. Wrapping the call in `asyncio.shield(...)` protects the inner work from the cancellation of the outer wait.

## sync_entry() {#sync-entry}
`sync_entry()` stands for an ordinary, non-async function that has to start asynchronous code. Such a function cannot use `await`. It starts the event loop with `asyncio.run(...)`, which runs the coroutine to completion and returns its result.

## async_work() {#async-work}
`async_work()` stands for a coroutine function, one defined with `async def`. Calling it does not run it. It gives a coroutine object, and the code runs only when something awaits it or `asyncio.run` runs it.

## counter.get_lock() {#get-lock}
`counter.get_lock()` is real. A shared value made with `multiprocessing.Value` carries a lock, and `get_lock()` returns it. The pattern is `with counter.get_lock(): counter.value += 1`. The lock matters because an increment is a read, a change and a write, and two processes could mix those steps.

## A worked example {#example}
A stand-in function used as a target:

```python
import threading

def my_func():
    print("running")  # running

t = threading.Thread(target=my_func)
t.start()
t.join()
```

This prints `running`. The brackets are missing on purpose: `my_func` is passed itself, and the thread calls it later.

A lock protecting stand-in work:

```python
import threading

lock = threading.Lock()
total = 0

def critical_work():
    global total
    total += 1

def job():
    for _ in range(1000):
        with lock:
            critical_work()

threads = [threading.Thread(target=job) for _ in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()
print(total)  # 4000
```

Each of four threads adds 1000 times, and the lock keeps the total exact at 4000.

A condition changing and announcing state:

```python
import threading

condition = threading.Condition()
ready = []

def waiter():
    with condition:
        condition.wait_for(lambda: ready)
        print("saw", ready[0])  # saw go

t = threading.Thread(target=waiter)
t.start()
with condition:
    ready.append("go")
    condition.notify_all()
t.join()
```

The waiting thread slept until the state changed, and woke when `notify_all()` was called.

A synchronous entry point and an async worker:

```python
import asyncio

async def async_work():
    await asyncio.sleep(0.01)
    return 42

def sync_entry():
    return asyncio.run(async_work())

print(sync_entry())  # 42
```

`sync_entry()` is an ordinary function, and `asyncio.run` runs the coroutine and returns 42. A shielded save follows the same shape:

```text
import asyncio

async def save_critical_data(log):
    await asyncio.sleep(0.05)
    log.append("saved")

async def main():
    log = []
    task = asyncio.create_task(save_critical_data(log))
    try:
        await asyncio.wait_for(asyncio.shield(task), timeout=0.01)
    except TimeoutError:
        log.append("gave up waiting")
    await task
    print(log)  # ['gave up waiting', 'saved']

asyncio.run(main())
```

The wait ended early, and the shielded save still completed.

Finally, a shared counter with its own lock:

```python
import multiprocessing

counter = multiprocessing.Value("i", 0)
with counter.get_lock():
    counter.value += 1
print(counter.value)  # 1
```

## A common mistake {#mistake}
Trying to memorise placeholder names, or typing them as if they existed. If you copy `critical_work()` into your own program, Python raises a `NameError`, because only the author defined it. The other slip is to confuse the target with its call: `target=my_func()` runs the function in the current thread and hands its result to the new one.

## Key points {#key-points}
- Placeholder names stand for your own code. Read their role, not their spelling.
- A thread or process target is passed without brackets.
- A process is created as an object and started with `start()`.
- `asyncio.run()` is how ordinary code starts a coroutine, and `asyncio.shield()` protects a job from cancellation.
- `get_lock()` is real: it returns the lock of a shared `Value`.

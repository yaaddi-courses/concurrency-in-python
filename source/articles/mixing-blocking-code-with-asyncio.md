# Blocking code, threads and subprocesses in asyncio

**Covers:** run_in_executor, asyncio.to_thread, to_thread(), run_coroutine_threadsafe, requests.get, create_subprocess_exec

## Not every library knows about the event loop {#overview}
Asyncio works well only when every slow call can be awaited. Many useful libraries are not written that way: they block the thread while they wait. Calling one directly from a coroutine freezes the event loop. Asyncio offers ways to run such code elsewhere, and to start external programs without waiting.

## requests.get {#requests-get}
`requests.get(url)` is the call from the popular `requests` library that sends an HTTP GET request. It blocks until the reply arrives. Used directly in a coroutine, it stops every other task until it finishes. It is the standard example of blocking code that needs to be moved off the loop.

## run_in_executor {#run-in-executor}
`loop.run_in_executor(executor, function, *args)` runs a blocking function in a thread pool or process pool, and returns an awaitable. The loop stays free while the function runs. Passing `None` as the executor uses the loop's default thread pool.

## asyncio.to_thread {#asyncio-to-thread}
`asyncio.to_thread(function, *args)` runs a blocking function in a worker thread and awaits its result. It is a shorter and friendlier form of `run_in_executor` for the common case of a thread. It also keeps the context variables of the calling task.

## to_thread() {#to-thread}
`to_thread()` is the short name of the same function. It suits blocking I/O, such as a file read or a call to a library that has no async version. It does not make calculation faster, because of the GIL.

## run_coroutine_threadsafe {#run-coroutine-threadsafe}
`asyncio.run_coroutine_threadsafe(coroutine, loop)` goes the other way: from an ordinary thread, it schedules a coroutine on a running loop and returns a `concurrent.futures.Future`. It is the safe bridge from threads back to the loop, for example from a GUI thread.

## create_subprocess_exec {#create-subprocess-exec}
`asyncio.create_subprocess_exec(program, *args)` starts an external program without blocking the loop. It returns a process object whose output you can read with awaitable calls, and whose end you can await with `await process.wait()`.

## A worked example {#example}
A blocking function moved off the loop, so that a ticking task goes on.

```python
import asyncio
import time

def blocking():
    time.sleep(0.2)
    return "done"

async def ticker(log):
    for _ in range(4):
        log.append("tick")
        await asyncio.sleep(0.04)

async def main():
    log = []
    t = asyncio.create_task(ticker(log))
    result = await asyncio.to_thread(blocking)
    await t
    print(result)        # done
    print(len(log))      # 4

asyncio.run(main())
```

The blocking function ran in a thread for 0.2 seconds. The ticker kept ticking meanwhile, four times in all. With `blocking()` called directly in the coroutine, the ticker would not have ticked until the function finished.

The older form:

```python
import asyncio

def add(a, b):
    return a + b

async def main():
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, add, 2, 3)
    print(result)  # 5

asyncio.run(main())
```

Both do the same job. Use `to_thread` for threads, and `run_in_executor` when you pass a process pool for calculation.

From a thread back into the loop:

```python
import asyncio
import threading

async def answer():
    return 42

async def main():
    loop = asyncio.get_running_loop()
    out = []

    def worker():
        future = asyncio.run_coroutine_threadsafe(answer(), loop)
        out.append(future.result())

    t = threading.Thread(target=worker)
    t.start()
    await asyncio.to_thread(t.join)
    print(out)  # [42]

asyncio.run(main())
```

The thread scheduled `answer()` on the loop and waited for the result. The main coroutine meanwhile waited for the thread without blocking.

Starting an external program:

```python
import asyncio
import sys

async def main():
    process = await asyncio.create_subprocess_exec(
        sys.executable, "-c", "print('hi from child')",
        stdout=asyncio.subprocess.PIPE,
    )
    out, _ = await process.communicate()
    print(out.decode().strip())  # hi from child
    print(process.returncode)    # 0

asyncio.run(main())
```

The loop was free while the child ran. `communicate()` collected its output.

Count the effect. A handler that calls `requests.get()` directly, 0.5 seconds per call, can serve 2 requests a second in total, since each call blocks all others. With `await asyncio.to_thread(requests.get, url)` and a thread pool of 20, it can serve up to 40 per second. A native async client would use even less memory, but the thread version needs no rewrite.

| Situation | Use |
|---|---|
| Blocking I/O call | `asyncio.to_thread` |
| Heavy calculation | `run_in_executor` with a process pool |
| Thread calls into the loop | `run_coroutine_threadsafe` |
| External command | `create_subprocess_exec` |

## A common mistake {#mistake}
Calling a blocking function straight from a coroutine and watching the whole program slow down. Another is expecting `to_thread` to speed up a calculation, which the GIL prevents.

## Key points {#key-points}
- Blocking calls such as `requests.get()` must not run in the event loop.
- `asyncio.to_thread` and `run_in_executor` move them to a worker.
- `run_coroutine_threadsafe` is the bridge from a thread back to the loop.
- `create_subprocess_exec` runs external programs without blocking.

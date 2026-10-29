# Pool and asyncio names that only look real

**Covers:** pool_size, ProcessPool, ExpiredError, NONE_COMPLETED, flush_sync, run_forever

## Guessing at a name {#overview}
Executors, asyncio waits, streams and signal handlers each have names you must know by heart. A guess built from plain English is often close and still wrong. The six names below are the near-misses that questions use as wrong answers. For each one, you will see what is really meant.

## pool_size {#pool-size}
`pool_size` is not an argument of an executor. A thread pool executor limits its threads with `max_workers`. The word "pool size" is a good description in prose, and some other libraries use it as a real argument, but here the call `ThreadPoolExecutor(pool_size=8)` stops with a `TypeError` about an unexpected keyword.

## ProcessPool {#process-pool}
`ProcessPool` is a shortened form of the real class name, `ProcessPoolExecutor`. The shortened name is not defined in `concurrent.futures`, so importing it fails. When people say "process pool" in conversation they mean that executor, which runs work in separate processes so that each has its own interpreter. Do not confuse it with `multiprocessing.Pool`, which is a different, older class.

## ExpiredError {#expired-error}
`ExpiredError` is not an exception that Python raises. When `result(timeout=...)` runs out of time, the exception is `TimeoutError`. The future itself is not harmed: after the timeout you can wait on it again.

## NONE_COMPLETED {#none-completed}
`NONE_COMPLETED` is not a constant of asyncio. The function `asyncio.wait()` takes a `return_when` argument with exactly three choices: `FIRST_COMPLETED`, `FIRST_EXCEPTION` and `ALL_COMPLETED`. "None completed" would mean waiting for nothing, which is never useful. The default is `ALL_COMPLETED`.

## flush_sync {#flush-sync}
`flush_sync` is not a method of an asyncio stream writer. After you write data, the way to apply backpressure is `await writer.drain()`. It pauses your coroutine until the buffer has room again, so a fast writer cannot fill memory while a slow reader falls behind. Note that `drain()` must be awaited, so a plain, non-async version would not fit the model.

## run_forever {#run-forever}
`run_forever` is real, but it is not the answer to "how do I register a signal handler". `loop.run_forever()` starts the event loop and keeps it going until `stop()` is called. A handler is registered with `loop.add_signal_handler(sig, callback)`, which is safe to use with asyncio because the callback runs inside the loop, not in an arbitrary moment like a normal signal handler. This is a name that exists but does a different job.

## A worked example {#example}
First, the argument name of a thread pool.

```python
from concurrent.futures import ThreadPoolExecutor

try:
    ThreadPoolExecutor(pool_size=8)
except TypeError:
    print("pool_size is not an argument")  # pool_size is not an argument

pool = ThreadPoolExecutor(max_workers=8)
print("max_workers accepted")  # max_workers accepted
pool.shutdown()
```

The invented argument failed, and the real one worked. Next, the class names and the timeout.

```python
import concurrent.futures as cf

print(hasattr(cf, "ProcessPool"))  # False
print(hasattr(cf, "ProcessPoolExecutor"))  # True
print(hasattr(cf, "ExpiredError"))  # False
print(hasattr(cf, "TimeoutError"))  # True
```

The module has the full executor name and its timeout error, but not the short names.

```text
import time
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=1) as pool:
    future = pool.submit(time.sleep, 0.2)
    try:
        future.result(timeout=0.01)
    except TimeoutError:
        print("not ready yet")  # not ready yet
    future.result()
    print(future.done())  # True
```

The first wait gave up after 0.01 seconds, and the work was still running. A second wait, without a limit, found the finished result.

The constants of `asyncio.wait`:

```python
import asyncio

print(hasattr(asyncio, "FIRST_COMPLETED"))  # True
print(hasattr(asyncio, "ALL_COMPLETED"))  # True
print(hasattr(asyncio, "NONE_COMPLETED"))  # False

async def main():
    fast = asyncio.create_task(asyncio.sleep(0.01, "fast"))
    slow = asyncio.create_task(asyncio.sleep(0.5, "slow"))
    done, pending = await asyncio.wait({fast, slow}, return_when=asyncio.FIRST_COMPLETED)
    print(len(done), len(pending))  # 1 1
    slow.cancel()

asyncio.run(main())
```

With `FIRST_COMPLETED`, the call returned as soon as the fast task finished. One task was done and one still pending, and we cancelled the slow one.

## A common mistake {#mistake}
People often write the name they would like to exist. The error comes late: a `TypeError` for a wrong argument, an `ImportError` for a missing class, an `AttributeError` for a missing method. A second slip is to pick a real name for the wrong job, as with `run_forever` for signals. When a name you know is real does not work, check what it actually does.

## Key points {#key-points}
- A thread pool is limited with `max_workers`, and the full class name is `ProcessPoolExecutor`.
- A future's timeout raises `TimeoutError`.
- `asyncio.wait()` accepts `FIRST_COMPLETED`, `FIRST_EXCEPTION` and `ALL_COMPLETED`.
- A stream writer is relieved with `await writer.drain()`.
- Signals are registered with `loop.add_signal_handler()`, and `run_forever()` is a different job.

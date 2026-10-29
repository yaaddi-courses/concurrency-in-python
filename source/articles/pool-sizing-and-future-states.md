# Pool size, backpressure and the states of a future object

**Covers:** executor.map, max_workers, backpressure, done(), cancel(), cancelled(), TimeoutError, InvalidStateError

## Controlling how much runs at once {#overview}
A pool does the work of many threads or processes with a simple interface. To use it well you need to choose its size, avoid feeding it faster than it can work, and know what state each job is in. This article covers those controls.

## executor.map {#executor-map}
`executor.map(function, items)` runs the function over every item in the pool, and returns the results in input order. It looks like the built-in `map()` and is the shortest way to run many similar jobs. If a job raises an error, the error appears when you reach that result while iterating.

## max_workers {#max-workers}
`max_workers` is the argument that sets how many workers the pool uses. For a thread pool, more workers than cores is normal for waiting jobs. For a process pool, the default is the number of cores, and going far above it adds overhead without speed.

## Backpressure {#backpressure}
Backpressure means slowing down the producer of work when the consumers cannot keep up. Without it, a fast producer fills memory with queued jobs. Techniques include a queue with a maximum size, which makes `put()` wait when full, and a semaphore that limits how many jobs are waiting.

## done() {#done}
`done()` returns `True` if the future object has finished, failed or been cancelled. It does not wait and does not tell you which of the three it was.

## cancel() {#cancel}
`cancel()` tries to cancel a future object that has not finished. A job that is still waiting in the pool's queue can be cancelled. A job that is already running cannot, and `cancel()` returns `False`.

## cancelled() {#cancelled}
`cancelled()` returns `True` if the future object was cancelled before it started.

## TimeoutError {#timeouterror}
`TimeoutError` is raised when `result(timeout=...)` waits longer than the limit. The job itself is not stopped. You only stopped waiting for it.

## InvalidStateError {#invalidstateerror}
`InvalidStateError` is raised when you try to set a result or an error on a future object that is already finished. You see it when writing your own future object-like code, and rarely in ordinary use.

## A worked example {#example}
Cancelling queued work. A pool of one worker gets three jobs, so two must wait in line.

```python
from concurrent.futures import ThreadPoolExecutor
import time

with ThreadPoolExecutor(max_workers=1) as pool:
    first = pool.submit(time.sleep, 0.3)
    second = pool.submit(time.sleep, 0.3)
    time.sleep(0.05)
    print(first.cancel())     # False
    print(second.cancel())    # True
    print(second.cancelled()) # True
    print(first.done())       # False
```

The first job was already running, so it cannot be cancelled. The second was still waiting, so `cancel()` worked. After the `with` block, the first job finishes.

A timeout does not stop the job:

```python
from concurrent.futures import ThreadPoolExecutor, TimeoutError
import time

with ThreadPoolExecutor() as pool:
    future = pool.submit(time.sleep, 0.3)
    try:
        future.result(timeout=0.05)
    except TimeoutError:
        print("gave up waiting")  # gave up waiting
    print(future.done())  # False
```

We stopped waiting after 0.05 seconds, but the job carries on, and the `with` block waits for it at the end.

A state error:

```python
from concurrent.futures import Future
from concurrent.futures import InvalidStateError

f = Future()
f.set_result(1)
try:
    f.set_result(2)
except InvalidStateError:
    print("already finished")  # already finished
print(f.result())  # 1
```

The second `set_result` is refused, since the first already finished the future object.

Backpressure in numbers. A producer makes 1,000 jobs a second, and the workers handle 100 a second. Without a limit, the queue grows by 900 jobs every second, so after one minute 54,000 jobs wait in memory. If each job holds 10 KB, that is 540 MB, still growing. With a limit of 200 waiting jobs, the producer is made to wait, memory stays near 2 MB, and the rate settles at the 100 per second that the workers can do.

Choosing `max_workers`. For jobs that mostly wait, such as web requests, start with 10 to 50 and measure. For calculation in processes, use the number of cores from `os.cpu_count()`. Too many workers can overload the other side, such as a server that allows only 10 connections.

| Question | Answer |
|---|---|
| Can I cancel a running job? | No |
| Does a timeout stop the job? | No, it only stops the waiting |
| What if the producer is faster? | Add backpressure |

## A common mistake {#mistake}
Believing that a timeout kills the job. It leaves it running, using up a worker. Another is an unlimited queue in front of a slow pool, which silently uses more and more memory.

## Key points {#key-points}
- `executor.map` keeps input order. `max_workers` sets the pool size.
- Backpressure stops a fast producer from flooding a slow consumer.
- Only a queued job can be cancelled. A timeout stops the waiting, not the job.
- `InvalidStateError` means a result was set twice.

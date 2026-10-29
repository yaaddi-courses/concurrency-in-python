# Executors and Future objects

**Covers:** ThreadPoolExecutor, ProcessPoolExecutor, Future object, Future, submit(), map(), shutdown(), result()

## Pools instead of one thread per job {#overview}
Starting a thread or a process for every small job wastes time and can use up the machine. A pool starts a fixed number of workers once, and feeds them jobs as they become free. The `concurrent.futures` module gives threads and processes the same simple interface for this.

## ThreadPoolExecutor {#threadpoolexecutor}
`ThreadPoolExecutor` runs jobs in a pool of threads. It suits I/O-bound work, such as fetching web pages, since the threads spend their time waiting.

## ProcessPoolExecutor {#processpoolexecutor}
`ProcessPoolExecutor` runs jobs in a pool of processes. It suits CPU-bound work, since each process has its own GIL and can use its own core. The functions and arguments must be picklable, because they are sent to other processes.

## submit() {#submit}
`submit()` schedules a function on the executor and returns a `Future` straight away. You pass the function and its arguments: `executor.submit(pow, 2, 5)`.

## Future object {#future-object}
A Future object is a placeholder for a result that will be ready later. The job runs in a worker, and the future stands for its eventual answer or error. You can ask whether it is finished, wait for it, or cancel it before it starts.

## Future {#future}
A `Future` is the class of those placeholders, from `concurrent.futures`. Asyncio has its own, similar `Future` class for its tasks. The idea is the same in both: a promise of a result that is not ready yet.

## result() {#result}
`result()` returns the future's value, waiting for it if necessary, or raises the exception that the job raised. It accepts a `timeout`.

## map() {#map}
`map()` runs a function over many inputs in the executor and yields the results in the same order as the inputs, even if jobs finish in a different order.

## shutdown() {#shutdown}
`shutdown()` tells the executor to finish its work and free its workers. Using the executor in a `with` block calls it for you when the block ends, and waits for the jobs.

## A worked example {#example}
```python
from concurrent.futures import ThreadPoolExecutor
import time

def slow_square(n):
    time.sleep(0.1)
    return n * n

with ThreadPoolExecutor(max_workers=4) as pool:
    future = pool.submit(slow_square, 6)
    print(future.result())  # 36
    results = list(pool.map(slow_square, [1, 2, 3, 4]))
    print(results)  # [1, 4, 9, 16]
```

`submit()` returned at once with a future. `result()` waited the 0.1 seconds and gave 36. `map()` ran four jobs on four workers together, so they took about 0.1 seconds in all, not 0.4, and returned the answers in input order.

Errors travel with the future:

```python
from concurrent.futures import ThreadPoolExecutor

def fail():
    raise ValueError("bad job")

with ThreadPoolExecutor() as pool:
    future = pool.submit(fail)
    try:
        future.result()
    except ValueError as error:
        print(error)  # bad job
```

The exception was raised inside the worker, kept in the future, and re-raised when `result()` was called. It does not crash the program until you ask for the result.

Processes for calculation:

```text
from concurrent.futures import ProcessPoolExecutor

def cube(n):
    return n ** 3

if __name__ == "__main__":
    with ProcessPoolExecutor() as pool:
        print(list(pool.map(cube, range(5))))  # [0, 1, 8, 27, 64]
```

The same interface, different workers. The guard is needed for the same reason as before.

Count a case. 8 jobs of 1 second each (a wait) with a pool of 4 threads: two rounds, 2 seconds. With a pool of 8: 1 second. With a single thread: 8 seconds. For a calculation that takes 1 second of CPU each, 4 threads still take about 8 seconds, because of the GIL, while 4 processes on 4 cores take about 2 seconds.

| Need | Use |
|---|---|
| Waiting on network or disk | `ThreadPoolExecutor` |
| Heavy calculation | `ProcessPoolExecutor` |
| Results in input order | `map()` |
| One job, result later | `submit()` |

## A common mistake {#mistake}
Calling `result()` right after each `submit()` in a loop, which waits for every job in turn and removes the concurrency. Submit them all first, then collect. Another is using a thread pool for calculation and expecting a speed-up.

## Key points {#key-points}
- A pool reuses workers. Thread pools suit waiting, process pools suit calculation.
- `submit()` gives a `Future`, and `result()` waits for it or raises its error.
- `map()` keeps the input order.
- Use `with` so `shutdown()` is called for you.

# Handling results as they finish

**Covers:** as_completed, add_done_callback, ALL_COMPLETED, FIRST_COMPLETED, FIRST_EXCEPTION, None

## Do not wait for the slowest {#overview}
With `map()`, you get results in input order, so a slow first job holds back the others. Often you would rather handle each result the moment it is ready. The `concurrent.future objects` module has three tools for that, and a function to wait for groups of jobs.

## as_completed {#as-completed}
`as_completed(futures)` is a function that yields each future object as it finishes, in the order of completion and not of submission. You loop over it, and in each turn the next ready future object is available at once.

## add_done_callback {#add-done-callback}
`add_done_callback(function)` is a method of a future object. It registers a function to be called with the future object when it finishes. The callback runs in the thread that finished the job (or immediately, if the job is already done), so keep it short.

## FIRST_COMPLETED {#first-completed}
`FIRST_COMPLETED` is an option of `concurrent.futures.wait()`. The call returns as soon as any one future object has finished.

## FIRST_EXCEPTION {#first-exception}
`FIRST_EXCEPTION` makes `wait()` return as soon as any future object ends with an error, or when all have finished if none fails.

## ALL_COMPLETED {#all-completed}
`ALL_COMPLETED` is the default option of `wait()`: it returns only when every future object has finished.

## None {#none}
`None` is Python's value for "nothing". Here it appears as the result of a job that returns nothing, such as a function without a `return`. `future object.result()` then gives `None`. The method `future object.exception()` also returns `None` when a job succeeded.

## A worked example {#example}
Three jobs with different durations. The results are handled in the order of finishing.

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def job(name, delay):
    time.sleep(delay)
    return name

with ThreadPoolExecutor() as pool:
    futures = [pool.submit(job, "slow", 0.3),
               pool.submit(job, "fast", 0.05),
               pool.submit(job, "medium", 0.15)]
    order = [f.result() for f in as_completed(futures)]
print(order)  # ['fast', 'medium', 'slow']
```

The jobs were submitted in the order slow, fast, medium, but `as_completed` gave them as they finished. With `map()`, the order would have been slow first, and the quick results would have waited behind it.

A callback:

```python
from concurrent.futures import ThreadPoolExecutor
import time

seen = []

def job():
    time.sleep(0.05)
    return 42

with ThreadPoolExecutor() as pool:
    future = pool.submit(job)
    future.add_done_callback(lambda f: seen.append(f.result()))
print(seen)  # [42]
```

The callback received the finished future object and stored its result. Leaving the `with` block waited for the job, so the callback has run by the time of the print.

Waiting for groups with `wait()`:

```python
from concurrent.futures import ThreadPoolExecutor, wait, FIRST_COMPLETED, ALL_COMPLETED
import time

with ThreadPoolExecutor() as pool:
    futures = [pool.submit(time.sleep, d) for d in (0.05, 0.3)]
    done, pending = wait(futures, return_when=FIRST_COMPLETED)
    print(len(done), len(pending))  # 1 1
    done, pending = wait(futures, return_when=ALL_COMPLETED)
    print(len(done), len(pending))  # 2 0
```

The first `wait` returned after the quick job, with one done and one still pending. The second waited for the rest. `FIRST_EXCEPTION` would have returned early if a job had raised an error, which suits "stop everything if one fails".

A job that returns nothing:

```python
from concurrent.futures import ThreadPoolExecutor

def log(message):
    print(message)  # hello

with ThreadPoolExecutor() as pool:
    future = pool.submit(log, "hello")
    print(future.result())  # None
```

The function printed "hello" and returned `None`. Calling `result()` is still useful, because it waits and re-raises any error.

Count the benefit in a case: 10 downloads of 1 to 10 seconds. With `map()`, the first result may wait behind a 10-second download, while others are ready after 1 second. With `as_completed`, the first result is handled after 1 second, and the screen shows progress at once.

| Tool | Gives you | Order |
|---|---|---|
| `map()` | Results | Input order |
| `as_completed` | Futures | Finishing order |
| `add_done_callback` | A call per future object | When it finishes |
| `wait(...)` | Done and pending sets | Depends on the option |

## A common mistake {#mistake}
Putting slow work in a callback, which blocks the thread that finished the job. Keep callbacks short. Another is using `map()` when the order of finishing matters.

## Key points {#key-points}
- `as_completed` handles each future object as it finishes.
- A done-callback runs when the job ends, in the finishing thread.
- `wait()` can return on `FIRST_COMPLETED`, `FIRST_EXCEPTION` or `ALL_COMPLETED`.
- A job that returns nothing gives `None`.

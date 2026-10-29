# Gathering tasks, task groups and cancellation

**Covers:** asyncio.gather, gather(), CancelledError, TaskGroup, ExceptionGroup, asyncio.wait, LAST_COMPLETED

## Running many tasks and handling their endings {#overview}
Real programs run groups of tasks: ten downloads, three queries. Asyncio offers several ways to run a group and collect the results, and each treats errors and cancellation differently. Knowing the difference decides whether a failure is reported properly or lost.

## asyncio.gather {#asyncio-gather}
`asyncio.gather(*awaitables)` runs several awaitables together and returns their results in the order you passed them. If one fails, the error is raised at once (by default), but the others continue running unless you cancel them yourself.

## gather() {#gather}
`gather()` is the same function called by its short name, after `from asyncio import gather`. With `return_exceptions=True`, errors are returned in the list of results instead of being raised, so one failure does not hide the other results.

## CancelledError {#cancellederror}
`CancelledError` is the exception raised inside a task when it is cancelled. It is raised at the point where the task is waiting. Code can catch it to clean up, but should then raise it again, so that the cancellation is complete.

## TaskGroup {#taskgroup}
`asyncio.TaskGroup` is a context manager that runs tasks as a team. Leaving the `async with` block waits for all tasks. If one fails, the group cancels the rest and then raises the errors, so no task is left running by accident. It is the modern, safer alternative to `gather`.

## ExceptionGroup {#exceptiongroup}
`ExceptionGroup` is an exception that holds several exceptions at once. A `TaskGroup` raises one when more than one task fails. You catch it with `except*`, which matches the contained errors by type.

## asyncio.wait {#asyncio-wait}
`asyncio.wait(tasks, return_when=...)` waits for a set of tasks and returns two sets, the done and the pending. It does not cancel or collect results by itself.

## LAST_COMPLETED {#last-completed}
`LAST_COMPLETED` is not a real option of `asyncio.wait`. The real choices are `FIRST_COMPLETED`, `FIRST_EXCEPTION` and `ALL_COMPLETED`. Wanting "the last one" is just `ALL_COMPLETED`, then looking at the final result.

## A worked example {#example}
Results come back in the order given:

```python
import asyncio

async def job(n, delay):
    await asyncio.sleep(delay)
    return n

async def main():
    results = await asyncio.gather(job(1, 0.05), job(2, 0.01), job(3, 0.02))
    print(results)  # [1, 2, 3]

asyncio.run(main())
```

Job 2 finished first, but the list follows the argument order.

Errors with `return_exceptions`:

```python
import asyncio

async def ok():
    return "fine"

async def bad():
    raise ValueError("oops")

async def main():
    results = await asyncio.gather(ok(), bad(), return_exceptions=True)
    print(results[0])                    # fine
    print(type(results[1]).__name__)     # ValueError

asyncio.run(main())
```

With the option set, the error became an item in the list, and the good result was kept.

A task group:

```text
import asyncio

async def good():
    await asyncio.sleep(0.05)

async def bad():
    raise ValueError("boom")

async def main():
    try:
        async with asyncio.TaskGroup() as group:
            group.create_task(good())
            group.create_task(bad())
    except* ValueError as eg:
        print(len(eg.exceptions))  # 1

asyncio.run(main())
```

The failure of `bad` made the group cancel `good`, then raise an `ExceptionGroup`. The `except*` clause caught the `ValueError` inside it.

Cancellation and `CancelledError`:

```python
import asyncio

async def worker(log):
    try:
        await asyncio.sleep(10)
    except asyncio.CancelledError:
        log.append("cleaning up")
        raise

async def main():
    log = []
    task = asyncio.create_task(worker(log))
    await asyncio.sleep(0.01)
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        log.append("cancelled")
    print(log)  # ['cleaning up', 'cancelled']

asyncio.run(main())
```

The worker caught the error to clean up, and raised it again so the cancellation was complete.

Waiting with `asyncio.wait`:

```python
import asyncio

async def main():
    tasks = [asyncio.create_task(asyncio.sleep(d)) for d in (0.01, 0.2)]
    done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)
    print(len(done), len(pending))  # 1 1
    for t in pending:
        t.cancel()

asyncio.run(main())
```

One finished, one pending. The pending one is cancelled by us, since `wait` does not do it.

| Tool | On a failure | Cleans up the others? |
|---|---|---|
| `gather` | Raises the first error | No |
| `gather(return_exceptions=True)` | Returns the error in the list | Not needed |
| `TaskGroup` | Cancels the rest, raises a group | Yes |

## A common mistake {#mistake}
Catching `CancelledError` and not raising it again, so the task refuses to stop. Another is using `gather` and leaving the other tasks running after one failure.

## Key points {#key-points}
- `gather` collects results in argument order. Use `return_exceptions=True` to keep the good ones.
- `TaskGroup` is the safer way: a failure cancels the rest.
- Catch `CancelledError` only to clean up, then raise it again.
- `asyncio.wait` returns done and pending sets and cancels nothing.

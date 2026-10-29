# Timeouts and shielding in asyncio

**Covers:** asyncio.timeout, timeout(), asyncio.wait_for, wait_for(), asyncio.shield, shield()

## Never wait for ever {#overview}
Anything that touches a network can hang. A program that waits without a limit hangs with it. Asyncio has two ways to put a deadline on a wait, and one way to protect an important job from being cancelled by accident.

## asyncio.timeout {#asyncio-timeout}
`asyncio.timeout(seconds)` is an async context manager that gives the block inside it one shared deadline. When the time runs out, the awaiting code inside is cancelled and a `TimeoutError` is raised at the end of the block. Several awaits in the block share the same limit.

## timeout() {#timeout}
`timeout()` is the short name of the same function. It can be moved with `reschedule()` if the deadline needs to change while the block runs.

## asyncio.wait_for {#asyncio-wait-for}
`asyncio.wait_for(awaitable, timeout)` waits for one awaitable, and raises `TimeoutError` if it takes too long. It also cancels that awaitable when the time is up. It is the older tool, for a single wait.

## wait_for() {#wait-for}
`wait_for()` is the short name of the same function. It is best for one operation with its own limit.

## asyncio.shield {#asyncio-shield}
`asyncio.shield(awaitable)` wraps work so that cancelling the waiter does not cancel the work. The outer wait ends, but the inner job carries on to the finish. Use it for work that must complete, such as a database write.

## shield() {#shield}
`shield()` is what you write after importing the name directly. Keep a reference to the shielded job even when you stop waiting for it, so that its result or error is not lost.

## A worked example {#example}
A deadline on one operation:

```text
import asyncio

async def main():
    try:
        await asyncio.wait_for(asyncio.sleep(1), timeout=0.05)
    except TimeoutError:
        print("too slow")  # too slow

asyncio.run(main())
```

The sleep of 1 second was cancelled after 0.05 seconds, and `TimeoutError` was raised.

One deadline for a block of steps:

```text
import asyncio

async def step():
    await asyncio.sleep(0.04)

async def main():
    try:
        async with asyncio.timeout(0.1):
            await step()
            await step()
            await step()
    except TimeoutError:
        print("whole block timed out")  # whole block timed out

asyncio.run(main())
```

Each step takes 0.04 seconds, within the budget of 0.1 for the whole block. Three steps need 0.12 seconds, so the third is cut off. With `wait_for` on each step, none would time out, because each step is below its own limit. That is the key difference: `timeout` limits the sum, `wait_for` limits one.

Shielding:

```text
import asyncio

async def save(log):
    await asyncio.sleep(0.05)
    log.append("saved")

async def main():
    log = []
    inner = asyncio.create_task(save(log))
    try:
        await asyncio.wait_for(asyncio.shield(inner), timeout=0.01)
    except TimeoutError:
        log.append("stopped waiting")
    await inner
    print(log)  # ['stopped waiting', 'saved']

asyncio.run(main())
```

The wait was cancelled after 0.01 seconds, but the shield kept the inner job alive. It finished and saved. Without the shield, the timeout would have cancelled `save` too, and nothing would be saved.

Count the effect on a job that makes 3 calls, each taking up to 2 seconds. With a deadline of 3 seconds for the whole job, `asyncio.timeout(3)` stops it at 3 seconds in the worst case. With `wait_for(call, 3)` on each, the worst case is 9 seconds, which may break the promise to the user.

| Tool | Limits | On time out |
|---|---|---|
| `timeout` | A block, total time | Cancels the block |
| `wait_for` | One awaitable | Cancels that awaitable |
| `shield` | Nothing; protects | Outer wait cancelled, inner continues |

## A common mistake {#mistake}
Using `wait_for` on every step and believing the whole job is limited. Use one `timeout` around the job. Another is shielding a job and then forgetting it, so that its error is never seen.

## Key points {#key-points}
- `asyncio.timeout` gives a block one shared deadline. `wait_for` limits a single wait.
- Both raise `TimeoutError` and cancel the work.
- `shield` lets inner work survive an outer cancellation.
- Keep a reference to a shielded job.

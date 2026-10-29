# Tasks, cooperative multitasking and starving the loop

**Covers:** cooperative multitasking, create_task, asyncio.Task, event loop starvation, starvation

## Starting work and letting it run {#overview}
Awaiting a coroutine runs it, but the caller waits for it to finish. To start something and carry on, you need a task. And because asyncio's jobs take turns by agreement, one job that refuses to give way can freeze all of them. This article covers both sides of that.

## Cooperative multitasking {#cooperative-multitasking}
Cooperative multitasking is a way of sharing one thread in which each job gives up control by itself, at points such as `await`. Nothing interrupts a job in the middle. This is the opposite of threads, where the system switches them. The benefit: no job is cut off halfway, so many race conditions disappear. The risk: a job that does not give way blocks everyone.

## create_task {#create-task}
`asyncio.create_task(coroutine)` wraps a coroutine in a task and schedules it to run on the event loop, without waiting for it. It returns the task at once. The coroutine starts running the next time the current one awaits something.

## asyncio.Task {#asyncio-task}
An `asyncio.Task` is the object that represents a scheduled coroutine. You can await it to get the result, ask whether it is done, or cancel it. It is a kind of `Future`. Keep a reference to every task you create, because the loop keeps only weak references, and a task nobody holds may be removed before it ends.

## Event loop starvation {#event-loop-starvation}
Event loop starvation happens when one coroutine runs for a long time without awaiting, so the loop cannot run any other coroutine. Timers do not fire, tasks make no progress and, in a server, new connections wait. The program seems frozen, though nothing has failed.

## Starvation {#starvation}
Starvation is the general name for any job being denied the chance to run because others take all the time. In asyncio, it is the usual result of calling slow, blocking code from a coroutine.

## A worked example {#example}
Starting two tasks and then waiting for them.

```python
import asyncio

async def work(name, delay, log):
    await asyncio.sleep(delay)
    log.append(name)

async def main():
    log = []
    t1 = asyncio.create_task(work("slow", 0.1, log))
    t2 = asyncio.create_task(work("fast", 0.02, log))
    log.append("started")
    await t1
    await t2
    print(log)  # ['started', 'fast', 'slow']
    print(t1.done(), t2.done())  # True True

asyncio.run(main())
```

`create_task` returned at once, so "started" was logged first, before either task did anything. Then the loop ran the tasks while `main` awaited. The fast one finished first.

The difference between awaiting a coroutine and making a task:

```python
import asyncio
import time

async def job():
    await asyncio.sleep(0.1)

async def main():
    start = time.perf_counter()
    await job()
    await job()
    one_after_another = time.perf_counter() - start

    start = time.perf_counter()
    t1 = asyncio.create_task(job())
    t2 = asyncio.create_task(job())
    await t1
    await t2
    together = time.perf_counter() - start
    print(one_after_another > together * 1.5)  # True

asyncio.run(main())
```

Awaiting two jobs in a row took about 0.2 seconds. Making both tasks first overlapped the waits, about 0.1 seconds.

Starvation. A coroutine that calculates without awaiting:

```python
import asyncio
import time

async def blocker():
    time.sleep(0.2)

async def ticker(ticks):
    for _ in range(3):
        ticks.append(time.perf_counter())
        await asyncio.sleep(0.01)

async def main():
    ticks = []
    t = asyncio.create_task(ticker(ticks))
    await asyncio.sleep(0)
    await blocker()
    await t
    gap = max(b - a for a, b in zip(ticks, ticks[1:]))
    print(gap > 0.15)  # True

asyncio.run(main())
```

The ticker wanted to tick every 0.01 seconds. While `blocker` held the loop for 0.2 seconds with `time.sleep`, no tick could happen, so a gap of about 0.2 seconds appeared. The fix is to use `await asyncio.sleep(0.2)`, or to move the slow work to a thread, described in a later article.

Count the damage in a server. If one handler blocks the loop for 0.5 seconds, then every other connection waits 0.5 seconds too. With 20 calls a second, each blocking for 0.5 seconds, the server cannot keep up: it needs 10 seconds of work every second.

| Style | Who decides the switch | Typical risk |
|---|---|---|
| Threads | The system | Races |
| Cooperative (asyncio) | Each job, at `await` | One blocking job stalls all |

## A common mistake {#mistake}
Creating a task and not keeping a reference to it, so it may be collected before finishing. Another is long calculation inside a coroutine with no `await`, which starves every other task.

## Key points {#key-points}
- Cooperative multitasking switches only at `await`.
- `create_task` starts a coroutine without waiting. Keep the task object.
- Blocking code in a coroutine starves the whole event loop.
- Waiting overlaps only when tasks are created before they are awaited.

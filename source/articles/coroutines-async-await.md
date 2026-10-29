# Coroutines, async def and await

**Covers:** coroutine, async def, await, awaiting, asyncio.run, asyncio.sleep, SyntaxError, RuntimeWarning

## One thread, many jobs in progress {#overview}
Threads switch whenever the system decides. Asyncio takes a different road: one thread, and the jobs themselves say when they can pause. A job that has to wait says so, the loop runs another job meanwhile, and the first resumes when its wait is over. The tools for this are a few keywords and functions.

## Coroutine {#coroutine}
A coroutine is a function that can pause in the middle and be resumed later. You define one with `async def`. Calling it does not run it. It returns a coroutine object, which is a recipe waiting to be run by the event loop.

## async def {#async-def}
`async def` is the keyword pair that defines a coroutine function. Inside it, you may use `await`. Outside such a function, `await` is not allowed.

## await {#await}
`await` pauses the current coroutine until the thing after it has finished, and hands control back to the event loop so that other coroutines can run. It gives you the result of what it waited for. It may only appear inside an `async def`.

## Awaiting {#awaiting}
Awaiting is the act of using `await` on something: a coroutine, a task or a future object. A coroutine that is never awaited never runs.

## asyncio.run {#asyncio-run}
`asyncio.run(main())` starts an event loop, runs the given coroutine until it finishes, and closes the loop. It is the normal entry point of an asyncio program, called once at the top.

## asyncio.sleep {#asyncio-sleep}
`asyncio.sleep(seconds)` is the awaitable version of waiting. While one coroutine sleeps, the loop runs others. It is the right replacement for `time.sleep()` inside coroutines.

## SyntaxError {#syntaxerror}
`SyntaxError` is raised when the code breaks the grammar of Python. Using `await` in an ordinary function is one such case, and the program does not even start.

## RuntimeWarning {#runtimewarning}
`RuntimeWarning` is a warning, not an error, that Python prints when something is suspicious while the program runs. The best known here is "coroutine ... was never awaited", which shows that you called a coroutine function and forgot `await`.

## A worked example {#example}
Two coroutines that each wait 0.1 seconds, run together.

```python
import asyncio
import time

async def fetch(name):
    await asyncio.sleep(0.1)
    return name

async def main():
    start = time.perf_counter()
    a, b = await asyncio.gather(fetch("a"), fetch("b"))
    elapsed = time.perf_counter() - start
    print(a, b)                 # a b
    print(elapsed < 0.19)       # True

asyncio.run(main())
```

`asyncio.run(main())` started the loop. Inside `main`, `gather` ran both coroutines together. Each awaited `asyncio.sleep(0.1)`, which handed control to the loop, so the waits overlapped. The total was about 0.1 seconds, not 0.2. A later article explains `gather`.

Forgetting `await`:

```python
import asyncio

async def hello():
    return "hi"

async def main():
    result = hello()
    print(type(result).__name__)  # coroutine
    print(await result)           # hi

asyncio.run(main())
```

`hello()` alone gave a coroutine object, not the text. Awaiting it ran it. If the line `result = hello()` is never followed by an `await`, Python prints a `RuntimeWarning: coroutine 'hello' was never awaited` when the object is thrown away.

Using `await` in a normal function is a `SyntaxError`:

```python
try:
    compile("def f():\n    await g()\n", "<test>", "exec")
except SyntaxError:
    print("await needs async def")  # await needs async def
```

Python refuses to compile the function.

Count the difference with time. Ten tasks that each await a 0.1-second wait: one by one, 1 second; together, about 0.1 seconds. This gain is for waiting only. A coroutine that calculates for 0.1 seconds without awaiting holds the loop for all that time, and nothing else can run.

| Piece | What it does |
|---|---|
| `async def` | Defines a coroutine function |
| `await x` | Pauses until `x` is done, lets others run |
| `asyncio.run(c)` | Runs `c` in a new event loop |
| `asyncio.sleep(n)` | Waits without blocking the loop |

## A common mistake {#mistake}
Using `time.sleep()` inside a coroutine. It blocks the whole loop, since it does not hand control back. Use `await asyncio.sleep()`. Another is calling a coroutine function and never awaiting the result.

## Key points {#key-points}
- `async def` defines a coroutine. Calling it gives an object that does nothing until awaited.
- `await` pauses the coroutine and lets the loop run others.
- `asyncio.run()` is the entry point, called once.
- Never use blocking calls such as `time.sleep()` inside a coroutine.

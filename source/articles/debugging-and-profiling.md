# Debugging and profiling concurrent programs

**Covers:** cProfile, asyncio debug mode, SIGSTOP, SIGCONT

## Finding out why it is slow or stuck {#overview}
Concurrent programs fail in ways that ordinary ones do not: they freeze, they are slower than expected, or they print warnings about things that never ran. Python gives you a profiler for finding slow code, a debug mode for asyncio, and signals that can freeze and resume a process from outside.

## cProfile {#cprofile}
`cProfile` is Python's built-in profiler. It records how many times each function was called and how much time it used. It tells you where the time goes. It measures the calling thread's time, so for threads and processes, profile the worker's function on its own.

## Asyncio debug mode {#asyncio-debug-mode}
Asyncio debug mode turns on extra checks in the event loop. It reports coroutines that were never awaited, and it logs any callback or task step that blocks the loop for more than 100 milliseconds. Turn it on with `asyncio.run(main(), debug=True)` or by setting the environment variable `PYTHONASYNCIODEBUG=1`. It slows the program, so use it for development.

## SIGSTOP {#sigstop}
`SIGSTOP` pauses a process from outside, and the process cannot refuse or catch it. The process does nothing until it is resumed. It is useful for freezing a worker to see what the others do.

## SIGCONT {#sigcont}
`SIGCONT` resumes a process that was paused by SIGSTOP. Together they are the Unix way of pausing and continuing a program without changing its code.

## A worked example {#example}
Profile a function that does two kinds of work:

```python
import cProfile
import pstats
import io

def slow():
    return sum(i * i for i in range(200_000))

def fast():
    return 1 + 1

def run():
    slow()
    fast()

profiler = cProfile.Profile()
profiler.enable()
run()
profiler.disable()

out = io.StringIO()
stats = pstats.Stats(profiler, stream=out)
names = [func[2] for func in stats.stats]
print("slow" in names and "fast" in names)  # True
```

The profile lists both functions, with the number of calls and time spent for each. In a real report, `slow` takes nearly all the time, and `fast` almost none. That tells you where to look. From the command line, `python -m cProfile -s cumulative script.py` prints the sorted table.

Asyncio debug mode finds a blocking call:

```python
import asyncio
import logging
import time

records = []

class Collect(logging.Handler):
    def emit(self, record):
        records.append(record.getMessage())

logging.getLogger("asyncio").addHandler(Collect())

async def blocker():
    time.sleep(0.2)

asyncio.run(blocker(), debug=True)
print(any("took" in message for message in records))  # True
```

In debug mode, the loop noticed that one step took more than 0.1 seconds and logged a message that says it "took 0.2 seconds". That is how you find the call that starves the loop.

Signals, shown for Unix:

```text
kill -STOP 4321     # freeze process 4321
kill -CONT 4321     # let it continue
```

While the process is stopped, its timers keep counting in the other processes, so a parent that waits with a timeout will see it as slow. It is a good way to test how a program behaves when one worker stalls, without touching the code.

A debugging routine for a program that seems frozen:

1. Is the CPU busy (a loop that never awaits or a calculation) or idle (waiting for something)?
2. If idle, a deadlock or a lost wake-up is likely. Print every thread's stack with `faulthandler.dump_traceback`.
3. If busy, profile it, and in asyncio, turn on debug mode to find the blocking call.
4. Reduce the program to the smallest version that still fails.

Count: a profile that shows `slow` taking 95% of the run is a clear target; optimizing the other 5% cannot give more than a 5% gain, however clever.

| Tool | Finds |
|---|---|
| `cProfile` | Which functions use the time |
| asyncio debug mode | Unawaited coroutines, blocked loop |
| `SIGSTOP`, `SIGCONT` | How the system reacts to a stalled process |

## A common mistake {#mistake}
Optimizing code that the profiler does not show as slow. Measure first. Another is leaving asyncio debug mode on in production, where it costs speed.

## Key points {#key-points}
- `cProfile` shows where time goes. Optimize what it shows.
- Asyncio debug mode reports unawaited coroutines and calls that block the loop.
- SIGSTOP pauses a process and SIGCONT resumes it.
- Profile one worker's function at a time.

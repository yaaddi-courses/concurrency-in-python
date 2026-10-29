# Concurrency and parallelism

**Covers:** concurrency, parallelism, event loop, CPU, os.cpu_count(), time.sleep(), I/O-bound, CPU-bound

## Two words that people mix up {#overview}
Say you cook dinner while a pot boils. You do not stand and watch the pot. You chop vegetables, check the pot now and then, and the whole meal is ready sooner. That is concurrency. Now imagine two cooks, each doing a different dish at the very same moment. That is parallelism. A program can use either idea, and Python gives you different tools for each, so it pays to keep the words apart from the start.

## Concurrency {#concurrency}
Concurrency means structuring a program so that several tasks can each make progress over the same stretch of time. They do not have to run at the same instant. A single worker can switch between tasks, and when one task has to wait, the worker moves on to another. The gain is not speed of calculation, but the time you stop wasting while something waits.

## Parallelism {#parallelism}
Parallelism means actually running several tasks at the exact same instant. It needs hardware that can do two things at once, which means at least two processor cores. Only parallelism makes a calculation finish faster, because the work is really split between cores.

## CPU {#cpu}
The CPU, the central processing unit, is the chip that carries out a program's instructions. A modern CPU has several cores, and each core can follow one stream of instructions at a time. How many cores you have sets the upper limit of parallelism on your machine.

## os.cpu_count() {#os-cpu-count}
`os.cpu_count()` returns how many cores the machine reports. It is the usual starting point for deciding how many parallel workers to start. It can return `None` if the number is unknown, so code that depends on it should handle that case.

## time.sleep() {#time-sleep}
`time.sleep()` pauses the calling thread for a number of seconds. It is the simplest way to imitate waiting, such as waiting for a slow server. While one thread sleeps, others can run, which makes it useful in small demonstrations.

## I/O-bound {#io-bound}
A program is I/O-bound when its speed is limited by waiting on something outside the CPU: a disk, a network, a database. The CPU is mostly idle. Doing other work during the wait helps a lot, so concurrency is the right tool.

## CPU-bound {#cpu-bound}
A program is CPU-bound when its speed is limited by raw calculation. The CPU is busy all the time, and there is nothing to wait for. Only parallelism helps, because only more cores add more calculation.

## Event loop {#event-loop}
An event loop is a mechanism, usually one per program, that runs and schedules tasks one at a time and switches between them whenever one has to wait. It runs in a single thread. The `asyncio` library is built around one. Because only one task runs at a time, it is concurrent, but not parallel.

## A worked example {#example}
Three jobs, each needing a one-second wait, such as three slow web requests.

```python
import time

start = time.perf_counter()
for _ in range(3):
    time.sleep(0.1)
one_by_one = time.perf_counter() - start
print(round(one_by_one, 1))  # 0.3
```

One after another, the waits add up: 3 × 0.1 = 0.3 seconds. Now let each wait happen at the same time, using threads:

```python
import threading
import time

def job():
    time.sleep(0.1)

start = time.perf_counter()
threads = [threading.Thread(target=job) for _ in range(3)]
for t in threads:
    t.start()
for t in threads:
    t.join()
together = time.perf_counter() - start
print(round(together, 1))  # 0.1
```

The three waits overlap, so the total is about 0.1 seconds, a third of the time. That is concurrency paying off for I/O-bound work. No extra core was needed, since the CPU was idle anyway.

Compare a CPU-bound job. Counting to 30 million takes, say, 2 seconds of pure calculation. Three such jobs one after another take 6 seconds. Run concurrently on one core, they take about 6 seconds still, because there is no waiting to hide. Run in parallel on three cores, they take about 2 seconds, in the best case. Check what you have:

```python
import os

print(os.cpu_count() >= 1)  # True
```

| Kind of work | Limited by | Helped by |
|---|---|---|
| I/O-bound | Waiting | Concurrency |
| CPU-bound | Calculation | Parallelism |

A quick way to tell: if the program's CPU use is near 100% on one core, it is CPU-bound. If it sits near 0% while slow, it is I/O-bound.

## A common mistake {#mistake}
Adding more threads to a CPU-bound job and expecting it to speed up. In standard Python it will not, for a reason explained in a later article. Another is starting far more workers than `os.cpu_count()` for calculation, which only adds overhead.

## Key points {#key-points}
- Concurrency overlaps waiting. Parallelism runs things at the same instant on several cores.
- I/O-bound work gains from concurrency. CPU-bound work gains only from parallelism.
- An event loop switches between tasks in one thread.
- `os.cpu_count()` tells you how many cores can work at once.

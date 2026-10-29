# Stopping workers cleanly: poison pills and graceful shutdown

**Covers:** poison pill, graceful shutdown, graceful-shutdown

## Ending a long-running worker without breaking it {#overview}
Workers that loop for ever, taking jobs from a queue, will have to be stopped sooner or later. Killing them is quick, but it can lose a job half done, or leave a file half written. The better way is to tell them to finish what they have and then leave.

## Poison pill {#poison-pill}
A poison pill is a special marker that a producer puts in a queue to tell a consumer to stop. When the consumer takes the marker out, it knows there will be no more work, and ends its loop. A common marker is `None`, or a unique object that can never be real work. Poison pills, the plural, are used when there are several consumers: you need one marker for each of them.

## Graceful shutdown {#graceful-shutdown}
A graceful shutdown is stopping a program in an orderly way: stop taking new work, let the current work finish, save what must be saved, release resources and exit. It is the opposite of pulling the plug, which is fast and loses things.

## Graceful-shutdown {#graceful-shutdown-hyphen}
The hyphenated form, graceful-shutdown, is the same idea used before another word, as in "a graceful-shutdown handler". It means a piece of code that carries out the orderly stop.

## A worked example {#example}
Two workers take jobs from one queue. The parent sends the jobs, then one marker for each worker.

```text
import multiprocessing as mp

STOP = "STOP"

def worker(jobs, results):
    while True:
        job = jobs.get()
        if job == STOP:
            break
        results.put(job * 2)

if __name__ == "__main__":
    jobs = mp.Queue()
    results = mp.Queue()
    procs = [mp.Process(target=worker, args=(jobs, results)) for _ in range(2)]
    for p in procs:
        p.start()
    for n in range(5):
        jobs.put(n)
    for _ in procs:
        jobs.put(STOP)
    answers = sorted(results.get() for _ in range(5))
    for p in procs:
        p.join()
    print(answers)  # [0, 2, 4, 6, 8]
```

The five jobs go in first, then two `STOP` markers, one for each worker. Because the queue is first in, first out, each worker meets its marker only after all the jobs ahead of it have been taken. Each worker leaves its loop when it sees the marker, and the program ends cleanly with every job done.

What if you sent only one marker? One worker would stop, and the other would wait for ever for a job, so `join()` would never return. The rule is one marker per consumer.

Compare with stopping by force. Suppose a worker is halfway through writing a result. A forced stop with `terminate()` can leave an incomplete file or an item that was taken from the queue but never processed. The marker approach lets the worker finish the job it holds before it reads the marker.

A shutdown needs a plan for a worker that does not stop. A common rule: ask nicely, wait for a limit such as 10 seconds, and only then force it:

```text
import multiprocessing as mp
import time

def stubborn():
    time.sleep(30)

if __name__ == "__main__":
    p = mp.Process(target=stubborn)
    p.start()
    p.join(timeout=0.2)
    alive_after_wait = p.is_alive()
    if alive_after_wait:
        p.terminate()
        p.join()
    print(alive_after_wait)  # True
    print(p.is_alive())  # False
```

The join waited 0.2 seconds and then gave up. The worker was still alive, so `terminate()` ended it. This is the last resort, after the polite way has had its chance.

The steps of a graceful shutdown, in order:

1. Stop accepting new work.
2. Send the stop markers.
3. Wait for the workers, with a time limit.
4. Force any that remain.
5. Close files and queues.

| Method | Speed | Risk |
|---|---|---|
| Poison pill | Waits for current work | None, if one is sent per worker |
| `terminate()` | Immediate | Half-done work is lost |

## A common mistake {#mistake}
Sending fewer markers than there are workers, so some wait for ever. Another is using a marker that could also be real work, such as `None` when `None` is a valid job.

## Key points {#key-points}
- A poison pill is a marker in the queue that tells a consumer to stop. Send one per consumer.
- A graceful shutdown finishes current work before exiting.
- Force a stop only after a time limit.
- Choose a marker that real work can never equal.

# Locks, events, semaphores and barriers across processes

**Covers:** multiprocessing.Lock, multiprocessing.RLock, multiprocessing.Event, multiprocessing.Semaphore, multiprocessing.Barrier

## The same tools, built for processes {#overview}
The `threading` module's locks and events work only inside one process. They cannot coordinate separate programs, because their memory is not shared. The multiprocessing module offers a matching set with the same names and the same methods, built on operating-system objects that all processes can reach. If you know the thread versions, you already know these.

## multiprocessing.Lock {#multiprocessing-lock}
`multiprocessing.Lock` is a lock that only one process can hold at a time. It has `acquire()` and `release()`, and works as a context manager with `with lock:`. It protects shared resources such as a file, a shared counter or the screen.

## multiprocessing.RLock {#multiprocessing-rlock}
`multiprocessing.RLock` is the reentrant version: the process that holds it can take it again without waiting for itself, and must release it as many times as it took it.

## multiprocessing.Event {#multiprocessing-event}
`multiprocessing.Event` is a flag shared between processes. One calls `set()`, and others sleeping in `wait()` wake up. `clear()` lowers it again and `is_set()` reads it.

## multiprocessing.Semaphore {#multiprocessing-semaphore}
`multiprocessing.Semaphore` is a counter that admits at most N processes into a block at once. It is the way to limit access to something scarce, such as a licence, a device or a rate-limited service, across all workers.

## multiprocessing.Barrier {#multiprocessing-barrier}
`multiprocessing.Barrier` makes a fixed number of processes wait for each other. Each calls `wait()`, and none continues until all have arrived.

## A worked example {#example}
Output from several processes can mix on the screen. A lock keeps each line whole.

```text
import multiprocessing as mp

def say(lock, name):
    with lock:
        print(f"{name} speaking")

if __name__ == "__main__":
    lock = mp.Lock()
    procs = [mp.Process(target=say, args=(lock, n)) for n in ("A", "B")]
    for p in procs:
        p.start()
    for p in procs:
        p.join()
```

Each process takes the lock, prints its full line and gives it back. Which one goes first can differ between runs.

An event as a start signal for all workers:

```text
import multiprocessing as mp

def worker(go, out):
    go.wait()
    out.put("started")

if __name__ == "__main__":
    go = mp.Event()
    out = mp.Queue()
    procs = [mp.Process(target=worker, args=(go, out)) for _ in range(3)]
    for p in procs:
        p.start()
    go.set()
    results = [out.get() for _ in range(3)]
    for p in procs:
        p.join()
    print(results)  # ['started', 'started', 'started']
```

All three workers wait for the signal. One `set()` releases all of them.

A semaphore that allows two at a time, checked by counting the largest number inside:

```text
import multiprocessing as mp
import time

def job(sem, active, peak):
    with sem:
        with active.get_lock():
            active.value += 1
            peak.value = max(peak.value, active.value)
        time.sleep(0.05)
        with active.get_lock():
            active.value -= 1

if __name__ == "__main__":
    sem = mp.Semaphore(2)
    active = mp.Value("i", 0)
    peak = mp.Value("i", 0)
    procs = [mp.Process(target=job, args=(sem, active, peak)) for _ in range(5)]
    for p in procs:
        p.start()
    for p in procs:
        p.join()
    print(peak.value)  # 2
```

Five processes tried to enter, but at most two were inside together.

A barrier for three workers:

```text
import multiprocessing as mp

def step(barrier, out):
    out.put("before")
    barrier.wait()
    out.put("after")

if __name__ == "__main__":
    barrier = mp.Barrier(3)
    out = mp.Queue()
    procs = [mp.Process(target=step, args=(barrier, out)) for _ in range(3)]
    for p in procs:
        p.start()
    items = [out.get() for _ in range(6)]
    for p in procs:
        p.join()
    print(items[:3])  # ['before', 'before', 'before']
```

No process puts "after" until all three have put "before".

Count the cost of the process version against the thread version. A thread lock takes a fraction of a microsecond. A process lock goes through the operating system and takes a few microseconds, so protecting a tiny operation in a tight loop is expensive. Take the lock around larger pieces of work.

An important rule: you must create the lock in the parent and pass it to the children. A lock created separately inside each child is a different lock and protects nothing.

| Thread version | Process version | Shared by |
|---|---|---|
| `threading.Lock` | `multiprocessing.Lock` | Processes |
| `threading.Event` | `multiprocessing.Event` | Processes |
| `threading.Semaphore` | `multiprocessing.Semaphore` | Processes |
| `threading.Barrier` | `multiprocessing.Barrier` | Processes |

## A common mistake {#mistake}
Using a `threading.Lock` across processes. With spawn, every child gets its own copy, so nothing is protected. Use the multiprocessing version, created once in the parent.

## Key points {#key-points}
- The multiprocessing versions have the same names and methods as the threading ones.
- Create each one in the parent and pass it to the workers.
- They cost more than thread locks, so guard larger blocks of work.
- A plain threading lock does not work between processes.

# Race conditions and critical sections

**Covers:** Locking, race condition, critical section, read-modify-write, livelock, append(), pop()

## When timing decides the answer {#overview}
Two threads that share data can produce a wrong answer even when each one is correct by itself. The bug depends on exact timing, so it may appear once in a thousand runs, and vanish when you add a print statement. This is the hardest kind of bug to catch, and knowing its shape is the best defense.

## Locking {#locking}
Locking means using a lock so that only one thread at a time can touch shared data. It is the standard cure for a race condition, and the next article shows how it is done. Locks have a cost: threads that wait do no work.

## Race condition {#race-condition}
A race condition is a bug where the result depends on the unpredictable order or timing in which several threads touch shared data. The threads are racing, and the result depends on who gets there first. Because the order changes from run to run, the bug is intermittent.

## Critical section {#critical-section}
A critical section is the block of code that touches shared, changeable data, and therefore must not run in two threads at the same time. Making the critical section as small as possible keeps the program fast, but it has to cover the whole of the update, not just a part.

## Read-modify-write {#read-modify-write}
Read-modify-write is the pattern behind most race conditions: read a number, work out a new one from it, then write it back. If another thread does the same in between, one of the two updates is lost. A simple statement like `counter += 1` is this pattern, hidden in one line.

## Livelock {#livelock}
A livelock looks like a freeze, but the threads are not stuck. They are busy, each reacting to the other, and getting nowhere. Two people stepping aside in a corridor, in the same direction, again and again, are in a livelock. Adding a random delay before retrying usually breaks it.

## append() {#append}
`append()` adds one item to the end of a list. In CPython, a single `append()` on a list is safe from threads in practice, but that is a detail of the implementation, so rely on it with care, and not for anything that combines it with another step.

## pop() {#pop}
`pop()` removes and returns an item from a list, the last one by default. A check followed by a pop, such as "if the list is not empty, pop", is a read-modify-write: another thread can empty the list between the check and the pop.

## A worked example {#example}
A shared counter and four threads, each adding one a hundred thousand times.

```python
import threading

counter = 0

def add():
    global counter
    for _ in range(100_000):
        counter += 1

threads = [threading.Thread(target=add) for _ in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()
print(counter <= 400_000)  # True
```

The expected total is 400,000. The line prints `True` because the result can never be larger. On many runs and versions it is smaller, such as 312,000, since some updates were lost. On some runs it is exactly right, which is what makes the bug hard.

Why does `counter += 1` lose updates? Because it is three steps:

1. Read `counter`, say 5.
2. Work out 5 + 1 = 6.
3. Write 6 back.

Suppose thread A reads 5. Before A writes, thread B also reads 5. A writes 6. B writes 6. Two additions, but the counter grew by one. A lost update.

Here is the same idea with `pop()`:

```python
import threading

items = list(range(1000))
taken = []

def take():
    while items:
        try:
            taken.append(items.pop())
        except IndexError:
            return

threads = [threading.Thread(target=take) for _ in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()
print(len(taken))  # 1000
```

The check `while items` and the `pop()` are separate steps, so one thread may find the list non-empty and another may empty it before the first pops. The `try` guards against that, which is why every item is taken exactly once. Without the guard, a thread could crash with a list error.

The critical section here is the update of `counter`, or the check-and-pop. Both must be protected as a whole. The next article shows how.

| Symptom | Likely cause |
|---|---|
| Total sometimes too low | Lost updates (read-modify-write) |
| Crash on empty list, rarely | Check then pop is not one step |
| Everything busy, nothing progresses | Livelock |

How to find them: run the code many times, with many threads, and a small sleep between the read and the write to force bad timing. A race you can reproduce is a race you can fix.

## A common mistake {#mistake}
Believing that a test which passes once proves the code is safe. A race may show up once in ten thousand runs. Another is protecting only half of the update, such as the write but not the read.

## Key points {#key-points}
- A race condition makes the result depend on timing.
- Read-modify-write, such as `counter += 1`, is the usual source.
- The critical section is the code that touches shared data, and it must run in one thread at a time.
- A livelock is busy, not frozen. Random delays help.

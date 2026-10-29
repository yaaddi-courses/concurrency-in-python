# Deadlock, lock ordering and atomic operations

**Covers:** deadlock, deadlocking, lock ordering, atomic

## When locks make things worse {#overview}
Locks fix races, but misused, they create a new problem: threads that wait for each other for ever. The program does not crash and shows no error. It just stops. Knowing the pattern lets you prevent it, since finding it afterwards is hard.

## Deadlock {#deadlock}
A deadlock is a state in which two or more threads wait for each other in a circle, so none of them can ever continue. Thread A holds lock 1 and waits for lock 2, while thread B holds lock 2 and waits for lock 1. Each holds exactly what the other needs, and neither will let go.

## Deadlocking {#deadlocking}
Deadlocking describes the act of getting into that state. A program that is deadlocking may work for weeks, and freeze the one time that the timing is unlucky. It also shows up more under heavy load.

## Lock ordering {#lock-ordering}
Lock ordering is the rule that every thread takes locks in the same fixed order. If everyone takes lock 1 before lock 2, nobody can hold 2 while waiting for 1, so the circle can never form. It is the simplest and most reliable prevention.

## Atomic {#atomic}
An operation is atomic if it cannot be interrupted halfway: other threads see it either not started or fully done. Atomic operations need no lock. In Python, which operations are atomic is a detail of the implementation, so do not rely on it for anything beyond simple cases.

## A worked example {#example}
Two accounts, each with its own lock, and two transfers in opposite directions.

```python
import threading

lock_a = threading.Lock()
lock_b = threading.Lock()
log = []

def one():
    with lock_a:
        with lock_b:
            log.append("one")

def two():
    with lock_b:
        with lock_a:
            log.append("two")
```

If `one` and `two` run at the same moment, `one` may hold `lock_a` and wait for `lock_b`, while `two` holds `lock_b` and waits for `lock_a`. Neither can move. The program freezes. It may not happen on every run, which is the danger.

The fix is the same order for everyone:

```python
import threading

lock_a = threading.Lock()
lock_b = threading.Lock()
log = []

def one():
    with lock_a:
        with lock_b:
            log.append("one")

def two():
    with lock_a:
        with lock_b:
            log.append("two")

threads = [threading.Thread(target=f) for f in (one, two)]
for t in threads:
    t.start()
for t in threads:
    t.join()
print(sorted(log))  # ['one', 'two']
```

Both take `lock_a` first. Whoever gets it first finishes, and the other follows. For many locks, sort them by a fixed key, such as an account number, and take them in that order.

A second protection is a timeout, so that a stuck thread can give up and report:

```python
import threading

lock = threading.Lock()
lock.acquire()
got_it = lock.acquire(timeout=0.1)
print(got_it)  # False
lock.release()
```

A timeout does not prevent the deadlock, but it turns an endless freeze into an error that you can log and fix.

Count the risk: with 3 locks that may be taken in any order, there are 6 possible orders, and a deadlock is possible whenever two threads use different ones. With a fixed order, there is only 1, and no circle.

About atomic operations. A single `list.append()` is atomic in CPython. `counter += 1` is not, because it reads, adds and writes in separate steps. Relying on atomic operations is fragile across versions, and free-threaded builds change the details. If correctness matters, use a lock.

| Rule | Why |
|---|---|
| Same lock order everywhere | No circle can form |
| Hold locks briefly | Less chance to meet |
| Use timeouts | A freeze becomes a visible error |
| Do not call unknown code while locked | It may take another lock |

## A common mistake {#mistake}
Taking locks in different orders in different functions, because each function looks correct alone. Document the order in one place. Another is trusting that an operation is atomic because it is one line of code.

## Key points {#key-points}
- A deadlock is a circle of threads each waiting for a lock another holds.
- Prevent it by taking locks in one fixed order.
- A timeout turns a silent freeze into an error.
- Do not depend on atomic operations unless you must.

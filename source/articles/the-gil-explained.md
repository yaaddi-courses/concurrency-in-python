# The GIL and why CPython has one

**Covers:** GIL, mutex, reference counting, bytecode, switch interval, setswitchinterval, JIT, ABI

## One thread at a time {#overview}
Python lets you start many threads, but in the standard implementation, only one of them can run Python code at any instant. The rule is enforced by a single lock, the GIL. Understanding it explains most surprises about threads in Python, such as why more threads rarely speed up a calculation.

## GIL {#gil}
GIL stands for Global Interpreter Lock. It is a lock inside CPython, the standard Python interpreter, that lets only one thread execute Python bytecode at a time. A thread must hold it to run, and gives it up so that another can have a turn. It protects the interpreter's own internal data from being damaged by two threads at once.

## Mutex {#mutex}
A mutex, short for mutual exclusion, is the general name for a lock that only one thread can hold at a time. The GIL is one mutex. Programs use many smaller ones to protect their own data.

## Reference counting {#reference-counting}
Reference counting is how CPython decides when to free an object's memory. Every object carries a count of how many names and containers point to it. When the count falls to zero, the memory is released at once. The count changes constantly, from every thread, so changing it unprotected could corrupt it. The GIL is the simple way CPython keeps this safe.

## Bytecode {#bytecode}
Bytecode is the lower-level set of instructions that Python source is compiled into before the interpreter executes it. The interpreter reads one bytecode instruction after another. The GIL is held while it runs them, and it is released between instructions, not in the middle of one.

## Switch interval {#switch-interval}
The switch interval is the timer that controls how often CPython considers handing the GIL to another waiting thread. The default is 5 milliseconds. When it passes, the running thread is asked to give up the lock at the next convenient point.

## setswitchinterval {#setswitchinterval}
`sys.setswitchinterval()` sets that time, in seconds. A smaller number makes threads take turns more often, which adds overhead. A larger one lets each thread run longer. Most programs never need to change it.

## JIT {#jit}
A JIT, a just-in-time compiler, turns code into machine code while the program runs. Recent versions of Python include an experimental one. It speeds up some code, but it does not remove the GIL.

## ABI {#abi}
The ABI, application binary interface, is the set of low-level rules that lets compiled code from different sources work together. Compiled extension modules are built against Python's ABI. Changing how threads work inside the interpreter can break that compatibility, which is one reason the GIL has stayed for so long.

## A worked example {#example}
Look at the interval and a simple count with two threads.

```python
import sys

print(sys.getswitchinterval())  # 0.005
```

The default is 0.005 seconds, 5 milliseconds. A thread that has been running for that long is asked to yield.

Now a CPU-bound task split over two threads. The work: count down from 2 million, once in a single thread, and then half and half in two.

```python
import threading
import time

def count(n):
    while n > 0:
        n -= 1

N = 2_000_000
start = time.perf_counter()
count(N)
single = time.perf_counter() - start

start = time.perf_counter()
t1 = threading.Thread(target=count, args=(N // 2,))
t2 = threading.Thread(target=count, args=(N // 2,))
t1.start(); t2.start()
t1.join(); t2.join()
double = time.perf_counter() - start
print(double > single * 0.5)  # True
```

With two cores and no GIL, you would hope for about half the time. The line prints `True` because two threads took more than half of the single-thread time. In practice, the two-thread run is about as slow as the single one, sometimes slower, since the threads take turns holding the GIL and the switching has a cost.

Why is I/O different? A thread that waits for a disk or a network releases the GIL while it waits. Another thread can use it. So for I/O-bound work, threads help, while for CPU-bound work they do not.

Count the effect: a calculation of 4 seconds split over 4 threads in standard Python still takes about 4 seconds, or a little more. The same calculation in 4 separate processes can take about 1 second on a 4-core machine.

| Question | Answer |
|---|---|
| What does the GIL protect? | Interpreter internals, such as reference counts |
| When is it released? | During waits, and every switch interval |
| Does it affect I/O-bound code? | Hardly |
| Does it affect CPU-bound code? | Yes, threads do not run calculations in parallel |

## A common mistake {#mistake}
Believing that the GIL makes your own code thread-safe. It does not. A thread can lose the GIL between two bytecode instructions, so a sequence like "read a number, add one, write it back" can still go wrong. You need your own locks, as later articles show. Another is changing the switch interval to "speed things up". It rarely does.

## Key points {#key-points}
- The GIL lets one thread run Python bytecode at a time.
- It exists to keep reference counts and other internals safe.
- Threads still help I/O-bound work, because waiting releases the GIL.
- The GIL does not protect your own data.

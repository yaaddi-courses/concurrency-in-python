# Getting around the GIL

**Covers:** C extension, free-threaded Python, NumPy, PEP, communicate()

## Real parallelism in Python {#overview}
The GIL stops threads from running Python code in parallel, but it does not stop you from using several cores. There are three ways around it, and a fourth that is on the way. Choosing among them is a practical decision about your task.

## C extension {#c-extension}
A C extension is a module written in C, not Python, that Python can call. The core of NumPy is one. Such code can release the GIL while it works, since it does not touch Python's internal data at that time. During that period, other threads can run Python code or other extensions, and truly in parallel.

## NumPy {#numpy}
NumPy is a library for fast numerical tables. Most of its heavy operations, such as multiplying large tables, run in compiled code and release the GIL. A program that mostly calls NumPy can therefore use several cores through ordinary threads.

## subprocess.run() {#subprocess-run}
`subprocess.run()` runs an external command, waits for it to finish and returns the result. Each command is a separate operating-system process, with its own interpreter and its own GIL. Several of them can run on different cores at once.

## communicate() {#communicate}
`communicate()` sends input to a running subprocess and reads its output until it ends. When you start processes yourself, it is the safe way to exchange text with them, because it reads and writes at the same time and avoids a stalled exchange.

## Free-threaded Python {#free-threaded-python}
Free-threaded Python is a build of CPython that can run without the GIL at all, so threads can run Python code in parallel. It is defined by PEP 703 and is available as an optional build in recent versions. Some extension modules need updating to work with it, and single-thread code can run a little slower.

## PEP {#pep}
A PEP, Python Enhancement Proposal, is a numbered design document for Python. PEP 703 proposed making the GIL optional. PEP 8 is the style guide. Reading the PEP behind a feature shows why it exists and what was rejected.

## A worked example {#example}
Four ways to use four cores for a CPU-bound task, from the simplest.

**1. Separate processes with `subprocess.run()`.**

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "-c", "print(sum(range(1000)))"],
    capture_output=True, text=True,
)
print(result.stdout.strip())  # 499500
print(result.returncode)      # 0
```

The child runs on its own. With four such commands started at once, four cores can work. `communicate()` is what you use when you start them with `subprocess.Popen` and need their output:

```python
import subprocess
import sys

p = subprocess.Popen(
    [sys.executable, "-c", "import sys; print(sys.stdin.read().upper())"],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE, text=True,
)
out, _ = p.communicate("hello")
print(out.strip())  # HELLO
```

It sent "hello" in, waited for the end and returned the result.

**2. A C extension, such as NumPy.** One thread calls a large table operation. While it runs inside compiled code, the GIL is released, so a second thread can start another operation on a second core. The speed-up depends on the operation, and for large tables it can approach the number of cores.

**3. Free-threaded Python.** Run the same threaded code on a free-threaded build, and the threads that were taking turns now run side by side. In the earlier article, two threads counting down took as long as one. On a free-threaded build, they can take close to half the time.

Count the difference for a 4-second calculation on 4 cores:

| Approach | Rough time | Cost |
|---|---|---|
| Plain threads | 4 s | None, but no gain |
| 4 subprocesses | 1 s to 1.3 s | Start-up and data copying |
| NumPy in threads | 1 s to 1.5 s | Must express the work as table operations |
| Free-threaded build | 1 s to 1.3 s | A special build, extension support |

Those times are only illustrations. What matters is the shape: only the last three can use all cores.

There is also a fourth, process pools, covered in later articles, which is the most common way in practice.

## A common mistake {#mistake}
Using threads for a pure-Python calculation and being surprised that nothing speeds up. Check whether the heavy work is inside a C extension, or move it into processes. Another is assuming that a free-threaded build makes data safe. Without the GIL, your own locks matter even more.

## Key points {#key-points}
- Separate processes each have their own GIL, so they run in parallel.
- C extensions such as NumPy release the GIL during heavy work.
- Free-threaded Python (PEP 703) removes the GIL, at some cost.
- `subprocess.run()` is simple, and `communicate()` exchanges text safely.

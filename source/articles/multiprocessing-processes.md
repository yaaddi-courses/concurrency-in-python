# Processes: starting, stopping and checking them

**Covers:** multiprocessing.Process, exitcode, terminate(), process.terminate, kill(), get_context(), __main__ guard, __name__

## A separate program with its own memory {#overview}
A thread shares memory with the others, and shares the GIL. A process is a whole separate program: its own memory, its own interpreter and its own GIL. That is why processes give real parallelism for calculation. The price is that they cannot share variables, and starting one costs more than starting a thread.

## multiprocessing.Process {#multiprocessing-process}
`multiprocessing.Process` is the class that creates a new process. Its interface copies `threading.Thread`: give it a `target` function and `args`, then call `start()` and `join()`. The function runs in a different process, so changes it makes to a normal variable are not seen by the parent.

## exitcode {#exitcode}
`exitcode` is an attribute that tells how a process ended. It is `None` while the process is still running, 0 after a normal finish, a positive number if the code stopped with an error number, and a negative number such as -15 if a signal ended it.

## terminate() {#terminate}
`terminate()` stops a process by sending it the terminate signal. It is a polite ask that the operating system delivers at once. The process gets no chance to clean up, so files it was writing can be left half done.

## process.terminate {#process-terminate}
`process.terminate` is the same method seen as an attribute of a process object, written in full as `p.terminate()`. Use it for a worker that must be stopped from outside, for example after a timeout. A better design asks the worker to stop on its own, as a later article shows.

## kill() {#kill}
`kill()` stops a process at once by sending SIGKILL. Unlike the terminate signal, it cannot be caught or ignored. Use it only when `terminate()` did not work.

## get_context() {#get-context}
`get_context()` returns a context object that uses a chosen way of starting processes, such as `"spawn"`. You then create processes, queues and locks from that context, and they all use the same method. It lets one part of a program choose a start method without changing the whole program.

## __name__ {#name}
`__name__` is the built-in variable that holds the name of the current module. It is `"__main__"` when the file is run directly, and the file's own name when it is imported.

## __main__ guard {#main-guard}
The `__main__` guard is the line `if __name__ == "__main__":`, placed around the code that starts processes. It matters here because a new process may import the script again, and without the guard, each child would start more children.

## A worked example {#example}
A worker process that exits cleanly, and one that is ended from outside.

```text
import multiprocessing as mp
import time

def quick():
    return

def slow():
    time.sleep(30)

if __name__ == "__main__":
    p = mp.Process(target=quick)
    print(p.exitcode)  # None
    p.start()
    p.join()
    print(p.exitcode)  # 0

    q = mp.Process(target=slow)
    q.start()
    q.terminate()
    q.join()
    print(q.exitcode != 0)  # True
```

Before it starts, `p.exitcode` is `None`. After a normal end, it is 0. The second process would sleep for 30 seconds, but `terminate()` sent the terminate signal, so it ended at once. On Linux and macOS the code is -15, the negative of the signal number, and on Windows it is 1. The comparison with 0 is true on every system.

The same job through a context:

```text
import multiprocessing as mp

def show():
    return

if __name__ == "__main__":
    ctx = mp.get_context("spawn")
    p = ctx.Process(target=show)
    p.start()
    p.join()
    print(p.exitcode)  # 0
```

The guard matters. Without `if __name__ == "__main__":`, a script that starts a process with the spawn method imports itself in the child, runs the start line again, and Python stops with a `RuntimeError` telling you to use the guard.

Count the cost of a process against a thread. Starting a thread takes about 0.1 milliseconds. Starting a process can take 10 to 100 milliseconds, depending on the system and the start method. So a process pays off only for work that takes noticeably longer than that, such as a calculation of a second or more.

| Call | Signal | Can the process clean up? |
|---|---|---|
| `terminate()` | terminate signal | No time to, but it can catch the signal |
| `kill()` | SIGKILL | No, never |

## A common mistake {#mistake}
Forgetting the guard and getting an error, or in the worst case an endless chain of new processes. Another is calling `kill()` first, which leaves no chance for cleanup, when `terminate()` would do.

## Key points {#key-points}
- A process has its own memory and its own GIL, so it runs in parallel.
- `exitcode` is `None` while running, 0 on success, and negative if a signal ended it.
- `terminate()` sends the terminate signal and `kill()` sends SIGKILL.
- Always put the starting code under `if __name__ == "__main__":`.

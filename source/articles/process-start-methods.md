# How a new process begins: fork, spawn and forkserver

**Covers:** fork, forking, spawn, spawning, forkserver, copy-on-write, IPC, RPC

## Three ways to make a process {#overview}
Creating a process is not one operation. The operating system offers different ways, and Python lets you choose between them. The choice affects speed, safety and what the child process starts with, so it is worth knowing what each one does.

## Fork {#fork}
Fork makes the child by copying the parent process as it is at that moment: its memory, its open files, its variables. The child continues from the same point in the code. It is fast. It is available on Linux and macOS, not on Windows. It is also risky: copying a program that has several threads, or that holds locks, can leave the child with a lock that nobody will ever release.

## Forking {#forking}
Forking is the act of using that method, and the word is used for the whole mechanism: "forking is fast but unsafe with threads". Python 3.14 changed the default on Linux away from fork for this reason.

## Copy-on-write {#copy-on-write}
Copy-on-write is how the system makes fork cheap. The child does not get a real copy of memory at once. It shares the parent's pages, and a page is copied only at the moment that one side writes to it. A child that mostly reads does not copy much.

## Spawn {#spawn}
Spawn starts a fresh Python interpreter for the child. The child imports your module from the start and receives only what you pass it. It is slower to start, but clean and predictable, and it is the only method on Windows, and the default on macOS.

## Spawning {#spawning}
Spawning is the act of starting a process this way. A consequence: everything passed to the child must be picklable, which means convertible to bytes, because it has to be sent across.

## Forkserver {#forkserver}
Forkserver is a middle way. A small server process is started once, early, while the program is still simple. Each new child is forked from that server and not from the busy main program. It avoids the thread problem of fork and is faster than starting a fresh interpreter every time. It is available on Unix systems.

## IPC {#ipc}
IPC, inter-process communication, is the general name for ways that separate processes exchange data: queues, pipes, shared memory and sockets. Since processes do not share variables, every exchange must go through one of them.

## RPC {#rpc}
RPC, remote procedure call, is a style of IPC in which one process calls a function that actually runs in another process, or on another machine, and gets the result back. Managers in the multiprocessing module work in this way.

## A worked example {#example}
Check the methods your system offers and the default:

```text
import multiprocessing as mp

if __name__ == "__main__":
    print("spawn" in mp.get_all_start_methods())  # True
    print(mp.get_start_method() in mp.get_all_start_methods())  # True
```

Spawn exists everywhere. The default is one of the available methods, and which one depends on the system.

What does the child receive? With spawn, only what you pass:

```text
import multiprocessing as mp

counter = 0

def read_counter(out):
    out.put(counter)

if __name__ == "__main__":
    counter = 5
    ctx = mp.get_context("spawn")
    out = ctx.Queue()
    p = ctx.Process(target=read_counter, args=(out,))
    p.start()
    print(out.get())  # 0
    p.join()
```

The parent set `counter` to 5 in the guarded block. The spawned child imported the module fresh, where `counter` is 0, and reported 0. With fork, the child would have copied the parent's memory and reported 5. This difference is the main surprise when moving code from Linux to Windows.

Count the start cost. Forking a program that uses 500 MB of memory is cheap, a few milliseconds, thanks to copy-on-write. Spawning needs a new interpreter and imports, say 50 to 200 milliseconds. For 100 short jobs, that is 5 to 20 seconds of start-up with spawn, against under half a second with fork. This is why pools reuse their processes, instead of starting one per job.

| Method | Speed | Safety with threads | Where |
|---|---|---|---|
| fork | Fastest | Risky | Linux, macOS |
| spawn | Slowest | Safe | All systems |
| forkserver | Medium | Safe | Unix |

## A common mistake {#mistake}
Writing code that works only with fork, because it uses a global variable the child inherits. It then breaks on Windows. Pass data explicitly. Another is passing something that cannot be pickled, such as a lock held in a lambda.

## Key points {#key-points}
- Fork copies the parent, spawn starts fresh, forkserver forks from a clean helper.
- Copy-on-write makes fork cheap.
- Choose spawn for portability and safety, and pass data in explicitly.
- Processes exchange data through IPC, since they share no variables.

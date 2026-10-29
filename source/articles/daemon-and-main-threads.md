# Main, worker and daemon threads

**Covers:** run(), RuntimeError, daemon thread, main thread, threading.main_thread(), sys.exit(), os._exit(), signal.signal()

## Who keeps the program alive {#overview}
When a Python program starts, there is already one thread running. Every thread you create is another. Whether the program stays alive depends on which kind of thread is still running, and on how the program is asked to stop.

## Main thread {#main-thread}
The main thread is the thread that runs your script's top-level code when the program starts. By default, Python waits for all other threads to finish before the whole program exits, even after the main thread has reached the end of the script.

## threading.main_thread() {#threading-main-thread}
`threading.main_thread()` returns the `Thread` object of the main thread. Comparing it with the current thread tells code whether it is running in the main thread or in a worker.

## Daemon thread {#daemon-thread}
A daemon thread is a thread marked with `daemon=True`. Python does not wait for it. The moment the program is about to exit, daemon threads are stopped, even in the middle of their work. They suit background jobs that may be dropped safely, such as a refresher that updates a cache. They do not suit anything that must finish, such as writing a file.

## run() {#run}
`run()` is the method that holds the code a thread executes. `start()` creates the new thread and calls `run()` in it for you. If you call `run()` yourself, it executes in the current thread, with no concurrency at all. You can also write a subclass of `Thread` and override `run()`.

## RuntimeError {#runtimeerror}
`RuntimeError` is the exception Python raises for an error that fits no other category. In threads, you meet it when you call `start()` twice on a thread, or set `daemon` after the thread has started.

## sys.exit() {#sys-exit}
`sys.exit()` ends the program by raising `SystemExit`. In a worker thread, it ends only that thread, and the rest of the program continues. It is not a way to stop the whole program from a worker.

## os._exit() {#os-exit}
`os._exit()` ends the process at once, without running cleanup handlers or waiting for any thread. It is a last resort, for example in a child process after a failure.

## signal.signal() {#signal-signal}
`signal.signal()` registers a function to run when the process receives a signal, such as the interrupt from pressing Ctrl+C. It can be called only from the main thread, and handlers run in the main thread too.

## A worked example {#example}
The difference between a normal and a daemon thread:

```python
import threading
import time

def background():
    time.sleep(0.3)
    print("background finished")

t = threading.Thread(target=background, daemon=True)
t.start()
print(t.daemon)  # True
print(threading.current_thread() is threading.main_thread())  # True
```

The script prints the two lines and exits at once. The 0.3-second wait never finishes, and "background finished" is never printed, because the daemon is stopped at exit. Remove `daemon=True`, and Python waits, then prints "background finished" too.

Work that must complete should be a normal thread, or a daemon that you `join()`:

```python
import threading

done = []

def work():
    done.append("saved")

t = threading.Thread(target=work, daemon=True)
t.start()
t.join()
print(done)  # ['saved']
```

With `join()`, the daemon finishes before the program goes on.

Calling `run()` directly:

```python
import threading

def show():
    print(threading.current_thread().name)

t = threading.Thread(target=show, name="worker-1")
t.run()
t.start()
t.join()
# MainThread
# worker-1
```

`t.run()` ran in the main thread, so it printed `MainThread`. `t.start()` ran the same code in the new thread, so it printed `worker-1`.

Setting daemon too late:

```python
import threading
import time

t = threading.Thread(target=lambda: time.sleep(0.1))
t.start()
try:
    t.daemon = True
except RuntimeError:
    print("daemon must be set before start")  # daemon must be set before start
t.join()
```

Exiting. `sys.exit()` in a worker ends only that worker. To end a whole program from anywhere, `os._exit(0)` does it immediately, but skips cleanup, so open files may not be flushed.

| Way to end | Effect |
|---|---|
| End of script | Waits for non-daemon threads |
| `sys.exit()` in the main thread | Exits after the others finish |
| `sys.exit()` in a worker | Ends that thread only |
| `os._exit()` | Ends everything at once, no cleanup |

## A common mistake {#mistake}
Making a thread daemon to avoid waiting for it, when it writes data. The program can exit in the middle of a write. Another is calling `thread.run()` and believing it started a new thread.

## Key points {#key-points}
- Python waits for non-daemon threads at exit, but not for daemon threads.
- `start()` runs `run()` in a new thread. Calling `run()` yourself does not.
- Set `daemon=True` before `start()`.
- `sys.exit()` in a worker ends only that thread. `os._exit()` ends the process abruptly.

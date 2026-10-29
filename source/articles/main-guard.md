# The main guard, in depth

**Covers:** __main__

## Why one line decides whether a script works {#overview}
The guard `if __name__ == "__main__":` was introduced alongside processes. This article looks at what the name `__main__` really is and shows the failure you get without it. It is a short topic, but the cause of many confusing bugs on Windows and macOS.

## __main__ {#main}
`__main__` is the name Python gives to the module that is run directly. When you start `python script.py`, the code of `script.py` runs as a module whose `__name__` is the text `"__main__"`. When another file imports it, the same code is a module with the file's own name. The guard compares `__name__` with `"__main__"` to tell the two cases apart.

The same name appears in two other places. A package can contain a file `__main__.py`, which runs when you start the package with `python -m package`. And inside a spawned child, the module that started the parent is imported under the name `__mp_main__`, not `__main__`, which is how the guard stays false in the child.

## A worked example {#example}
A file whose top level starts a process, with no guard:

```text
# broken.py
import multiprocessing as mp

def work():
    print("working")

p = mp.Process(target=work)
p.start()
p.join()
```

Run with the spawn method (the only one on Windows), the child begins by importing `broken.py`. That executes the last three lines again in the child, which tries to start another process. Python detects this and stops with:

```text
RuntimeError: An attempt has been made to start a new process before the
current process has finished its bootstrapping phase.
```

The fix is to put the start-up lines under the guard:

```text
import multiprocessing as mp

def work():
    print("working")

if __name__ == "__main__":
    p = mp.Process(target=work)
    p.start()
    p.join()
# working
```

In the parent, `__name__` is `"__main__"`, so the block runs. In the spawned child, the module is imported under another name, so the block is skipped, and the child only defines `work`. The check:

```python
print(__name__ == "__main__")  # True
```

When this snippet runs as a script, it prints `True`. When it is imported, `__name__` is the file's name, and the same expression is `False`.

Count the explosion that the guard prevents. If every child started two more children, 1 process would lead to 2, then 4, then 8: after 10 levels, 1,024 processes. Python stops this early with the error above, but on older versions and with other tools, a missing guard has frozen machines.

A practical layout for any program that uses processes:

```text
def main():
    ...            # all the real work

if __name__ == "__main__":
    main()
```

Keeping the work in `main()` makes the file easy to import for tests too, since importing it starts nothing.

Tests are another reason for the habit. A test file imports your module to call its functions. If the module starts a long job at the top level, merely importing it would run that job, and every test run would pay for it. With the guard, importing is harmless, and only running the file starts the work. The cost is one extra line and one level of indentation, which is small against the surprises it prevents.

| Situation | `__name__` | Guard block runs? |
|---|---|---|
| `python script.py` | `"__main__"` | Yes |
| `import script` | `"script"` | No |
| Spawned child | `"__mp_main__"` | No |

## A common mistake {#mistake}
Putting the guard around only part of the start-up, and leaving a function call outside it. Anything at the top level runs in every child. Another is testing only on Linux, where fork hides the problem, and finding the error when the program first runs on Windows.

## Key points {#key-points}
- `__main__` is the name of the module that was run directly.
- Without the guard, a spawned child re-runs the start-up code.
- Put process creation inside `if __name__ == "__main__":`.
- Keep the real work in a `main()` function.

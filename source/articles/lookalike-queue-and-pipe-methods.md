# Queue, pipe and shared-value names that only look real

**Covers:** .push(), .receive(), q.peek(), .read_all(), .flush(), .commit(), .check()

## Names borrowed from other tools {#overview}
Queues, pipes and futures all pass work or results between workers, so many programmers expect them to look like a stack, a socket or a database. They do not. Each of the names below is a plausible guess, and each is wrong. The real names are short and few. Learn them as pairs and the wrong answers stop looking tempting.

## .push() {#push}
`.push()` is the word for a stack, where you push and pop. A queue is different: you add with `put()` and remove with `get()`. The order is first in, first out, so the first item put in is the first one taken out.

## .receive() {#receive}
`.receive()` is not a method of a queue. It is also not the name used by a pipe. A pipe connection sends with `send()` and receives with `recv()`. A queue adds with `put()` and removes with `get()`. The two pairs are easy to mix up, so keep them apart: queue means put and get, pipe means send and recv.

## q.peek() {#peek}
`q.peek()` would look at the next item without removing it. A Python queue has no such method, and for a good reason: in a program with several workers, the item you peeked at may be taken by another worker before you act on it. What exists is `get_nowait()`, which takes an item at once or raises `queue.Empty` if there is none. Compare it with `get()`, which waits.

## .read_all() {#read-all}
`.read_all()` does not exist on a pipe connection. A connection reads one object at a time with `recv()`. If you want everything, you call `recv()` in a loop. It is also worth knowing that `recv()` waits if nothing has arrived yet. `poll()` tells you whether something is waiting.

## .flush() {#flush}
`.flush()` appears as a wrong answer when the question is how to change a shared value. A shared `Value` is changed by assigning to its `value` attribute, inside the lock that protects it. Flushing is something you do to a file or a stream, to push buffered bytes out. It has no meaning for a shared number.

## .commit() {#commit}
`.commit()` is a database word. A database connection commits a transaction. A shared `Value` has no transaction and nothing to commit. If another process should see a change, assign to `value` while holding the lock.

## .check() {#check}
`.check()` is not a method of a future. To test a future without waiting, use `done()`. It returns `True` when the work has finished, with a result or an error. `result()` is different: it waits, and then gives the result or raises the error that happened in the worker.

## A worked example {#example}
First the queue. It returns items in the order they went in.

```python
import queue

q = queue.Queue()
q.put("a")
q.put("b")
print(q.get())  # a
print(q.get_nowait())  # b
try:
    q.get_nowait()
except queue.Empty:
    print("empty")  # empty
print(hasattr(q, "push"), hasattr(q, "peek"), hasattr(q, "receive"))  # False False False
```

The first item in came out first. When the queue was empty, `get_nowait()` raised at once instead of waiting, and the invented names are missing.

Now a pipe, where the pair of names is different.

```python
import multiprocessing

a, b = multiprocessing.Pipe()
a.send({"n": 1})
a.send("done")
print(b.poll())  # True
print(b.recv())  # {'n': 1}
print(b.recv())  # done
print(b.poll())  # False
print(hasattr(b, "read_all"), hasattr(b, "receive"))  # False False
```

Two objects were sent, and two calls to `recv()` brought them back one at a time. There is no way to take both in one call.

A shared value changed under its lock:

```python
import multiprocessing

counter = multiprocessing.Value("i", 0)
with counter.get_lock():
    counter.value += 1
print(counter.value)  # 1
print(hasattr(counter, "flush"), hasattr(counter, "commit"))  # False False
```

The update happens inside the `with` block, which is what makes it safe when several processes do it together.

Finally a future.

```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=1) as pool:
    future = pool.submit(sum, [1, 2, 3])
    print(future.result())  # 6
    print(future.done())  # True
    print(hasattr(future, "check"))  # False
```

After `result()` returned, `done()` is certainly true. Before that, `done()` can be used for a quick look that does not block.

## A common mistake {#mistake}
A typical bug is to treat a queue as a stack and call `push()` or `pop()`, which fail with an `AttributeError`. The harder mistake is to want a peek: code that looks at the front of a shared queue and then acts on what it saw is a race condition, because another worker may take the item in between. Take the item with `get()` or `get_nowait()` and decide afterwards what to do with it.

## Key points {#key-points}
- A queue uses `put()` and `get()`. `get_nowait()` raises `queue.Empty` instead of waiting.
- A pipe connection uses `send()` and `recv()`, one object per call, and `poll()` to look.
- A shared `Value` is updated inside its lock. It has no flush or commit.
- A future is tested with `done()`, and its result is read with `result()`.
- `push`, `receive`, `peek`, `read_all`, `flush`, `commit` and `check` do not exist on these objects.

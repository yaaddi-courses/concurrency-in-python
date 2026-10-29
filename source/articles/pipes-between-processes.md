# Pipes: a direct line between two processes

**Covers:** Pipe, multiprocessing.Pipe, send(), recv(), duplex, simplex

## When two processes need to talk to each other {#overview}
A queue suits many workers and one collector. When exactly two processes need a private line between them, a pipe is lighter and faster. It has two ends, and each end belongs to one process. Whatever one end sends, the other end receives.

## Pipe {#pipe}
A pipe is a two-ended connection between processes. In Python, `Pipe()` creates it and returns the two ends, as a pair of connection objects. You keep one and give the other to the second process. Like a queue, it carries picklable objects, so what arrives is a copy.

## multiprocessing.Pipe {#multiprocessing-pipe}
`multiprocessing.Pipe` is the function from the multiprocessing module that creates the pair of connected ends. Called with no argument, it makes a two-way pipe. Called as `Pipe(duplex=False)`, it makes a one-way pipe.

## send() {#send}
`send()` sends one object through the connection. It returns when the object has been handed to the system, which may be before the other side has read it.

## recv() {#recv}
`recv()` receives one object from the connection, waiting until one arrives. If the other end has been closed and nothing is left, it raises `EOFError`. To check without waiting, call `poll()`, which returns `True` if something is ready to be read.

## Duplex {#duplex}
A duplex pipe is a two-way pipe: both ends can send and receive. It is the default. A single pipe is then enough for a question and its answer.

## Simplex {#simplex}
A simplex pipe is one-way: one end only sends, and the other only receives. It is made with `Pipe(duplex=False)`. It is the clearer choice when data flows in one direction only, since a mistake in direction is caught at once.

## A worked example {#example}
A parent sends a number, and a child answers with its square.

```text
import multiprocessing as mp

def worker(conn):
    n = conn.recv()
    conn.send(n * n)
    conn.close()

if __name__ == "__main__":
    parent_end, child_end = mp.Pipe()
    p = mp.Process(target=worker, args=(child_end,))
    p.start()
    parent_end.send(7)
    print(parent_end.recv())  # 49
    p.join()
```

`Pipe()` returned two ends. The child got one, the parent kept the other. The parent sent 7, the child received it, computed 49 and sent it back over the same pipe, because it is duplex.

A one-way pipe, where the child can only send:

```text
import multiprocessing as mp

def producer(out):
    for i in range(3):
        out.send(i)
    out.close()

if __name__ == "__main__":
    receiver, sender = mp.Pipe(duplex=False)
    p = mp.Process(target=producer, args=(sender,))
    p.start()
    sender.close()
    items = []
    while True:
        try:
            items.append(receiver.recv())
        except EOFError:
            break
    p.join()
    print(items)  # [0, 1, 2]
```

With `duplex=False`, the first end returned can only receive, and the second can only send. The parent closes its copy of the sending end. That matters: the receiver sees `EOFError` only when every copy of the sending end is closed, so a forgotten copy in the parent would make `recv()` wait for ever.

Compare a pipe with a queue for speed. A pipe has no lock and no background thread, so sending 100,000 small messages can take roughly half the time of a queue. A queue, though, lets any number of processes put and get, while a pipe is only safe when one process uses each end. Two processes reading from the same end at the same time can corrupt the messages.

| | Pipe | Queue |
|---|---|---|
| Endpoints | Exactly two | Any number |
| Speed | Faster | Slower, has locking |
| Direction | Two-way or one-way | Any direction, shared |
| End of data | `EOFError` on `recv()` | Needs a stop marker |

## A common mistake {#mistake}
Leaving a copy of the sending end open in the parent and waiting for `EOFError` that never comes. Close the ends you do not use. Another is having two processes receive from the same end.

## Key points {#key-points}
- `Pipe()` returns two connected ends. `send()` and `recv()` move picklable objects.
- By default a pipe is duplex (two-way). `Pipe(duplex=False)` makes a simplex (one-way) pipe.
- `recv()` raises `EOFError` once every sending end is closed.
- Use a pipe for exactly two processes, and a queue for many.

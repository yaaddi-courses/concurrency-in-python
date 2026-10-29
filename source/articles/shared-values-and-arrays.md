# Shared values and arrays

**Covers:** Value, multiprocessing.Value, Array, multiprocessing.Array

## Sharing one number between processes {#overview}
Queues and pipes move copies. Sometimes you want several processes to read and update the same number, such as a progress counter. The multiprocessing module has a small shared block of memory for that, with a fixed type, so the processes can all see the same bytes.

## Value {#value}
A `Value` is a single shared number or character kept in memory that all the processes can see. You create it with a type code, such as `"i"` for a whole number or `"d"` for a decimal, and an initial number. You read and write it through its `.value` attribute.

## multiprocessing.Value {#multiprocessing-value}
`multiprocessing.Value` is the full name of that function. By default it comes with its own lock, reachable as `.get_lock()`. Passing `lock=False` gives a raw value without a lock, which is faster but unsafe for updates.

## Array {#array}
An `Array` is a fixed-length shared list of numbers, all of the same type. You give a type code and either a length or the starting items. Items are read and written by position, like a list, but you cannot add or remove items.

## multiprocessing.Array {#multiprocessing-array}
`multiprocessing.Array` is the full name of that function. Like `Value`, it has a lock by default. Slices of it can be read as a whole with `arr[:]`.

## A worked example {#example}
Four processes add to one shared counter.

```text
import multiprocessing as mp

def add(counter):
    for _ in range(1000):
        with counter.get_lock():
            counter.value += 1

if __name__ == "__main__":
    counter = mp.Value("i", 0)
    procs = [mp.Process(target=add, args=(counter,)) for _ in range(4)]
    for p in procs:
        p.start()
    for p in procs:
        p.join()
    print(counter.value)  # 4000
```

The type code `"i"` means a signed integer, and the start is 0. The `with counter.get_lock():` line is needed. `counter.value += 1` is a read, an add and a write, the same pattern that causes races between threads. Without the lock, the result would often be below 4000.

A shared array, where each process fills its own slot:

```text
import multiprocessing as mp

def fill(arr, i):
    arr[i] = i * 10

if __name__ == "__main__":
    arr = mp.Array("i", 4)
    procs = [mp.Process(target=fill, args=(arr, i)) for i in range(4)]
    for p in procs:
        p.start()
    for p in procs:
        p.join()
    print(arr[:])  # [0, 10, 20, 30]
```

Each process writes a different position, so no two of them touch the same slot, and no lock is needed for this particular pattern. `arr[:]` reads all four items as a normal list.

The cost of sharing. Reading `counter.value` is much faster than getting a number through a queue, since there is no pickling and no message. But there is a limit: only simple C types fit. A shared dictionary or an object with several fields cannot be placed in a `Value`. For those, use a manager, described in the next article.

Sizes: an `Array("i", 1000)` holds 1,000 integers of 4 bytes each, so 4,000 bytes in shared memory. The array has a fixed length, so you must decide the size when you create it.

| Type code | Meaning | Size |
|---|---|---|
| `"i"` | Signed integer | 4 bytes |
| `"d"` | Decimal number | 8 bytes |
| `"c"` | One character | 1 byte |

## A common mistake {#mistake}
Updating `counter.value` without holding its lock and losing updates. Another is trying to store a list or a dictionary in a `Value`, which fails because only simple C types are allowed.

## Key points {#key-points}
- A `Value` is one shared number and an `Array` is a fixed-length list of numbers.
- Both have a lock by default. Use `with obj.get_lock():` for read-modify-write.
- Only simple C types fit.
- No pickling is needed, so access is fast.

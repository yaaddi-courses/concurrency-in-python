# Shared memory blocks

**Covers:** shared_memory, SharedMemory, resource tracker, unlink(), close()

## Sharing large data without copying it {#overview}
Queues and managers copy data, which is slow for big tables of numbers. A shared memory block gives several processes access to the very same piece of memory, with no copy at all. A video frame, an image or a large table can be passed to a worker in microseconds, whatever its size.

## shared_memory {#shared-memory}
`multiprocessing.shared_memory` is the module that provides shared memory blocks. A block is a named piece of memory reserved by the operating system. Any process that knows the name can attach to it and read or write its bytes.

## SharedMemory {#sharedmemory}
`SharedMemory` is the class in that module. `SharedMemory(create=True, size=n)` makes a new block of n bytes and gives it a random name. `SharedMemory(name=...)` attaches to an existing one. The bytes are available as the `.buf` attribute, which works like a `bytearray`.

## close() {#close}
`close()` releases this process's access to the block. The memory remains for other processes that still use it. Every process that attached to a block should call `close()` when it is done.

## unlink() {#unlink}
`unlink()` deletes the block once no process needs it. It should be called exactly once, by the process that created it, after the others have closed their access. If nobody unlinks, the memory stays reserved until the system restarts, on some systems.

## Resource tracker {#resource-tracker}
The resource tracker is a helper process that Python starts to keep an eye on shared resources such as these blocks. If a program ends without unlinking a block, the tracker deletes it and prints a warning about leaked resources. It is a safety net, and the warning means that some `unlink()` call was missing.

## A worked example {#example}
A parent creates a block, a child changes it, and the parent reads the change.

```text
from multiprocessing import Process, shared_memory

def child(name):
    shm = shared_memory.SharedMemory(name=name)
    shm.buf[0] = 99
    shm.close()

if __name__ == "__main__":
    shm = shared_memory.SharedMemory(create=True, size=4)
    shm.buf[0] = 1
    p = Process(target=child, args=(shm.name,))
    p.start()
    p.join()
    print(shm.buf[0])  # 99
    shm.close()
    shm.unlink()
```

The parent set the first byte to 1. It passed only the block's name, a short string, to the child. The child attached by name, wrote 99 and closed its access. After the join, the parent saw 99. Nothing was copied: both processes used the same memory.

The order of cleanup matters. Each process calls `close()`. Only the creator calls `unlink()`, and only after the other processes are finished.

Compare costs for a 100 MB table. Sending it through a queue means converting 100 MB to bytes, pushing it, and rebuilding it: roughly 0.2 to 0.5 seconds each way. Putting it in shared memory takes one copy in, perhaps 50 milliseconds, and each worker attaches in well under a millisecond. For 8 workers, a queue means 8 copies, about 800 MB of memory, while shared memory holds just the 100 MB once.

There are limits. A shared block is raw bytes: you must decide the layout yourself, or view it through NumPy. There is no locking, so two processes writing to the same byte at the same time can interfere. Combine it with a lock, or give each process its own part of the block.

If you forget `unlink()` in the example above, Python prints on exit a warning like `resource_tracker: There appear to be 1 leaked shared_memory objects`. The tracker cleans up, but treat the warning as a bug in your code.

| Call | Who calls it | When |
|---|---|---|
| `SharedMemory(create=True, size=n)` | Creator | Start |
| `SharedMemory(name=...)` | Other processes | To attach |
| `close()` | Every process | When finished |
| `unlink()` | Creator only | After all others closed |

## A common mistake {#mistake}
Calling `unlink()` in a child, which deletes the block while the parent still uses it. Another is forgetting it altogether and leaking the memory. A third is writing to the same bytes from two processes without any lock.

## Key points {#key-points}
- Shared memory lets processes use the same bytes with no copy.
- Pass the block's name, not its content.
- Every process calls `close()`. The creator alone calls `unlink()`.
- The resource tracker warns about blocks that were never unlinked.

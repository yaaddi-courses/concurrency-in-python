# Managers and proxies: shared dictionaries and lists

**Covers:** Manager, multiprocessing.Manager, proxy, proxies, manager.dict(), manager.list()

## Sharing real Python objects {#overview}
A shared `Value` holds one number, and an `Array` a fixed list of numbers. Often you want something richer: a dictionary of results, or a list that grows. A manager makes that possible by keeping the real object in one server process and letting the others talk to it.

## Manager {#manager}
A `Manager` is a server process that holds Python objects such as lists, dictionaries and locks, and lets other processes use them. The other processes do not hold the object. They hold a stand-in and send each operation to the manager, which carries it out.

## multiprocessing.Manager {#multiprocessing-manager}
`multiprocessing.Manager` is the function that starts a manager. It is usually used in a `with` block, so that the server process is stopped when the block ends. From it you create shared objects.

## Proxy {#proxy}
A proxy is the stand-in object that each process holds. It looks like a real dictionary or list: you read and write it in the same way. Each operation is sent to the manager as a message, and the result comes back. Proxies are picklable, so you can pass them to other processes as arguments.

## Proxies {#proxies}
Proxies, the plural, are all the stand-in objects that the different processes hold for the same shared object. Four workers share one dictionary through four proxies, and all of them talk to the one real dictionary in the manager.

## manager.dict() {#manager-dict}
`manager.dict()` gives you a dictionary that lives in the manager. After `d["a"] = 1` in one process, every other process that holds the proxy sees the key.

## manager.list() {#manager-list}
`manager.list()` gives you a list that lives in the manager. When one process calls `append()` on its proxy, the new item appears for all of them.

## A worked example {#example}
Four workers store results in one shared dictionary.

```text
import multiprocessing as mp

def work(shared, n):
    shared[n] = n * n

if __name__ == "__main__":
    with mp.Manager() as manager:
        shared = manager.dict()
        procs = [mp.Process(target=work, args=(shared, n)) for n in range(4)]
        for p in procs:
            p.start()
        for p in procs:
            p.join()
        print(dict(shared))  # {0: 0, 1: 1, 2: 4, 3: 9}
```

Each worker sets its own key. The manager receives four messages and updates the real dictionary. After the join, `dict(shared)` copies the content into a normal dictionary. A normal dictionary shows its keys in the order they were added, so the printed order here depends on which worker finished first. Sort the keys if you need a fixed order.

A shared list:

```text
import multiprocessing as mp

def add(items, n):
    items.append(n)

if __name__ == "__main__":
    with mp.Manager() as manager:
        items = manager.list()
        procs = [mp.Process(target=add, args=(items, n)) for n in range(3)]
        for p in procs:
            p.start()
        for p in procs:
            p.join()
        print(sorted(items))  # [0, 1, 2]
```

One catch: a proxy only sees changes made through itself. If the dictionary holds a list, and you change the inner list in place, the manager does not notice. Write the new list back with `shared["key"] = new_list`.

```text
import multiprocessing as mp

if __name__ == "__main__":
    with mp.Manager() as manager:
        d = manager.dict()
        d["items"] = []
        d["items"].append(1)
        print(d["items"])  # []
        d["items"] = [1]
        print(d["items"])  # [1]
```

The first `append` changed a temporary copy, so the shared list stayed empty. Assigning the whole list back worked.

The cost of a manager: every operation is a message to another process, and takes perhaps 50 to 100 microseconds. A loop with a million updates takes about a minute. The same loop on a plain dictionary takes a fraction of a second. So update a local object in the worker, and send the result once.

| Tool | Holds | Speed | Flexibility |
|---|---|---|---|
| `Value`, `Array` | Simple C types | Fast | Low |
| `manager.dict()`, `manager.list()` | Any picklable object | Slow, a message each time | High |
| Queue | Messages | Medium | Medium |

## A common mistake {#mistake}
Changing a nested object in place and expecting the change to be shared. Assign it back. Another is calling a manager object a million times in a loop, instead of collecting results locally and sending them once.

## Key points {#key-points}
- A manager keeps real objects in a server process. Workers hold proxies.
- `manager.dict()` and `manager.list()` give shared dictionaries and lists.
- A change to a nested object is not seen unless you assign it back.
- Every operation is a message, so keep the number of operations low.

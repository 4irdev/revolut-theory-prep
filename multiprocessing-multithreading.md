# Multiprocessing / Multithreading

## Threading vs Multiprocessing — the GIL context

CPython has a **GIL** (Global Interpreter Lock): only one thread executes Python bytecode at any moment. So:

- **Multithreading is real parallelism only for I/O.** When a thread blocks on a socket / file / `sleep`, it releases the GIL → other threads run. Great for waiting on networks, DBs, disks.
- **Multithreading does NOT speed up CPU-bound code** in CPython. Threads take turns; total wall time is about the same as single-threaded.
- **Multiprocessing IS real parallelism for CPU.** Each process has its own interpreter and its own GIL → multiple cores actually run Python in parallel.

| | Multithreading | Multiprocessing |
| --- | --- | --- |
| Memory | **Shared** (same address space) | **Separate** (each process has its own) |
| Communication | Direct variable access | IPC: `Queue`, `Pipe`, shared memory |
| Spawn cost | Cheap (~µs) | Expensive (~ms, fork/spawn copies state) |
| Best for | I/O-bound (HTTP, DB, files) | CPU-bound (compute, parsing, ML) |
| GIL impact | Bound by GIL | Each process has own GIL → no contention |
| Crash isolation | One bad thread takes the process | One bad process leaves others alive |
| Debugging | Harder (shared state races) | Easier (isolated state) |

**Rule of thumb:** I/O? Threads or `asyncio`. CPU? Processes. Mixed? `concurrent.futures` lets you swap `ThreadPoolExecutor` ↔ `ProcessPoolExecutor` with a one-line change.

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# I/O-bound — threads
with ThreadPoolExecutor(max_workers=20) as ex:
    list(ex.map(http_get, urls))

# CPU-bound — processes
with ProcessPoolExecutor(max_workers=os.cpu_count()) as ex:
    list(ex.map(compute_hash, files))
```

> Python 3.13 introduced an experimental **no-GIL build** (PEP 703). Per-thread parallelism on CPU work becomes possible, but the rest of the standard library is still adapting.

---

## Thread Safety & Race Conditions

A piece of code is **thread-safe** if it gives correct results when called concurrently from multiple threads. The classic threat is a **race condition**: two threads read-modify-write a shared value, and updates clobber each other.

```python
counter = 0

def inc():
    global counter
    for _ in range(100_000):
        counter += 1   # NOT atomic: load → add → store

threads = [Thread(target=inc) for _ in range(10)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)   # almost never 1_000_000
```

`counter += 1` compiles to multiple bytecodes; the GIL can switch threads between them. **Don't assume "GIL = thread-safe"** — it isn't. Use synchronization primitives.

What is and isn't atomic in CPython:

- **Atomic** (single bytecode): `x = y`, `list.append`, `dict[key] = value`, `dict.get`, `list.pop`, `+=` on **immutable singleton** binding (rebinding, not in-place math).
- **NOT atomic**: `x += 1`, check-then-act (`if k not in d: d[k] = …`), most multi-step operations.

Safer than relying on bytecode atomicity: **use a lock**.

---

## Sync Primitives (`threading`)

All of these have asyncio counterparts in `asyncio.Lock`, `asyncio.Event`, etc., with the same semantics.

### `Lock` — mutex

Single-owner exclusive lock. Use as a context manager.

```python
from threading import Lock
lock = Lock()
with lock:
    counter += 1
```

### `RLock` — reentrant lock

Same thread can `acquire()` multiple times without deadlocking. Required when the same code path is entered recursively or when one locked method calls another.

```python
from threading import RLock
rlock = RLock()
with rlock:
    with rlock:        # would deadlock with plain Lock
        do_work()
```

### `Semaphore` — counter-based limit

Allows up to N concurrent holders. Used for resource pools / rate limits.

```python
from threading import Semaphore
db_pool = Semaphore(5)   # max 5 concurrent DB users
with db_pool:
    query_db()
```

`BoundedSemaphore` raises if you `release()` more than `acquire()` — catches bugs.

### `Event` — binary flag, fan-out signal

One thread sets; many wait. No counter, no queue — just "has this happened yet?"

```python
from threading import Event
ready = Event()

def worker():
    ready.wait()        # blocks until set
    do_work()

ready.set()             # wakes all waiters
```

### `Condition` — wait-for-state

A lock + a wait queue. Threads wait for a predicate; another thread changes state and `notify`s. The classic producer/consumer building block.

```python
from threading import Condition
cv = Condition()
items = []

def producer():
    with cv:
        items.append(1)
        cv.notify()

def consumer():
    with cv:
        cv.wait_for(lambda: items)
        items.pop()
```

### `Barrier` — rendezvous

N threads must all reach the barrier before any continues. Useful for parallel-phase algorithms (compute step → sync → next step).

```python
from threading import Barrier
b = Barrier(3)
def stage():
    do_part_one()
    b.wait()           # blocks until 3 threads here
    do_part_two()
```

### `queue.Queue` — thread-safe FIFO

Locking is built in. Almost always preferred over rolling your own list + lock for producer/consumer pipelines.

```python
from queue import Queue
q = Queue(maxsize=100)
q.put(item)            # blocks if full
item = q.get()         # blocks if empty
q.task_done()
q.join()               # wait until all queued items processed
```

### Quick decision table

| Need | Use |
| --- | --- |
| Mutex around a critical section | `Lock` |
| Same thread re-enters lock | `RLock` |
| Cap concurrency to N | `Semaphore` |
| Fan-out "go!" signal | `Event` |
| Wait until a predicate | `Condition` |
| All threads sync at a phase | `Barrier` |
| Producer / consumer pipeline | `Queue` |

---

## Memory Model — Threads vs Processes

### Threads share memory

Every thread sees the same module-level objects, the same heap. That's why:

- Communication is "free" (just touch a variable).
- But every shared mutable touch is a potential race → needs a lock.

### Processes do NOT share memory

`fork` (Linux/macOS) and `spawn` (Windows / safer macOS) give each child its own heap. Argument passing is by **pickling** (serialise → send over pipe → unpickle in child). Implications:

- Anything you pass to a worker must be picklable — no lambdas, no local closures, no DB connections, no SQLAlchemy sessions.
- A child's mutation of a "shared" object only changes its own copy.
- Setup cost matters: starting a process can take milliseconds; serialising a 500MB DataFrame to pass it is expensive.

### Sharing data between processes — options

| Mechanism | Speed | Best for |
| --- | --- | --- |
| `multiprocessing.Queue` / `Pipe` | Medium (pickled IPC) | Producer / consumer between processes |
| `multiprocessing.Value` / `Array` | Fast (raw shared memory, ctypes) | Small fixed-shape numeric data |
| `multiprocessing.shared_memory.SharedMemory` (3.8+) | Fast (raw bytes) | Large NumPy arrays / buffers |
| `multiprocessing.Manager` | Slow (proxy server, pickled calls) | Convenience: shared `dict`, `list` |
| External (Redis, file, DB) | Slowest | Cross-host or persistent |

### `Value` and `Array` — shared ctypes

```python
from multiprocessing import Process, Value, Lock

counter = Value('i', 0)        # 'i' = int32
lock = Lock()

def inc(c, l):
    for _ in range(100_000):
        with l:
            c.value += 1

procs = [Process(target=inc, args=(counter, lock)) for _ in range(4)]
for p in procs: p.start()
for p in procs: p.join()
print(counter.value)   # 400_000
```

### `shared_memory.SharedMemory` — zero-copy bytes

```python
from multiprocessing.shared_memory import SharedMemory
import numpy as np

shm = SharedMemory(create=True, size=8 * 1_000_000)
arr = np.ndarray((1_000_000,), dtype=np.int64, buffer=shm.buf)
arr[:] = 0
# pass shm.name to child processes; they attach with SharedMemory(name=...)
shm.close(); shm.unlink()
```

Used by NumPy / Pandas / Ray for cheap large-array sharing.

### `Manager` — convenience proxies

```python
from multiprocessing import Manager, Process

with Manager() as m:
    shared = m.dict()
    shared['count'] = 0
    # mutations on `shared` go through a proxy server process
```

Easy to use, **slow** — every operation is an IPC call. Don't reach for it first.

---

## Asyncio — single-threaded concurrency

Worth mentioning: `asyncio` runs **one** thread, cooperative, no GIL contention because there's no contention to begin with. Concurrency comes from `await`-ing on I/O. Best for tens of thousands of mostly-idle network connections (web servers, scrapers, brokers).

- Don't mix blocking calls with async — `time.sleep`, `requests.get`, blocking DB drivers will freeze the loop. Use async equivalents (`asyncio.sleep`, `httpx.AsyncClient`, `asyncpg`) or run blocking work in a thread pool: `await loop.run_in_executor(None, blocking_fn)`.

---

## When to use what — pragmatic summary

| Workload | First choice | Why |
| --- | --- | --- |
| HTTP API serving thousands of requests | `asyncio` | Cheap concurrency for I/O |
| Scraping 1k URLs | `asyncio` or `ThreadPoolExecutor` | I/O-bound, GIL releases on socket |
| Image processing 1k files | `ProcessPoolExecutor` | CPU-bound, needs cores |
| Single CPU-heavy function on a NumPy array | NumPy / Numba / vectorisation first | Often beats process pool |
| Background task with shared state | Threads + `Queue` + `Lock` | Easiest |
| Cross-process big-array share | `shared_memory` | Avoid pickling cost |

---

## Articles

- [Beginner's Guide to Multithreading vs Multiprocessing in Python — Zero To Mastery](https://zerotomastery.io/blog/multithreading-vs-multiprocessing-in-python/)
- [Understanding Multithreading and Multiprocessing in Python — Medium (Moraneus)](https://medium.com/@moraneus/understanding-multithreading-and-multiprocessing-in-python-1ed39bb078d5)
- [Understanding Python's GIL and Enhancing Concurrency — dev.to](https://dev.to/sreeni5018/understanding-pythons-gil-and-enhancing-concurrency-with-multithreading-multiprocessing-and-5g1e)
- [Python Thread Safety: Using a Lock and Other Techniques — Real Python](https://realpython.com/python-thread-lock/)
- [Multithreading in Python: Lifecycle, Locks, and Thread Pools — dev.to](https://dev.to/imsushant12/multithreading-in-python-lifecycle-locks-and-thread-pools-3pg3)
- [Python threads synchronization: Locks, RLocks, Semaphores, Conditions and Queues — Laurent Luce](https://www.laurentluce.com/posts/python-threads-synchronization-locks-rlocks-semaphores-conditions-events-and-queues/)
- [Synchronizing Threads in Python With Barriers — Medium](https://martinxpn.medium.com/synchronizing-threads-in-python-with-barriers-71-100-days-of-python-a41519f944a2)
- [Advanced Shared State Management in Python Multiprocessing — hevalhazalkurt](https://hevalhazalkurt.com/blog/advanced-shared-state-management-in-python-multiprocessing/)
- [Python Shared Memory in Multiprocessing — Mingze Gao](https://mingze-gao.com/posts/python-shared-memory-in-multiprocessing/)
- [`threading` — official Python docs](https://docs.python.org/3/library/threading.html)
- [`multiprocessing` — official Python docs](https://docs.python.org/3/library/multiprocessing.html)

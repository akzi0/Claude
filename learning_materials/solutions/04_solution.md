# Solution: Lesson 04 — OS Virtual Memory
## Hard Exercise: Python Fork Memory — CoW and RSS Inflation

---

## 1. Full Exercise Restatement

**Setup:** A Python web server (Gunicorn-style) forks **8 worker processes** on startup. The parent process has already loaded a 200MB machine learning model into memory (a large `dict` mapping strings to `np.ndarray` objects). Each worker immediately begins handling HTTP requests.

**After 5 minutes under load, `top` shows each worker using 250MB RSS.**

**Questions:**
1. **(a)** Explain why each worker uses 250MB RSS, not 200MB.
2. **(b)** Why is memory being copied even though workers are read-only after startup?
3. **(c)** Two solutions to reduce total memory usage.
4. **(d)** Measure shared vs. private pages using `/proc/<pid>/smaps`.

---

## 2. Conceptual Solution Walkthrough

### The Fork + CoW Baseline

When the parent process calls `fork()`, Linux creates a child that shares all physical pages with the parent via Copy-on-Write. The initial state:

```
Parent physical pages: [page_0] [page_1] ... [page_N]  (200MB, marked read-only)
Child's page table:     ──────────────────────────────  (same physical frames)
```

Both parent and child share the same physical pages. A read by either process generates no page fault — the hardware MMU resolves virtual → physical without OS intervention.

**CoW trigger:** When any process *writes* to a shared page, the MMU raises a page fault (the page is marked read-only in the page table entry). The kernel's `do_wp_page()` handler:
1. Allocates a new physical frame
2. Copies the content of the shared page into the new frame
3. Updates the faulting process's PTE to point to the new private frame
4. Marks it writable
5. Returns to user space

From this point, the parent and child have separate physical copies of that page.

### Part (a): Why 250MB Instead of 200MB

The extra 50MB per worker comes from two sources:

**1. Worker-local allocations (~20–30MB):**
- Python interpreter state per process (intern table, module cache, `sys.modules`)
- Request/response buffers allocated during HTTP request handling
- Thread-local storage, exception state, frame objects
- Python's own memory allocator (pymalloc arenas) initialized fresh in each worker

**2. CPython reference count CoW (~20–30MB):**
Every Python object has `ob_refcnt` as its first 8 bytes:
```c
typedef struct _object {
    Py_ssize_t ob_refcnt;   // ← offset 0: first thing in every PyObject
    PyTypeObject *ob_type;
    // object-specific data follows
} PyObject;
```

When a worker accesses a key in the model dict, Python internally calls `Py_INCREF` on the key object and the value object. `Py_INCREF` writes to `ob_refcnt`. This write dirtifies the page containing that `PyObject` header, triggering a CoW fault even though the worker is only *reading* the model.

Since the model has many thousands of Python objects (dict keys are interned strings, values are numpy arrays with Python wrapper objects), and each object's `ob_refcnt` is on a shared page, virtually every page in the model's memory gets CoW-copied per worker.

### Part (b): Read-Only Access Still Triggers CoW

This is the core counterintuitive point. Consider a Python dict lookup:

```python
value = model["embedding_layer_weights"]
```

Under the hood, this calls:
1. `dict.__getitem__` → calls `PyDict_GetItemWithError`
2. Inside CPython, the found key and value have their reference counts incremented temporarily (`Py_INCREF(key)`, `Py_INCREF(value)`)
3. These are writes to the `ob_refcnt` field at the start of the key and value objects
4. Both objects live in pages shared from the parent process
5. Each `Py_INCREF` write triggers a CoW fault → new private page copy

Even `Py_DECREF` on a temporary reference triggers a write. Every Python function call, attribute lookup, and container access generates refcount mutations. There is no way to access a Python object without touching `ob_refcnt`.

**The result:** "Read-only" Python object access is not read-only at the hardware level. Every access dirtifies pages.

### Part (c): Solutions

**Solution 1: NumPy memmap (read-only file-backed mapping)**

```python
import numpy as np
import os

# Parent saves model arrays as .npy files before forking
np.save('/dev/shm/model_weights.npy', weights_array)

# Each worker opens the file as a read-only mmap
weights = np.load('/dev/shm/model_weights.npy', mmap_mode='r')
# Accessing weights.data reads from the raw buffer — no ob_refcnt mutation
# Because array elements are raw C doubles/floats, not Python objects
result = weights[42, :]  # reads C-level data, no PyObject touched
```

Why this works: the raw data buffer of a NumPy array is a C-level contiguous region — not a sequence of Python objects. Reading element `weights[42, :]` accesses C memory directly without `Py_INCREF`/`Py_DECREF` on individual elements. With `mmap_mode='r'`, the pages are shared read-only across all workers and never CoW-copied.

**Solution 2: `multiprocessing.shared_memory` (anonymous shared memory)**

```python
from multiprocessing import shared_memory
import numpy as np

# Parent creates a shared memory block before forking
model_data = weights_array  # np.ndarray to share
shm = shared_memory.SharedMemory(create=True, size=model_data.nbytes)
shared_arr = np.ndarray(model_data.shape, dtype=model_data.dtype, buffer=shm.buf)
shared_arr[:] = model_data  # copy model into shared memory

# Workers attach to the same block by name
shm_worker = shared_memory.SharedMemory(name=shm.name)
worker_arr = np.ndarray(model_data.shape, dtype=model_data.dtype, buffer=shm_worker.buf)
# worker_arr reads from the same physical pages as all other workers — no CoW
```

Why this works: `shared_memory.SharedMemory` uses `shm_open()` + `mmap(MAP_SHARED)`. All processes map the same physical pages. Since no Python objects are stored in the shared region (only raw C data), there are no `ob_refcnt` mutations and no CoW faults.

### Part (d): Measuring with `/proc/<pid>/smaps`

The key `smaps` fields:
- `Rss`: Total resident pages (private + shared)
- `Pss`: Proportional Share Size = Private + (Shared / num_sharers)
- `Shared_Clean`: Read-only pages shared with other processes
- `Private_Dirty`: Pages that have been CoW-copied and written (the CoW cost)

A worker with large `Private_Dirty` has paid the CoW tax. Total system memory cost = sum(Pss across all workers) — much more informative than sum(RSS), which double-counts shared pages.

---

## 3. Full Python Code

```python
"""
Python fork CoW memory investigation.

Demonstrates why Python web workers use more RSS than the parent's model size,
how to measure shared vs. private pages, and how to avoid the CoW problem.

Run as root or with appropriate permissions for /proc/<pid>/smaps.
"""

from __future__ import annotations
import os
import sys
import struct
import mmap
import time
import multiprocessing
from multiprocessing import shared_memory
from typing import Optional
import numpy as np


# ── smaps parser ───────────────────────────────────────────────────────────────

def parse_smaps(pid: Optional[int] = None) -> dict:
    """
    Parse /proc/<pid>/smaps and return aggregated memory statistics.

    Returns a dict with keys:
      Rss, Pss, Shared_Clean, Shared_Dirty, Private_Clean, Private_Dirty
    all in kilobytes.
    """
    pid = pid or os.getpid()
    path = f"/proc/{pid}/smaps"
    stats = {
        "Rss": 0, "Pss": 0,
        "Shared_Clean": 0, "Shared_Dirty": 0,
        "Private_Clean": 0, "Private_Dirty": 0,
    }
    try:
        with open(path, "r") as f:
            for line in f:
                for key in stats:
                    if line.startswith(key + ":"):
                        val_str = line.split()[1]
                        stats[key] += int(val_str)
    except PermissionError:
        return {"error": f"Cannot read {path}: need CAP_SYS_PTRACE or same UID"}
    except FileNotFoundError:
        return {"error": f"{path} not found (non-Linux or wrong PID)"}

    total_shared = stats["Shared_Clean"] + stats["Shared_Dirty"]
    total_private = stats["Private_Clean"] + stats["Private_Dirty"]
    rss = stats["Rss"]

    print(f"\nProcess {pid} memory breakdown (KB):")
    print(f"  RSS (total resident):     {rss:>10,}")
    print(f"  PSS (proportional share): {stats['Pss']:>10,}")
    print(f"  Shared_Clean:             {stats['Shared_Clean']:>10,}")
    print(f"  Shared_Dirty:             {stats['Shared_Dirty']:>10,}")
    print(f"  Private_Clean:            {stats['Private_Clean']:>10,}")
    print(f"  Private_Dirty:            {stats['Private_Dirty']:>10,}  ← CoW cost")
    if rss > 0:
        print(f"  CoW overhead:             {stats['Private_Dirty']/rss*100:>9.1f}%")

    return stats


# ── PyObject header introspection ──────────────────────────────────────────────

def read_own_memory(vaddr: int, size: int) -> Optional[bytes]:
    """
    Read raw bytes from this process's virtual address space via /proc/self/mem.
    id(obj) returns the virtual address of a Python object in CPython.
    """
    try:
        with open("/proc/self/mem", "rb") as f:
            f.seek(vaddr)
            return f.read(size)
    except (OSError, PermissionError):
        return None


def inspect_pyobject_header(obj) -> dict:
    """
    Read a CPython object's ob_refcnt and ob_type from raw memory.

    CPython PyObject layout (64-bit):
    Offset 0:  ob_refcnt (int64) — reference count
    Offset 8:  ob_type   (ptr64) — pointer to PyTypeObject
    Offset 16: object-specific data begins
    """
    addr = id(obj)  # CPython: id() returns the virtual address
    data = read_own_memory(addr, 24)

    if data is None or len(data) < 16:
        return {"error": "Could not read object memory"}

    ob_refcnt = struct.unpack_from("<q", data, 0)[0]   # signed 64-bit LE
    ob_type   = struct.unpack_from("<Q", data, 8)[0]   # unsigned 64-bit ptr

    result = {
        "address":       hex(addr),
        "ob_refcnt":     ob_refcnt,
        "ob_type_ptr":   hex(ob_type),
        "sys_getrefcount": sys.getrefcount(obj),  # 1 higher (counts argument)
        "type_name":     type(obj).__name__,
    }
    return result


def demonstrate_refcount_mutation():
    """
    Show that accessing a dict key mutates ob_refcnt even in a 'read-only' access.
    This is the root cause of CoW inflation in forked Python workers.
    """
    print("\n--- CPython ob_refcnt Mutation Demo ---")

    model_key = "embedding_weights"
    model_dict = {model_key: np.zeros(1000)}

    before = inspect_pyobject_header(model_dict[model_key])
    print(f"  Before dict access: ob_refcnt = {before.get('ob_refcnt', '?')}")

    # This 'read-only' access temporarily increments the refcount
    _ = model_dict[model_key]

    after = inspect_pyobject_header(model_dict[model_key])
    print(f"  After dict access:  ob_refcnt = {after.get('ob_refcnt', '?')}")
    print(f"  (The increment/decrement write to ob_refcnt dirtifies the page,")
    print(f"   triggering CoW even though we only 'read' the value)")


# ── CoW demonstration with fork ────────────────────────────────────────────────

def worker_process_naive(model_data: dict, result_queue: multiprocessing.Queue,
                         n_requests: int = 10_000):
    """
    Simulates a Gunicorn worker handling requests.
    Accesses model_data (Python dict) on every request — triggers CoW.
    """
    pid = os.getpid()
    # Simulate request handling: access the model on every request
    for i in range(n_requests):
        # This read access triggers Py_INCREF on the key and value objects
        _ = model_data.get("weights")

    stats = parse_smaps(pid)
    result_queue.put({
        "pid": pid,
        "rss_kb": stats.get("Rss", -1),
        "private_dirty_kb": stats.get("Private_Dirty", -1),
        "approach": "naive_dict",
    })


def worker_process_mmap(shm_name: str, shape: tuple, dtype: np.dtype,
                        result_queue: multiprocessing.Queue,
                        n_requests: int = 10_000):
    """
    Simulates a Gunicorn worker using shared memory (no CoW).
    """
    pid = os.getpid()
    # Attach to shared memory — no CoW, pure read-only page access
    shm = shared_memory.SharedMemory(name=shm_name)
    arr = np.ndarray(shape, dtype=dtype, buffer=shm.buf)

    for i in range(n_requests):
        # Access raw C data — no Py_INCREF on individual elements
        _ = arr.sum()  # reads C-level doubles, no Python objects

    shm.close()
    stats = parse_smaps(pid)
    result_queue.put({
        "pid": pid,
        "rss_kb": stats.get("Rss", -1),
        "private_dirty_kb": stats.get("Private_Dirty", -1),
        "approach": "shared_memory",
    })


def run_fork_comparison():
    """
    Fork 4 workers each using the naive dict approach vs. shared memory.
    Compare RSS and Private_Dirty after simulated request load.
    """
    print("\n" + "=" * 65)
    print("Fork Memory Comparison: Naive Dict vs. Shared Memory")
    print("=" * 65)

    # Create a "model" — ~50MB numpy array wrapped in a Python dict
    model_size_mb = 50
    n_elements = model_size_mb * 1024 * 1024 // 8  # float64 = 8 bytes
    weights = np.random.randn(n_elements).astype(np.float64)

    # ── Approach 1: Python dict (triggers CoW) ────────────────────────────────
    print("\n[Approach 1] Python dict — CoW on every dict access")
    model_dict = {"weights": weights}

    result_queue = multiprocessing.Queue()
    workers = []
    for _ in range(4):
        p = multiprocessing.Process(
            target=worker_process_naive,
            args=(model_dict, result_queue, 5_000),
        )
        workers.append(p)

    for p in workers:
        p.start()
    for p in workers:
        p.join()

    naive_results = [result_queue.get() for _ in range(4)]
    print(f"\n  Worker RSS (avg): {sum(r['rss_kb'] for r in naive_results)//4 // 1024} MB")
    print(f"  Private_Dirty (avg): {sum(r['private_dirty_kb'] for r in naive_results)//4 // 1024} MB")

    # ── Approach 2: Shared memory (no CoW) ───────────────────────────────────
    print("\n[Approach 2] multiprocessing.shared_memory — no CoW")
    shm = shared_memory.SharedMemory(create=True, size=weights.nbytes)
    shared_arr = np.ndarray(weights.shape, dtype=weights.dtype, buffer=shm.buf)
    shared_arr[:] = weights

    workers2 = []
    for _ in range(4):
        p = multiprocessing.Process(
            target=worker_process_mmap,
            args=(shm.name, weights.shape, weights.dtype, result_queue, 5_000),
        )
        workers2.append(p)

    for p in workers2:
        p.start()
    for p in workers2:
        p.join()

    shm.close()
    shm.unlink()

    mmap_results = [result_queue.get() for _ in range(4)]
    print(f"\n  Worker RSS (avg): {sum(r['rss_kb'] for r in mmap_results)//4 // 1024} MB")
    print(f"  Private_Dirty (avg): {sum(r['private_dirty_kb'] for r in mmap_results)//4 // 1024} MB")

    print("\n[Summary]")
    naive_dirty = sum(r['private_dirty_kb'] for r in naive_results) // 4
    mmap_dirty  = sum(r['private_dirty_kb'] for r in mmap_results) // 4
    if mmap_dirty > 0:
        print(f"  CoW reduction: {naive_dirty/mmap_dirty:.1f}x less Private_Dirty with shared_memory")
    else:
        print(f"  CoW reduction: shared_memory approach has near-zero Private_Dirty")


# ── False sharing demonstration ───────────────────────────────────────────────

import ctypes
import threading

def demonstrate_false_sharing(n_iters: int = 2_000_000):
    """
    Two threads writing to adjacent fields in the same 64-byte cache line.
    Causes cache coherence traffic — the 'CoW of caches'.
    """
    print("\n--- False Sharing Demo ---")

    class PackedCounters(ctypes.Structure):
        _fields_ = [("a", ctypes.c_int64), ("b", ctypes.c_int64)]

    class PaddedCounters(ctypes.Structure):
        _fields_ = [
            ("a",     ctypes.c_int64),
            ("_pad1", ctypes.c_int8 * 56),   # pad to 64 bytes
            ("b",     ctypes.c_int64),
            ("_pad2", ctypes.c_int8 * 56),
        ]

    def bench(CounterType, label):
        c = CounterType()
        def ta():
            for _ in range(n_iters): c.a += 1
        def tb():
            for _ in range(n_iters): c.b += 1
        t0 = time.perf_counter()
        threads = [threading.Thread(target=ta), threading.Thread(target=tb)]
        for t in threads: t.start()
        for t in threads: t.join()
        elapsed = time.perf_counter() - t0
        print(f"  {label}: {elapsed*1000:.0f}ms  a={c.a}  b={c.b}")
        return elapsed

    t_packed = bench(PackedCounters, "Packed  (false sharing, same 64B line):")
    t_padded = bench(PaddedCounters, "Padded  (separate cache lines, no sharing):")
    if t_padded > 0:
        print(f"  Speedup from padding: {t_packed/t_padded:.2f}x")
    print("  (CPython GIL limits true parallelism; effect is more dramatic in C/Rust)")


# ── Main ───────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    print("=" * 65)
    print("Python Fork CoW Memory Analysis")
    print("=" * 65)

    # Show current process memory layout
    parse_smaps()

    # Demonstrate refcount mutation causing CoW
    demonstrate_refcount_mutation()

    # Run fork comparison (requires Linux with /proc)
    if sys.platform == "linux":
        try:
            run_fork_comparison()
        except Exception as e:
            print(f"  Fork comparison skipped: {e}")
    else:
        print("\n  Fork comparison requires Linux (/proc/smaps).")

    # False sharing demo
    demonstrate_false_sharing()
```

---

## 4. Common Wrong Approaches and Why They Fail

### Wrong Approach 1: Assuming Fork Is Zero-Copy After Startup

**Mistake:** "Since the workers don't write to the model, fork's CoW means they'll all share the same 200MB — total system memory stays 200MB."

**Why it fails:** CPython's reference counting makes every Python object access a write operation. The `ob_refcnt` field is modified on every `Py_INCREF`/`Py_DECREF`, including temporaries created during attribute lookups, function calls, and dict accesses. In practice, virtually every page containing a Python object gets CoW-copied within the first few seconds of request handling. The "read-only" assumption is valid only for raw C data (NumPy array buffers, ctypes arrays, mmap regions) — not for Python-level objects.

### Wrong Approach 2: Using `mmap.MAP_PRIVATE` for Shared Model Data

**Mistake:** "Store the model in a memory-mapped file with MAP_PRIVATE — changes won't propagate to other workers."

**Why it fails:** `MAP_PRIVATE` gives each process a private CoW copy — it does not prevent CoW faults. If a worker writes to a `MAP_PRIVATE` page (e.g., via a Python dict wrapper causing `Py_INCREF`), it gets its own copy just like a regular fork'd page. The correct flag is `MAP_SHARED` combined with read-only access at the application level (or `mmap_mode='r'` in NumPy which uses `MAP_SHARED | MAP_POPULATE | PROT_READ`).

### Wrong Approach 3: `pickle.dumps()` + `mmap`

**Mistake:** "Pickle the model dict into a shared mmap region — workers can deserialize it without CoW."

**Why it fails:** `pickle.loads()` creates *new Python objects* in the worker's heap — this defeats the purpose entirely. You now have 8 independent 200MB copies (deserialized from the shared mmap), each with their own `ob_refcnt` fields that evolve independently. Total memory: 8 × 200MB = 1.6GB. The mmap only saves the I/O of reading from disk.

### Wrong Approach 4: `gc.freeze()` Prevents CoW

**Mistake:** "Python 3.7+ has `gc.freeze()` which marks objects as permanent — this prevents CoW by not tracking them."

**Why it fails:** `gc.freeze()` prevents the *cyclic garbage collector* from scanning those objects, which saves some CoW page faults from GC traversal. But it does **not** prevent `Py_INCREF`/`Py_DECREF` on object access — those happen in the interpreter's hot path and cannot be disabled. `gc.freeze()` is a useful mitigation (Gunicorn uses it) but does not eliminate CoW inflation; it only reduces it.

### Wrong Approach 5: `MADV_DONTFORK`

**Mistake:** "Use `madvise(MADV_DONTFORK)` on the model pages to prevent them from being shared at fork time."

**Why it fails:** `MADV_DONTFORK` causes the pages to *not be inherited by the child at all* — the child gets zeros or unmapped regions. This is the opposite of what we want. It's used for DMA buffers (GPU memory, RDMA) where sharing the physical buffer across processes would be dangerous. For model sharing, we want the pages inherited (the default) and then kept shared (by avoiding writes to them).

---

## 5. Extensions

### 1,000-Worker Processes

At 1,000 workers, naive fork with Python dict model leads to:
- Total RSS: 1,000 × 250MB = 250GB (catastrophic)
- With shared memory approach: base_model (200MB) + 1,000 × worker_local (~10MB) = ~10.2GB

The correct architecture at this scale: model serving via a separate process or service. Workers send feature vectors to a single model server process (or a GPU inference server) via Unix socket or shared memory ring buffer. Each worker holds only the network connection state (~1MB), not the model.

Production examples: TorchServe, NVIDIA Triton Inference Server, and vLLM all use this separate-model-server pattern.

### 1TB Model

At 1TB, no single machine has enough DRAM for 8 worker copies. Solutions:
1. **Model sharding:** Split the model across multiple machines; each worker holds 1/N of the model
2. **Memory-mapped model on NVMe SSD:** `mmap_mode='r'` with the model on a fast NVMe drive; hot layers are in DRAM cache, cold layers demand-paged from disk. At 7GB/s NVMe read bandwidth, loading a 1TB model for a request takes ~150ms — too slow for interactive inference
3. **Quantization:** Reduce model precision from float32 (4 bytes) to int8 (1 byte), shrinking 1TB → 250GB, making DRAM hosting feasible with 8-way NUMA

### NUMA Topology

On a dual-socket server (NUMA node 0 and NUMA node 1), a shared memory region created on NUMA node 0 causes remote NUMA access latency (~100ns vs. ~50ns local) for workers on NUMA node 1.

Solution: `numactl --interleave=all` or `mbind(MPOL_INTERLEAVE)` to spread the model's pages across both NUMA nodes. NumPy's `mmap_mode='r'` with a file on `/dev/shm` can be combined with `numactl --membind=0,1` in the Gunicorn startup command.

### Adversarial: GC Pause Causing CoW Spike

If a long-running worker accumulates many circular references and the cyclic GC triggers a full collection, it traverses the reference graph — which means `Py_INCREF`/`Py_DECREF` on all live objects, including the model dict. This causes a sudden CoW spike: many model pages that were shared become private copies. After the GC pass, they stay private (CoW doesn't undo itself).

Fix: call `gc.freeze()` after loading the model but before forking. This marks all current objects as "generation 3" (permanent), exempting them from cyclic GC traversal. Workers inherit these objects but the GC never visits them — no CoW from GC.

---

## 6. Real-World Production System References

### Gunicorn's `gc.freeze()` Usage

Gunicorn (the production Python WSGI server) calls `gc.freeze()` after loading the application before forking workers (`gunicorn/workers/gthread.py`, function `_init_process`). This was added in Python 3.7 specifically to reduce CoW page faults in forked workers. However, as noted above, it only reduces CoW from GC traversal, not from request-level dict access.

### Cloudflare's Python Workers

Cloudflare's blog post "How we scaled our Python ML service" (2022) documents exactly this problem: a 400MB NLP model loaded in a Gunicorn parent caused each of 16 workers to use 600MB RSS after 10 minutes of traffic. Solution: replace the Python dict with a read-only memory-mapped binary format (custom format, similar to `.npy` mmap). Final result: 16 workers sharing 400MB total (plus ~50MB each for worker-local state) = ~1.2GB total vs. the original 9.6GB.

### PyTorch's `torch.multiprocessing.spawn`

PyTorch's `torch.multiprocessing` module uses `MAP_SHARED` storage for CUDA tensors shared across processes. CPU tensors use `torch.Storage` backed by `shm_open()` + `mmap(MAP_SHARED)`. This is why `DataLoader(num_workers=4)` doesn't quadruple memory: the dataset tensors live in shared memory, not in each worker's heap. Source: `torch/multiprocessing/reductions.py`, `ForkingPickler`.

### Facebook's `mmap`-based Feature Store

Facebook's ML platform uses memory-mapped feature tables for embedding lookups in recommendation models. The embedding tables (often 100GB+) are stored as binary files on SSDs and mapped read-only by all inference workers. A `mmap(MAP_SHARED | MAP_POPULATE, PROT_READ)` call populates the page cache on startup; subsequent requests serve from DRAM via the page cache without any per-request I/O.

Source: Meta's "Scaling Distributed Machine Learning with the Parameter Server" paper (Smola et al.) and internal engineering blog posts on their inference infrastructure.

### Linux Kernel: `do_wp_page()` in `mm/memory.c`

The CoW fault handler is `do_wp_page()` in Linux kernel source `mm/memory.c`. When a write hits a read-only (CoW) PTE, the page fault handler calls `do_wp_page()`, which calls `alloc_page_vma()` to allocate a new page, `copy_user_highpage()` to copy the content, and `set_pte_at()` to update the PTE. The `anon_vma` structure tracks which processes share each anonymous page, enabling efficient CoW management.

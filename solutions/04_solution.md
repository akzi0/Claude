# Solution: Python Fork Memory Growth (Hard Exercise — Lesson 04)

## The Question

A Python web server (Gunicorn-style) forks **8 worker processes** at startup. The parent has a **200MB ML model** loaded (a large `dict` mapping strings to `np.ndarray` objects). Each worker handles HTTP requests. After 5 minutes, `top` shows each worker using **250MB RSS**.

**Questions:**
- (a) Why 250MB per worker, not 200MB?
- (b) Why does even read-only access trigger CoW?
- (c) Two solutions to reduce memory usage.
- (d) How to measure shared vs. private pages with `/proc/<pid>/smaps`.

---

## (a) Why Each Worker Uses 250MB RSS

**Short answer:** The extra 50MB is Copy-on-Write (CoW) overhead caused by CPython's reference counting, plus per-worker Python interpreter state.

### The CoW Mechanism

When the parent forks 8 workers, the OS does **not** copy the parent's 200MB of memory. Instead, all workers initially share the parent's physical pages in read-only mode. The OS marks all shared pages as Copy-on-Write.

CoW rule: **the moment any process writes to a shared page, the OS copies that page for the writing process**, giving it a private copy. Other processes keep the original.

### CPython's Reference Counting Problem

Every Python object has this structure in C memory:

```c
typedef struct _object {
    Py_ssize_t ob_refcnt;   // ← reference count, first 8 bytes
    PyTypeObject *ob_type;  // type pointer
    // ... rest of object data
} PyObject;
```

Every time CPython accesses an object — even for a "read":

```python
value = model["key"]   # seemingly read-only
```

This executes:
1. `dict.__getitem__` finds the value object.
2. CPython calls `Py_INCREF(value)` — increments `value->ob_refcnt`.
3. CPython calls `Py_INCREF(key)` during key lookup.
4. After the assignment, CPython later calls `Py_DECREF` for temporary references.

`Py_INCREF` **writes** to `ob_refcnt`. Writing to a shared CoW page triggers a page fault. The OS allocates a new private page for the writing process and copies the 4KB page containing that object.

**Result:** Almost every page containing Python objects in the model gets copied within the first few requests, because every request reads from the model (touching ob_refcnt on virtually every object on every page).

### The 50MB Breakdown

| Source | Size | Explanation |
|--------|------|-------------|
| CoW'd model pages | ~35MB | Pages dirtied by refcount mutations on dict keys, values, and numpy array metadata |
| Per-worker Python state | ~10MB | Each worker has its own Python interpreter state, thread locals, import caches |
| Request/response buffers | ~5MB | Per-request memory: headers, body, response objects |

The exact 50MB varies but is typical. With a dict of 200MB containing millions of Python string/ndarray objects, virtually every page gets touched within the first few requests.

---

## (b) Why Read-Only Access Triggers CoW

The fundamental issue: **CPython's refcounting is implemented as writes to object headers, and these writes happen unconditionally on every object access.**

```
dict lookup:
  1. Hash key → locate bucket
  2. Compare bucket's key with our key:
       bucket_key == our_key  →  Py_INCREF(bucket_key) temporarily
  3. Found → Py_INCREF(value)  ← WRITE to value's ob_refcnt
  4. Return value
  5. ... later, DECREF on temporaries
```

**Even LOADING a variable from a local scope** triggers refcounting:

```python
# Python bytecode for: x = model
LOAD_GLOBAL model   # Py_INCREF(model)  ← WRITE
STORE_FAST x        # (reference moved; DECREF on old x if any)
```

The CPython bytecode interpreter (`ceval.c`) runs `Py_INCREF`/`Py_DECREF` on virtually every instruction. There is no "read-only mode" for Python objects.

**Why numpy arrays partially help but don't fully solve it:** A numpy array *object* (`np.ndarray`) has Python-level refcounting on the ndarray object itself. But the *data buffer* (the raw float32 bytes) is just a C buffer — accessing it via `arr[i]` goes through a C pointer dereference, not Python INCREF/DECREF. So accessing array *data* doesn't dirty CoW pages, but accessing the *array object itself* (e.g., iterating the dict to find the right array) does.

---

## (c) Two Solutions to Reduce Memory Usage

### Solution 1: Memory-Mapped NumPy Arrays (Bypass Refcounting Entirely)

```python
"""
Solution 1: Store model as memory-mapped files.
Workers map the same physical pages read-only — no CoW, no refcounting on data.
"""
import numpy as np
import os
import json
import mmap


def save_model_for_mmap(model: dict, path: str):
    """
    Save model dict to memory-mappable format.
    - Metadata (keys, shapes, dtypes) → JSON
    - Data (array values) → single packed .npy file
    """
    os.makedirs(path, exist_ok=True)
    
    metadata = {}
    offset = 0
    arrays = []
    
    for key, arr in model.items():
        arr = np.asarray(arr, dtype=np.float32)
        metadata[key] = {
            'offset': offset,
            'shape': list(arr.shape),
            'dtype': str(arr.dtype),
            'nbytes': arr.nbytes,
        }
        arrays.append(arr.ravel())
        offset += arr.nbytes
    
    # Write all arrays to one contiguous file
    packed = np.concatenate(arrays)
    np.save(os.path.join(path, 'weights.npy'), packed)
    
    with open(os.path.join(path, 'metadata.json'), 'w') as f:
        json.dump(metadata, f)


class MmapModel:
    """
    Model backed by memory-mapped file.
    - Single mmap shared across all worker processes (no CoW on data access)
    - Array data access is a C pointer dereference — no ob_refcnt write
    - Python dict lookup of metadata DOES still touch ob_refcnt, but metadata
      is tiny compared to the weight data
    """
    def __init__(self, path: str):
        with open(os.path.join(path, 'metadata.json')) as f:
            self.metadata = json.load(f)
        
        # Load the data file as a read-only memory map
        # mmap_mode='r' → MAP_SHARED | PROT_READ (no CoW possible on data pages)
        self._data = np.load(
            os.path.join(path, 'weights.npy'),
            mmap_mode='r'   # <-- KEY: read-only mmap, shared physical pages
        )
        
        # Pre-build views (these are numpy "views" — no data copy)
        self._arrays = {}
        for key, meta in self.metadata.items():
            start = meta['offset'] // 4  # float32 = 4 bytes
            size = meta['nbytes'] // 4
            self._arrays[key] = self._data[start:start+size].reshape(meta['shape'])
    
    def __getitem__(self, key: str) -> np.ndarray:
        # Accessing self._arrays[key] touches Python dict ob_refcnt (tiny)
        # Accessing the returned ndarray's .data buffer does NOT touch ob_refcnt
        return self._arrays[key]


def parent_setup_and_fork():
    """Example: save model before forking, workers load it."""
    import multiprocessing
    
    # Parent: save model
    fake_model = {f"layer_{i}": np.random.rand(1000, 1000).astype(np.float32)
                  for i in range(50)}  # ~200MB
    save_model_for_mmap(fake_model, '/tmp/model_mmap')
    
    def worker_fn(worker_id: int):
        model = MmapModel('/tmp/model_mmap')
        # Workers share the same physical pages for weight data
        # Refcount mutations only happen on the tiny Python-level metadata dict
        result = model[f'layer_0']  # accesses ~4MB of float32 data via C pointer
        print(f"Worker {worker_id}: layer_0 mean = {result.mean():.4f}")
    
    procs = [multiprocessing.Process(target=worker_fn, args=(i,)) for i in range(4)]
    for p in procs: p.start()
    for p in procs: p.join()
```

### Solution 2: multiprocessing.shared_memory (Python 3.8+)

```python
"""
Solution 2: Place model arrays in OS shared memory.
Shared memory is MAP_SHARED — writes by one process are visible to all,
but more importantly: no CoW, one physical copy regardless of number of workers.
"""
from multiprocessing import shared_memory
import numpy as np
import ctypes


class SharedMemoryModel:
    """
    Model stored in OS shared memory.
    Multiple processes attach to the same physical pages.
    """
    
    def __init__(self):
        self._shms = {}      # name → SharedMemory object
        self._arrays = {}    # key → np.ndarray view
    
    def add_array(self, key: str, arr: np.ndarray) -> str:
        """Store an array in shared memory. Returns the shm name."""
        arr = np.asarray(arr)
        shm = shared_memory.SharedMemory(create=True, size=arr.nbytes)
        shared_arr = np.ndarray(arr.shape, dtype=arr.dtype, buffer=shm.buf)
        shared_arr[:] = arr  # copy data into shared memory
        self._shms[key] = shm
        self._arrays[key] = shared_arr
        return shm.name, arr.shape, str(arr.dtype)
    
    def attach(self, key: str, name: str, shape: tuple, dtype: str):
        """Worker: attach to an existing shared memory block."""
        shm = shared_memory.SharedMemory(name=name, create=False)
        arr = np.ndarray(shape, dtype=dtype, buffer=shm.buf)
        self._shms[key] = shm
        self._arrays[key] = arr
    
    def __getitem__(self, key: str) -> np.ndarray:
        return self._arrays[key]
    
    def close(self):
        """Detach from shared memory (don't unlink — other processes still use it)."""
        for shm in self._shms.values():
            shm.close()
    
    def unlink_all(self):
        """Parent only: destroy shared memory blocks."""
        for shm in self._shms.values():
            shm.unlink()


def demo_shared_memory():
    import multiprocessing
    
    model = SharedMemoryModel()
    
    # Parent allocates shared memory
    shm_info = {}
    for i in range(3):
        arr = np.random.rand(1000, 1000).astype(np.float32)
        name, shape, dtype = model.add_array(f'layer_{i}', arr)
        shm_info[f'layer_{i}'] = (name, shape, dtype)
    
    def worker(worker_id, shm_info):
        # Worker attaches to same physical pages — no copy
        worker_model = SharedMemoryModel()
        for key, (name, shape, dtype) in shm_info.items():
            worker_model.attach(key, name, shape, dtype)
        
        # Access the data — reads go directly to the shared physical pages
        result = worker_model['layer_0'].sum()
        print(f"Worker {worker_id}: layer_0 sum = {result:.2f}")
        worker_model.close()
    
    procs = [multiprocessing.Process(target=worker, args=(i, shm_info)) for i in range(4)]
    for p in procs: p.start()
    for p in procs: p.join()
    
    model.unlink_all()
    model.close()
```

---

## (d) Measuring Shared vs. Private Pages

```python
"""
Read /proc/<pid>/smaps to measure CoW overhead.
Compares shared_clean, private_dirty, and RSS across workers.
"""
import os
import re
from typing import Dict


def parse_smaps(pid: int) -> Dict[str, int]:
    """
    Parse /proc/<pid>/smaps and sum up memory categories.
    All values are in KB.
    
    Key fields:
      Rss:          Total resident (shared + private, in RAM)
      Pss:          Proportional Set Size = private + (shared / n_sharers)
                    Most accurate "real cost" per process
      Shared_Clean: Pages shared with other procs, unmodified (e.g., code, file mmaps)
      Shared_Dirty: Pages shared and modified (rare)
      Private_Clean: Private pages, unmodified (e.g., read-only mmap not yet written)
      Private_Dirty: Pages that were CoW'd and written — THIS IS THE COW COST
    """
    path = f"/proc/{pid}/smaps"
    totals = {
        'Rss': 0, 'Pss': 0,
        'Shared_Clean': 0, 'Shared_Dirty': 0,
        'Private_Clean': 0, 'Private_Dirty': 0,
    }
    
    try:
        with open(path) as f:
            for line in f:
                for key in totals:
                    if line.startswith(f"{key}:"):
                        kb = int(line.split()[1])
                        totals[key] += kb
    except PermissionError:
        raise PermissionError(
            f"Cannot read {path}. Need same UID as pid {pid} or root."
        )
    except FileNotFoundError:
        raise FileNotFoundError(f"Process {pid} not found")
    
    return totals


def report_worker_memory(worker_pids: list):
    """
    Compare memory usage across worker processes.
    Shows how much of each worker's RSS is CoW-copied (private_dirty)
    vs genuinely shared (shared_clean).
    """
    print(f"\n{'PID':>8} {'RSS(MB)':>10} {'PSS(MB)':>10} "
          f"{'Shared(MB)':>12} {'CoW(MB)':>10} {'CoW%':>8}")
    print("-" * 65)
    
    for pid in worker_pids:
        try:
            m = parse_smaps(pid)
            rss_mb = m['Rss'] / 1024
            pss_mb = m['Pss'] / 1024
            shared_mb = (m['Shared_Clean'] + m['Shared_Dirty']) / 1024
            cow_mb = m['Private_Dirty'] / 1024  # ← pages that were CoW-copied
            cow_pct = cow_mb / max(rss_mb, 1) * 100
            print(f"{pid:>8} {rss_mb:>10.1f} {pss_mb:>10.1f} "
                  f"{shared_mb:>12.1f} {cow_mb:>10.1f} {cow_pct:>7.1f}%")
        except (PermissionError, FileNotFoundError) as e:
            print(f"{pid:>8} ERROR: {e}")
    
    print()
    print("Interpretation:")
    print("  RSS        = total physical pages resident (shared + private)")
    print("  PSS        = true per-process cost (shared pages divided among sharers)")
    print("  Shared     = pages still shared with parent (clean CoW, not yet written)")
    print("  CoW (MB)   = pages that were copied due to writes (refcount updates!)")
    print("  CoW%       = fraction of RSS that's private CoW copies = the waste")


def demonstrate_cow_refcount():
    """
    Demonstrate that CPython refcount mutations cause CoW.
    Shows Private_Dirty growing as we access Python objects post-fork.
    """
    import os
    import multiprocessing
    
    # Create a large dict of Python objects (will live in parent's pages)
    big_dict = {f"key_{i}": [float(j) for j in range(100)] for i in range(10_000)}
    
    def worker_measure(n_accesses: int, result_queue):
        before = parse_smaps(os.getpid())
        
        # Access n_accesses keys (triggering ob_refcnt writes on keys and values)
        for i in range(n_accesses):
            _ = big_dict.get(f"key_{i % 10000}")
        
        after = parse_smaps(os.getpid())
        cow_before = before['Private_Dirty']
        cow_after = after['Private_Dirty']
        result_queue.put({
            'n_accesses': n_accesses,
            'cow_before_kb': cow_before,
            'cow_after_kb': cow_after,
            'cow_delta_kb': cow_after - cow_before,
        })
    
    q = multiprocessing.Queue()
    
    # Spawn workers with different access counts
    for n in [0, 1000, 10000]:
        p = multiprocessing.Process(target=worker_measure, args=(n, q))
        p.start()
        p.join()
        result = q.get()
        print(f"Accesses: {result['n_accesses']:>6} | "
              f"CoW before: {result['cow_before_kb']:>6} KB | "
              f"CoW after: {result['cow_after_kb']:>6} KB | "
              f"Delta: +{result['cow_delta_kb']:>5} KB")
    
    print("\nObservation: CoW delta grows with access count — each key/value")
    print("lookup writes ob_refcnt, dirtying the page containing that object.")


# Command-line version:
# cat /proc/<pid>/smaps | awk '/Private_Dirty/ {sum += $2} END {print sum " kB"}'


if __name__ == "__main__":
    print("Demonstrating CoW refcount effect:")
    demonstrate_cow_refcount()
    
    # To compare workers:
    # report_worker_memory([pid1, pid2, pid3, ...])
```

---

## The Full Memory Picture

```
Parent process: 200MB RSS
    ├── 200MB model dict (Python objects)
    │   ├── dict keys: Python str objects (each ~50-100 bytes + ob_refcnt)
    │   ├── dict values: np.ndarray objects (Python wrapper + data buffer)
    │   └── Data buffers: raw float32 bytes (no ob_refcnt — C allocated)
    └── Interpreter overhead: ~5MB

After fork (t=0):
    8 workers × ~200MB physical shared pages (no physical copy yet)
    OS marks all pages as CoW read-only.
    Physical RAM used: ~200MB (shared)

After 5 minutes of requests (t=5min):
    Each worker has dirtied ~70% of model pages (ob_refcnt mutations)
    Each worker's Private_Dirty ≈ 140MB (copied from 200MB model pages)
    Each worker's Shared_Clean ≈ 60MB (pages not yet touched)
    Per-worker heap growth: +50MB (request/response + interpreter)
    Total per-worker RSS: 200MB + 50MB = 250MB ✓
    
    Total physical RAM: 8 × 200MB = 1.6GB (vs 200MB if truly shared)
    Wasted RAM from CoW: 1.4GB
```

## Common Mistakes and Traps

1. **Thinking fork() shares memory forever.** It shares pages until any write happens. In Python, "writing" includes refcount updates which happen constantly.

2. **Assuming numpy arrays avoid the problem entirely.** The ndarray object itself (the Python wrapper) has ob_refcnt. Putting an ndarray in a dict means the dict lookup touches ob_refcnt on the ndarray. The raw float data buffer avoids refcounting, but the Python-level dict lookup does not.

3. **Loading the model AFTER forking** (common Gunicorn pattern with `preload_app = False`). Each worker loads its own copy → no sharing at all. This is actually WORSE than the CoW problem.

4. **Using `multiprocessing.Value` or `multiprocessing.Array` for large arrays.** These have synchronization overhead (locks) inappropriate for read-only data at high throughput.

5. **Measuring RSS and concluding "8 workers × 250MB = 2GB used."** PSS (Proportional Set Size) is more accurate: Shared pages are counted once, divided by the number of processes sharing them. Total PSS ≈ RSS_worker_unique × 8 + RSS_shared.

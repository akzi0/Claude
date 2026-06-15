# Lesson 09: Language Runtime Internals — CPython, GC, JIT, and the Machinery Beneath Python

> **Level:** CPython internals / language runtime researcher  
> **Prerequisites:** Lessons 04 (Virtual Memory), 05 (Compiler Internals)  
> **References:** CPython `ceval.c`, `gcmodule.c`, `obmalloc.c`; PyPy — Bolz et al. 2009, "Tracing the Meta-Level"; PEP 659 (Specializing Adaptive Interpreter); PEP 703 (No-GIL)

---

## 1. CPython Object Model

Every Python object — integer, string, list, function, class — is a C struct in memory. The minimum structure is **`PyObject`**:

```c
/* Include/object.h */
typedef struct _object {
    Py_ssize_t ob_refcnt;      /* reference count — 8 bytes on 64-bit */
    PyTypeObject *ob_type;     /* pointer to type descriptor — 8 bytes */
} PyObject;
```

Variable-length objects (strings, lists, tuples) extend this with `PyVarObject`, which adds `ob_size` (the number of items). An integer (`PyLongObject`) in CPython ≤ 3.11 looks like:

```c
typedef struct _PyLongObject {
    PyObject_VAR_HEAD       /* ob_refcnt, ob_type, ob_size */
    digit ob_digit[1];      /* flexible array of 30-bit digits */
} PyLongObject;
```

In CPython 3.12+, integers were restructured into a compact representation where small values fit in the header itself (avoiding the heap allocation for `ob_digit`).

### Inspecting the struct layout with ctypes

```python
import ctypes, sys

x = [1, 2, 3]
addr = id(x)  # id() IS the memory address in CPython

# Read ob_refcnt at the start of the struct
refcnt = ctypes.cast(addr, ctypes.POINTER(ctypes.c_long))[0]
print(f"ob_refcnt at {hex(addr)}: {refcnt}")
print(f"sys.getrefcount(x): {sys.getrefcount(x)}")  # getrefcount itself adds 1

# Read ob_type pointer (8 bytes after ob_refcnt on 64-bit)
ob_type_ptr = ctypes.cast(addr + 8, ctypes.POINTER(ctypes.c_ulong))[0]
print(f"ob_type pointer: {hex(ob_type_ptr)}")
print(f"type(x): {type(x)}")
print(f"id(list): {hex(id(list))}")  # should match ob_type_ptr
```

`sys.getrefcount(x)` returns N+1 because the argument passing to `getrefcount` creates a temporary reference. The raw `ctypes` read returns N, the actual live count.

### PyTypeObject — the type descriptor

Every type (int, str, list, your custom class) is a `PyTypeObject`. This struct has ~100 fields including:

| Field | Purpose |
|---|---|
| `tp_name` | C string: `"int"`, `"list"` |
| `tp_basicsize` | sizeof the object |
| `tp_dealloc` | called when refcount hits 0 |
| `tp_repr` | `__repr__` |
| `tp_hash` | `__hash__` |
| `tp_call` | `__call__` |
| `tp_as_number` | pointer to `PyNumberMethods` struct |
| `tp_as_sequence` | pointer to `PySequenceMethods` |
| `tp_dict` | the class `__dict__` |
| `tp_bases` | base classes tuple |
| `tp_mro` | method resolution order tuple |

`tp_as_number` is a pointer to a `PyNumberMethods` struct that contains function pointers for `nb_add`, `nb_multiply`, `nb_subtract`, `nb_power`, etc. When you write `a + b`, CPython calls `a->ob_type->tp_as_number->nb_add(a, b)`.

### Method Resolution Order — C3 Linearization

The MRO stored in `tp_mro` is computed once at class creation time using the C3 linearization algorithm. It is never recomputed on attribute access (unlike naïve depth-first search in old-style classes).

```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass

print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
```

C3 ensures that: (1) a class always precedes its parents, (2) the order of parents in the inheritance list is preserved, (3) if a class appears in multiple parents' MROs, it appears at the latest possible position. Violations raise `TypeError: Cannot create a consistent method resolution order`.

### Small Integer Cache and String Interning

CPython pre-allocates a static array of `PyLongObject` for integers −5 through 256. These are singletons:

```python
a = 256; b = 256
print(a is b)   # True — same object

a = 257; b = 257
print(a is b)   # False — two allocations (usually; REPL may intern in same code block)

# See the cache boundary
print(id(255) == id(255))  # True
print(id(257) == id(257))  # May be False
```

**Why 256?** It is a pragmatic choice — single-byte ASCII values and loop indices up to 256 are extremely common. Pre-allocating them eliminates malloc/free churn for the most-used integers.

String interning: CPython automatically interns string literals that look like identifiers (match `[A-Za-z_][A-Za-z0-9_]*`). You can force interning with `sys.intern()`. Interned strings are stored in a dict; identity comparison (`is`) replaces equality comparison (`==`), which is O(1) vs O(n) for long strings.

```python
import sys
a = 'hello_world'
b = 'hello_world'
print(a is b)   # Likely True — auto-interned (identifier-like)

a = 'hello world'   # space — not auto-interned
b = 'hello world'
print(a is b)   # Likely False

a = sys.intern('hello world')
b = sys.intern('hello world')
print(a is b)   # True — forced into intern pool
```

---

## 2. CPython Bytecode and the Evaluation Loop

### The Code Object (`PyCodeObject`)

When CPython compiles a function, it produces a **code object**:

```python
import dis, sys

def sum_squares(n):
    total = 0
    for i in range(n):
        total += i * i
    return total

code = sum_squares.__code__
print(f"co_varnames:  {code.co_varnames}")   # ('n', 'total', 'i')
print(f"co_consts:    {code.co_consts}")     # (None, 0)
print(f"co_stacksize: {code.co_stacksize}")  # max stack depth needed
print(f"co_flags:     {hex(code.co_flags)}") # bitmask: generator, nested, etc.
print(f"co_freevars:  {code.co_freevars}")   # variables from enclosing scope
print(f"co_cellvars:  {code.co_cellvars}")   # variables captured by inner scopes
```

### Frame Object (`PyFrameObject`)

Every function call creates a **frame object** that is the runtime execution context:

```python
import inspect

def outer():
    x = 10
    def inner():
        frame = inspect.currentframe()
        print(f"f_code:    {frame.f_code.co_name}")
        print(f"f_locals:  {frame.f_locals}")
        print(f"f_back:    {frame.f_back.f_code.co_name}")  # outer
        print(f"f_lasti:   {frame.f_lasti}")   # byte offset of last instruction
    inner()

outer()
```

The frame chain (`f_back`) is Python's call stack. `traceback.extract_stack()` walks this chain. In CPython 3.11+, frames were restructured ("frame objects on demand") — the internal `_PyInterpreterFrame` is stack-allocated when possible, and a heap-allocated `PyFrameObject` is only created if you call `sys._getframe()` or access `__frame__`.

### Disassembling Bytecode

```python
import dis

def sum_squares(n):
    total = 0
    for i in range(n):
        total += i * i
    return total

dis.dis(sum_squares)
```

Example output on CPython 3.12:
```
  2           RESUME          0
  3           LOAD_CONST      1 (0)
              STORE_FAST      1 (total)
  4           LOAD_GLOBAL     1 (NULL + range)
              LOAD_FAST       0 (n)
              CALL            1
              GET_ITER
  5     >>    FOR_ITER        ...
              STORE_FAST      2 (i)
  6           LOAD_FAST       2 (i)
              LOAD_FAST       2 (i)
              BINARY_OP       5 (*)
              ...
              BINARY_OP       13 (+=)
              STORE_FAST      1 (total)
              JUMP_BACKWARD   ...
  7           RETURN_VALUE
```

**Key instructions:**
- `LOAD_FAST` — load a local variable (array index into `f_localsplus`, O(1))
- `LOAD_GLOBAL` — dict lookup in `f_globals` then `f_builtins` (O(1) hash, but slower than LOAD_FAST)
- `STORE_FAST` — store to local slot
- `BINARY_OP` — dispatches to `tp_as_number->nb_add` etc. (CPython 3.11+ unified opcode)
- `CALL` — CPython 3.11+ unified call instruction (replaces `CALL_FUNCTION`, `CALL_METHOD`, etc.)
- `FOR_ITER` — calls `__next__` on the iterator; jumps forward on `StopIteration`

### The Evaluation Loop — `ceval.c`

The heart of CPython is the `_PyEval_EvalFrameDefault()` function in `Python/ceval.c`. It is an infinite loop dispatching on opcodes:

```c
/* Simplified pseudocode of ceval.c */
for (;;) {
    opcode = NEXTOPARG();
    switch (opcode) {
        case LOAD_FAST: {
            PyObject *value = GETLOCAL(oparg);
            Py_INCREF(value);
            PUSH(value);
            DISPATCH();
        }
        case BINARY_OP: {
            PyObject *right = POP();
            PyObject *left  = TOP();
            PyObject *res   = binary_ops[oparg](left, right);
            SET_TOP(res);
            DISPATCH();
        }
        /* ... ~150 more opcodes ... */
    }
}
```

**CPython 3.11+ computed gotos:** GCC supports a non-standard extension — `void *dispatch_table[] = {&&LOAD_FAST, &&BINARY_OP, ...}` where `&&label` is the address of a label. The `DISPATCH()` macro becomes `goto *dispatch_table[opcode]`. This avoids the branch predictor pollution of `switch`/`case` and gives ~20% speedup on typical code. MSVC does not support this; on Windows, CPython falls back to `switch`.

### Specializing Adaptive Interpreter (PEP 659, CPython 3.11+)

After a bytecode instruction executes **8 times**, the interpreter replaces it with a specialized variant:

```
LOAD_ATTR  →  LOAD_ATTR_INSTANCE_VALUE   (direct slot offset, no dict lookup)
             LOAD_ATTR_MODULE             (module-level global, cached)
BINARY_OP  →  BINARY_OP_ADD_INT          (fast path: no type check needed)
             BINARY_OP_ADD_FLOAT
CALL       →  CALL_PY_EXACT_ARGS         (known Python function, no *args/**kwargs)
```

If a guard fails (e.g., the object is no longer an int), the instruction **despecializes** back to the generic form and the counter restarts. This is analogous to inline caching in V8/SpiderMonkey, implemented directly in the bytecode stream.

```python
import dis

def add_ints(a, b):
    return a + b

# Force specialization by running it
for _ in range(20):
    add_ints(1, 2)

# On 3.12+ you can inspect the adaptive bytecode
# (requires debug build or sys._getframe tricks)
dis.dis(add_ints)
```

**Frame allocation cost:** Each Python function call allocates a `_PyInterpreterFrame` of ~200–400 bytes and does significant bookkeeping: argument binding, cell variable setup, generator/coroutine detection. A Python function call takes ~100–200 ns. A C function call is ~1 ns. This is why tight loops in Python use built-in functions (`sum`, `map`, `sorted`) over Python-level loops.

---

## 3. The GIL — What It Actually Does

The **Global Interpreter Lock** is a `PyMutex` (wrapping a platform mutex) that must be held to execute Python bytecode. CPython's internal structures — object refcounts, the memory allocator, the import system, the garbage collector — are **not thread-safe** in isolation. The GIL makes them safe by serializing all access.

### What the GIL protects and doesn't protect

**Protected:** `Py_INCREF`/`Py_DECREF` (refcount updates), `pymalloc` arena bookkeeping, the `PyEval_EvalFrameDefault` loop, builtin type operations.

**NOT protected:** Python-level data structures from your perspective as a programmer. The GIL ensures bytecodes execute atomically at the C level, but not that Python-level operations are atomic:

```python
# DANGEROUS: NOT atomic even with the GIL
d = {}
def writer():
    for i in range(10000):
        d[i] = i   # may interleave with reader

def reader():
    for i in range(10000):
        try:
            _ = list(d.items())   # RuntimeError: dictionary changed size during iteration
        except RuntimeError:
            pass
```

### GIL release for I/O and C extensions

The GIL IS released during:
- Blocking I/O: `socket.recv()`, `file.read()` — C code calls `Py_BEGIN_ALLOW_THREADS` before the syscall
- NumPy array operations on non-Python objects
- `time.sleep()`
- Any C extension that calls `Py_BEGIN_ALLOW_THREADS` explicitly

This is why threading helps I/O-bound Python but not CPU-bound Python:

```python
import threading, time

def cpu_bound(n):
    total = 0
    for i in range(n):
        total += i * i
    return total

N = 5_000_000

# Single-threaded
t0 = time.perf_counter()
cpu_bound(N)
cpu_bound(N)
single = time.perf_counter() - t0

# Two threads (both want the GIL — only one runs at a time)
t0 = time.perf_counter()
t1 = threading.Thread(target=cpu_bound, args=(N,))
t2 = threading.Thread(target=cpu_bound, args=(N,))
t1.start(); t2.start()
t1.join(); t2.join()
threaded = time.perf_counter() - t0

print(f"Single-threaded:  {single:.3f}s")
print(f"Two threads (GIL): {threaded:.3f}s")  # ~same or SLOWER (GIL thrashing)

# Multiprocessing — separate GILs, true parallelism
from multiprocessing import Pool
t0 = time.perf_counter()
with Pool(2) as p:
    p.map(cpu_bound, [N, N])
parallel = time.perf_counter() - t0
print(f"Two processes:    {parallel:.3f}s")  # ~50% of single-threaded
```

### GIL check interval

Every `sys.getswitchinterval()` seconds (default 5ms = 0.005s), the GIL holder checks `eval_breaker` — a flag set by threads waiting for the GIL, signal handlers, and the GC. If set, the thread drops the GIL, allowing others to acquire it. The OS scheduler then decides which waiting thread gets it next — CPython has no control over fairness.

### CPython 3.13 No-GIL (PEP 703)

Python 3.13 ships an experimental free-threaded build (`python3.13t`). Key mechanisms:
- **Per-object biased locking:** Each object has a per-object lock. The first thread to acquire it "biases" the lock to itself; subsequent acquires by the same thread are lock-free. Cross-thread access triggers the slow path.
- **Immortal objects (PEP 683):** Singletons like `None`, `True`, `False`, small integers — never freed, so their refcounts don't need protection.
- **Mimalloc:** replaced `pymalloc` with Microsoft's mimalloc, which is thread-safe and cache-friendly.

This is the largest internal change to CPython in its 30-year history. Many C extensions required rewriting to become thread-safe.

---

## 4. Reference Counting — The Real Algorithm

### The mechanics

`Py_INCREF(op)` and `Py_DECREF(op)` are macros (inlined for speed):

```c
/* Simplified */
#define Py_INCREF(op) (((PyObject*)(op))->ob_refcnt++)
#define Py_DECREF(op)                          \
    do {                                        \
        if (--((PyObject*)(op))->ob_refcnt == 0) \
            (((PyObject*)(op))->ob_type->tp_dealloc)(op); \
    } while(0)
```

In CPython 3.12+ free-threaded mode, these become atomic operations (`_Py_atomic_add_ssize_t`), which is slower but safe.

### Immediate deallocation

When refcount hits 0, `tp_dealloc` is called **immediately and synchronously**. For a list, `list_dealloc` decrements refcounts of all contained objects (which may chain-deallocate). For a dict, `dict_dealloc`. This gives CPython deterministic finalization — `__del__` runs as soon as the last reference is dropped (except for cycles).

```python
import sys

class Tracked:
    def __del__(self):
        print(f"Freed {self.name}")
    def __init__(self, name):
        self.name = name

a = Tracked("A")
b = a
print(sys.getrefcount(a))   # 3 (a, b, getrefcount arg)
del a
print("After del a")        # no deallocation yet — b still holds ref
del b
# Output: "Freed A" — immediately, here, on this line
```

### The circular reference problem

```python
import gc, sys

a = []
a.append(a)    # a refers to itself

print(sys.getrefcount(a))   # 3: variable a, the list's own element, getrefcount

del a
# Refcount of the list is now 1 (its own element still holds a reference)
# The list will NEVER reach refcount 0 via Py_DECREF alone
# It is unreachable garbage — a memory leak without the cycle collector
```

The cycle collector (`gc` module, `Modules/gcmodule.c`) exists solely to handle this case. Regular (acyclic) objects are freed by refcounting alone — the GC never sees them.

```python
import gc

class Node:
    def __init__(self, val):
        self.val = val
        self.next = None
    def __del__(self):
        print(f"  __del__ called: Node({self.val})")

gc.disable()
print("Creating cycle:")
a = Node(1)
b = Node(2)
a.next = b
b.next = a   # cycle: a → b → a

print(f"gc.get_count() before del: {gc.get_count()}")
del a, b
print(f"gc.get_count() after del:  {gc.get_count()}")
print("Objects still alive (cycle not collected):")

n = gc.collect()
print(f"gc.collect() freed {n} objects")
```

---

## 5. Generational Garbage Collection

### Three generations

CPython maintains three **generations** of container objects (objects that can hold references to other objects: lists, dicts, sets, instances with `__dict__`):

```python
import gc

print(gc.get_threshold())  # (700, 10, 10)
print(gc.get_count())      # (young, mid, old) allocation deltas
```

- **Generation 0 (young):** newly allocated objects. Collected when `(allocs - deallocs) > 700`.
- **Generation 1 (middle):** survived one gen0 collection. Collected after gen0 runs 10 times.
- **Generation 2 (old):** survived a gen1 collection. Collected after gen1 runs 10 times.

**The generational hypothesis:** Most objects die young (they are temporaries in function calls, list comprehension elements, etc.). Scanning only gen0 — which is small and recently touched (cache-warm) — is far cheaper than scanning the entire heap.

### The collection algorithm (mark phase)

`gcmodule.c` implements a variant of the **tricolor marking** algorithm:

1. **Copy refcounts:** For every object O in the collected generation, set a shadow count `gc_refs = ob_refcnt`.
2. **Subtract internal references:** For each object O, traverse its references. For each referenced object R *also in the generation*, decrement `R->gc_refs`.
3. **After step 2:** Objects with `gc_refs > 0` are referenced from *outside* the generation (reachable roots). Objects with `gc_refs == 0` are candidates for collection.
4. **Propagate reachability:** From the roots, follow all references. Mark every reachable object as alive.
5. **Collect:** Objects still with `gc_refs == 0` are garbage. Restore refcounts of live objects. Call `tp_dealloc` on garbage.

```python
import gc, tracemalloc

tracemalloc.start()
gc.set_debug(gc.DEBUG_STATS)

def create_cycle():
    class Cycle:
        pass
    c = Cycle()
    c.self_ref = c
    # c goes out of scope — refcount drops to 1 (self_ref holds it)
    # Only gc.collect() can free this

for _ in range(1000):
    create_cycle()

before = tracemalloc.take_snapshot()
gc.collect()
after = tracemalloc.take_snapshot()

stats = after.compare_to(before, 'lineno')
print("Top memory changes after collect:")
for stat in stats[:5]:
    print(f"  {stat}")
```

### Measuring GC pressure

```python
import gc, time

# Allocate many objects that survive gen0 and get promoted
def measure_gc():
    gc.collect()   # start clean
    gc.set_debug(0)
    
    t0 = time.perf_counter()
    survivors = []
    for _ in range(100_000):
        node = {"val": _, "children": []}
        survivors.append(node)   # keeps alive, promotes to gen1/gen2
    
    elapsed = time.perf_counter() - t0
    stats = gc.get_stats()
    print(f"Allocation time: {elapsed*1000:.1f}ms")
    print(f"GC stats: {stats}")
    print(f"Gen counts: {gc.get_count()}")

measure_gc()
```

---

## 6. asyncio Internals

### The event loop is a single-threaded I/O dispatcher

`asyncio` does not provide parallelism — it provides **concurrency** via cooperative multitasking. The event loop wraps `select()`/`epoll()`/`kqueue()` (via the `selectors` module) and dispatches callbacks when file descriptors become ready.

```
Event Loop
│
├── _ready queue: callbacks to run immediately
├── _scheduled heap: callbacks with delays (heapq sorted by time)
└── selector: maps fd → callback (waiting for I/O)

Each iteration (_run_once):
1. Pop all _ready callbacks, run them
2. Calculate timeout: min(next scheduled callback time, I/O poll time)
3. Call selector.select(timeout) → get list of ready (fd, event) pairs
4. Schedule the callbacks for those fds into _ready
5. Run any _scheduled callbacks whose time has come
```

### Coroutines are generator-based state machines

```python
import dis

async def fetch(url):
    await asyncio.sleep(1)
    return f"data from {url}"

# A coroutine function returns a coroutine object when called
coro = fetch("http://example.com")
print(type(coro))   # <class 'coroutine'>

# The coroutine compiles to bytecode with GET_AWAITABLE and YIELD_VALUE
dis.dis(fetch)
```

`async def` functions compile to code with `CO_COROUTINE` flag. `await expr` compiles to `GET_AWAITABLE` + `YIELD_VALUE` + `RESUME`. This means a coroutine is literally a generator that yields `Future` objects up to the event loop, which registers them with the selector and resumes the coroutine when the I/O completes.

### Tasks and the await chain

```python
import asyncio

async def step_a():
    print("A: start")
    await asyncio.sleep(0.1)   # yields to event loop
    print("A: after sleep")
    return "A"

async def step_b():
    print("B: start")
    await asyncio.sleep(0.05)
    print("B: after sleep")
    return "B"

async def main():
    # Sequential: A finishes before B starts
    ra = await step_a()
    rb = await step_b()
    print(f"Sequential: {ra}, {rb}")

    # Concurrent: both scheduled, interleaved at awaits
    ta = asyncio.create_task(step_a())
    tb = asyncio.create_task(step_b())
    ra, rb = await asyncio.gather(ta, tb)
    print(f"Concurrent: {ra}, {rb}")

asyncio.run(main())
```

When `step_a` hits `await asyncio.sleep(0.1)`, it calls `loop.call_later(0.1, callback)` and yields `None` to the event loop. The event loop is now free to run `step_b`. After 0.05s, `step_b` completes its sleep and resumes. After 0.1s, `step_a` resumes. Total wall time: ~0.1s, not ~0.15s.

### Minimal event loop from scratch

```python
import selectors, heapq, time, types

class SimpleEventLoop:
    def __init__(self):
        self.sel = selectors.DefaultSelector()
        self._ready = []          # deque of (callback, args)
        self._scheduled = []      # heapq of (when, callback, args)

    def call_soon(self, callback, *args):
        self._ready.append((callback, args))

    def call_later(self, delay, callback, *args):
        when = time.monotonic() + delay
        heapq.heappush(self._scheduled, (when, callback, args))

    def _run_once(self):
        # Fire scheduled callbacks that are due
        now = time.monotonic()
        while self._scheduled and self._scheduled[0][0] <= now:
            _, cb, args = heapq.heappop(self._scheduled)
            self._ready.append((cb, args))

        # Determine I/O poll timeout
        timeout = None
        if self._scheduled:
            timeout = max(0.0, self._scheduled[0][0] - time.monotonic())
        if self._ready:
            timeout = 0.0

        # Poll I/O
        events = self.sel.select(timeout)
        for key, mask in events:
            callback, args = key.data
            self._ready.append((callback, args))

        # Run all ready callbacks
        ntodo = len(self._ready)
        for _ in range(ntodo):
            cb, args = self._ready.pop(0)
            cb(*args)

    def run_until_complete(self, coro):
        # Wrap coroutine in a simple task driver
        def step(coro, future=None):
            try:
                # Send the result of the awaited future back in
                yielded = coro.send(None)
                # yielded is a Future or sleep handle
                if hasattr(yielded, '_when'):   # asyncio.TimerHandle-like
                    self.call_later(yielded._when - time.monotonic(),
                                    step, coro)
            except StopIteration as e:
                print(f"Coroutine returned: {e.value}")

        self.call_soon(step, coro)
        while self._ready or self._scheduled:
            self._run_once()
```

This captures the essential structure of `asyncio`'s `BaseEventLoop._run_once()` in ~40 lines.

---

## 7. PyPy JIT Internals

### Architecture

PyPy is not CPython with a JIT bolted on. It is a separate Python implementation written in **RPython** (Restricted Python — a statically-typed subset of Python). RPython is compiled by the RPython toolchain to C, which is then compiled to native machine code. The JIT compiler is *generated* semi-automatically from the interpreter by the toolchain.

Reference: **Bolz, Cuni, Fijalkowski, Rigo — "Tracing the Meta-Level: PyPy's Tracing JIT Compiler" (2009, ICOOOLPS).**

### Tracing JIT

PyPy uses a **tracing JIT**, not a method JIT (like Java's HotSpot). The distinction:
- **Method JIT** (HotSpot): compiles entire methods when they become hot
- **Tracing JIT** (PyPy, LuaJIT): identifies hot **loops**, records a linear trace of one iteration's execution, compiles that trace

When a loop back-edge executes **1039 times** (the JIT threshold), PyPy enters **tracing mode**. It executes one more iteration while recording every operation — Python bytecode dispatch, dict lookups, type checks — as a flat sequence of operations (the "trace"). The trace is then compiled to native machine code.

### Guards and deoptimization

The compiled trace contains **guards** at every point where the trace assumed something about the program state:

```
# Trace for: total += i * i  (assuming i is int, total is int)
guard(type(i) == int)          # guard_class
guard(type(total) == int)      # guard_class
r1 = int_mul(i, i)
guard_no_overflow()            # guard_overflow — Python ints are arbitrary precision
r2 = int_add(total, r1)
guard_no_overflow()
total = r2
```

If a guard fails — say `i` is now a float — execution **deoptimizes** back to the interpreter from the point of failure. The interpreter continues from there, and PyPy tracks how often each guard fails.

### Bridges and meta-tracing

When a guard fails **frequently** (threshold: ~1039 times), PyPy traces from the guard failure point as a new trace — a **bridge**. The bridge jumps back into the main loop's compiled code when it can. This gives PyPy near-optimal code for polymorphic loops.

**Meta-tracing** is the key insight of Bolz et al.: PyPy traces the **interpreter** executing Python bytecode. The trace therefore includes the interpreter's type dispatch, dictionary lookups for attributes, and function call machinery — and the JIT can eliminate all of it when types are monomorphic. This is why PyPy achieves 5–50× speedups over CPython on numeric code.

### Numba as a practical JIT for CPython users

CPython does not have a built-in JIT (as of 3.12). Numba provides tracing JIT compilation for NumPy-heavy numerical code:

```python
from numba import njit
import numpy as np, time

@njit(cache=True)
def sum_squares_jit(n):
    total = 0.0
    for i in range(n):
        total += i * i
    return total

def sum_squares_py(n):
    total = 0.0
    for i in range(n):
        total += i * i
    return total

N = 10_000_000

# Warm up JIT
sum_squares_jit(1000)

t0 = time.perf_counter()
sum_squares_py(N)
py_time = time.perf_counter() - t0

t0 = time.perf_counter()
sum_squares_jit(N)
jit_time = time.perf_counter() - t0

print(f"CPython:  {py_time*1000:.1f}ms")
print(f"Numba JIT: {jit_time*1000:.1f}ms")
print(f"Speedup: {py_time/jit_time:.1f}×")
```

Numba compiles the function to LLVM IR, which LLVM optimizes and compiles to native code — similar to what PyPy's JIT backend does.

---

## 8. CPython Memory Allocator

### pymalloc — arena-based allocator

For objects ≤ 512 bytes (the vast majority of Python objects), CPython bypasses `malloc()` and uses its own **pymalloc** (`Objects/obmalloc.c`):

```
Arena (256 KB)
├── Pool 0 (4 KB) — size class 8 bytes  — 512 blocks
├── Pool 1 (4 KB) — size class 8 bytes
├── Pool 2 (4 KB) — size class 16 bytes — 256 blocks
├── Pool 3 (4 KB) — size class 24 bytes — 170 blocks
│   ...
└── Pool 63 (4 KB) — size class 512 bytes — 8 blocks
```

**Size classes:** 8, 16, 24, 32, ..., 512 bytes in steps of 8 (64 classes). A request for 17 bytes is served from the 24-byte pool. This eliminates fragmentation *within* a size class and makes allocation O(1).

**Free list recycling:** Freed blocks go onto a per-pool free list (a linked list stored in the block itself). The next allocation from that pool pops from the free list — no zeroing, no system call.

### Memory fragmentation and arena retention

```python
import tracemalloc, gc

tracemalloc.start()

# Create many small objects
objs = [{"key": i, "val": i * 2} for i in range(100_000)]
snap1 = tracemalloc.take_snapshot()

# Delete half — but if they're scattered across arenas,
# those arenas cannot be returned to the OS
del objs[::2]
gc.collect()
snap2 = tracemalloc.take_snapshot()

print("Top allocations remaining:")
for stat in snap2.statistics('lineno')[:5]:
    print(f"  {stat}")
```

**The fragmentation trap:** pymalloc cannot release an arena to the OS until **every pool in that arena is completely empty**. If even one object survives in an arena, the entire 256KB stays in process memory. Long-running servers accumulate fragmented arenas — this is why Python processes often have high RSS (Resident Set Size) that never shrinks.

**Solutions:**
- `gc.collect()` after bulk deletion
- Use `__slots__` (see section 9) to reduce per-object footprint
- Restart workers periodically (gunicorn's `--max-requests`)
- Use `jemalloc` or `mimalloc` as the base allocator (CPython 3.13 free-threaded build uses mimalloc)

### Tracking allocations with tracemalloc

```python
import tracemalloc, linecache

tracemalloc.start(25)   # keep 25-frame tracebacks

def leaky():
    # Simulated leak: growing cache, never pruned
    cache = {}
    for i in range(10_000):
        cache[f"key_{i}"] = list(range(100))
    return cache

c = leaky()

snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('traceback')

print("Top 3 memory allocations:")
for stat in top_stats[:3]:
    print(f"\n{stat.size / 1024:.1f} KiB — {stat.count} allocations")
    for line in stat.traceback.format():
        print(f"  {line}")
```

---

## 9. What Textbooks Don't Tell You

### The GIL doesn't protect your data structures

As shown in section 3, the GIL serializes bytecode execution — but many Python-level operations (list append, dict update, object attribute set) compile to *multiple* bytecodes. Another thread can be scheduled between any two bytecodes. The GIL does NOT make `dict[key] = value` atomic from the perspective of iterating that dict in another thread.

Use `threading.Lock()` for any shared mutable state accessed from multiple threads.

### `id()` reuse after deallocation

```python
a = object()
addr_a = id(a)
del a
b = object()
print(id(b) == addr_a)   # Possibly True!
```

`id(x)` is the memory address of `x`. After `del a`, the address may be immediately reused for `b`. If you store `id()` values for later comparison, ensure the original object is still alive (e.g., hold a reference).

### `__slots__` — eliminating `__dict__` overhead

By default, every Python instance has a `__dict__` (a dict) that stores its attributes. This costs ~200–400 bytes per object (the dict object itself, plus its hash table).

```python
import sys

class WithDict:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class WithSlots:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y

d = WithDict(1, 2)
s = WithSlots(1, 2)

print(f"WithDict:   {sys.getsizeof(d)} bytes + {sys.getsizeof(d.__dict__)} (dict) = "
      f"{sys.getsizeof(d) + sys.getsizeof(d.__dict__)} total")
print(f"WithSlots:  {sys.getsizeof(s)} bytes (no __dict__)")

# At 1 million instances:
N = 1_000_000
saving_per_obj = sys.getsizeof(d.__dict__)  # ~200-400 bytes
print(f"Memory saved for {N:,} objects: {saving_per_obj * N / 1024**2:.0f} MB")
```

`__slots__` stores attributes in a fixed array (like C struct fields), not a dict. Attribute access is an O(1) offset lookup vs O(1) hash lookup — both fast, but slots avoid the 200-byte dict overhead.

**Limitation:** `__slots__` breaks pickling unless you implement `__getstate__`/`__setstate__`, and prevents dynamic attribute assignment. Use when you have many instances of a fixed-schema class.

### Copy-on-write and forked Python processes

When Python forks (e.g., with `multiprocessing`), the OS uses copy-on-write — child and parent share physical pages until one writes. But CPython's reference counting **writes `ob_refcnt` on every object access**. Reading a module-level constant increments its refcount — causing a CoW page fault. A freshly forked worker immediately dirtifies a huge fraction of the parent's heap.

Instagram engineering solved this with `gc.freeze()` (added in Python 3.7): freeze all current objects, making them permanently immortal (no refcount updates). Forked workers share these pages with zero CoW.

```python
import gc

# Before forking: freeze all current objects
gc.freeze()
print(f"Frozen objects: {gc.get_freeze_count()}")

# Now fork — frozen objects won't cause CoW
import os
pid = os.fork()
if pid == 0:
    # Child: forked pages for frozen objects are shared
    print(f"Child PID {os.getpid()}")
    os._exit(0)
```

### Object resurrection in `__del__`

```python
class Zombie:
    _alive = None
    
    def __del__(self):
        print("__del__ called — resurrecting!")
        Zombie._alive = self   # create a new reference to self
        # CPython detects this and does NOT free the object
        # __del__ will never be called again for this object

z = Zombie()
del z
print(f"Zombie._alive: {Zombie._alive}")   # the object survived!
print(f"__del__ won't fire again: {Zombie._alive is not None}")
Zombie._alive = None   # now the object is truly freed (silently)
```

CPython checks after `tp_dealloc` calls `__del__` whether `ob_refcnt` is still 0. If resurrection occurred, it sets `ob_refcnt = 1` and does not free the object. The `__del__` method is cleared (set to None) so it won't fire again, preventing infinite resurrection loops.

---

## 10. Hard Exercises

### (a) GIL benchmark: threads vs processes

```python
import threading, multiprocessing, time

def cpu_sum(n):
    return sum(i * i for i in range(n))

N = 8_000_000

def run_threaded():
    results = []
    lock = threading.Lock()
    def worker():
        r = cpu_sum(N)
        with lock:
            results.append(r)
    t1 = threading.Thread(target=worker)
    t2 = threading.Thread(target=worker)
    t0 = time.perf_counter()
    t1.start(); t2.start()
    t1.join(); t2.join()
    return time.perf_counter() - t0

def run_sequential():
    t0 = time.perf_counter()
    cpu_sum(N)
    cpu_sum(N)
    return time.perf_counter() - t0

def run_parallel():
    t0 = time.perf_counter()
    with multiprocessing.Pool(2) as p:
        p.map(cpu_sum, [N, N])
    return time.perf_counter() - t0

print(f"Sequential:   {run_sequential():.3f}s  (baseline)")
print(f"2 Threads:    {run_threaded():.3f}s   (expect: same or worse due to GIL)")
print(f"2 Processes:  {run_parallel():.3f}s   (expect: ~50% of sequential)")
```

**Expected results:** Sequential ≈ Threaded (GIL prevents true parallelism; may be slower due to GIL acquisition overhead). Parallel ≈ Sequential / 2 (two CPUs, no GIL contention).

### (b) Memory leak analysis with tracemalloc

```python
import tracemalloc, gc

cache = {}

def process(key, data):
    if key not in cache:
        result = list(range(len(data)))
        cache[key] = result
    return cache[key]

tracemalloc.start()
snap1 = tracemalloc.take_snapshot()

for i in range(1000):
    process(f"key_{i}", list(range(1000)))   # 1000 unique keys → 1000 cache entries

snap2 = tracemalloc.take_snapshot()
top = snap2.compare_to(snap1, 'lineno')
print("Memory growth:")
for stat in top[:5]:
    print(f"  {stat}")
```

**Root cause:** `cache` is a module-level dict that grows without bound. `process(key, data)` stores `list(range(len(data)))` for every unique key — never evicted. With 1000 calls on data of size 1000, this allocates 1000 × 1000 × 8 bytes = ~8MB in lists alone, plus dict overhead.

**Fix:** Use `functools.lru_cache`, or bound the cache with `collections.OrderedDict` + manual eviction, or use `cachetools.LRUCache`.

### (c) Deep object size measurement

```python
import sys

def object_size(obj, seen=None):
    """Return total memory used by obj and all objects it transitively references."""
    if seen is None:
        seen = set()
    
    obj_id = id(obj)
    if obj_id in seen:
        return 0   # already counted — handles circular references
    seen.add(obj_id)
    
    size = sys.getsizeof(obj)
    
    # Recurse into containers
    if isinstance(obj, dict):
        size += sum(object_size(k, seen) + object_size(v, seen)
                    for k, v in obj.items())
    elif isinstance(obj, (list, tuple, set, frozenset)):
        size += sum(object_size(item, seen) for item in obj)
    elif hasattr(obj, '__dict__'):
        size += object_size(obj.__dict__, seen)
    elif hasattr(obj, '__slots__'):
        size += sum(object_size(getattr(obj, slot, None), seen)
                    for slot in obj.__slots__ if hasattr(obj, slot))
    
    return size

# Test it
x = {"key": [1, 2, 3, {"nested": "value"}]}
print(f"sys.getsizeof: {sys.getsizeof(x)} bytes (immediate only)")
print(f"object_size:   {object_size(x)} bytes (full graph)")

# Circular reference test
a = []
b = [a]
a.append(b)
print(f"Circular: {object_size(a)} bytes (terminates correctly)")
```

### (d) Parent–Child cycle analysis

```python
import gc, sys

class Parent:
    def __init__(self):
        self.child = Child(self)
    def __del__(self):
        print("Parent freed")

class Child:
    def __init__(self, parent):
        self.parent = parent
    def __del__(self):
        print("Child freed")

gc.disable()
p = Parent()
# Memory graph: p (refcnt=1) → child (refcnt=1) → parent=p (refcnt=2)
#               p.child (refcnt=1 from p.__dict__)
# After del p: p refcnt drops to 1 (child.parent still holds it)
# child.parent: refcnt 1 (p.__dict__ held it, now gone; child.parent=1)
# Neither reaches 0 → cycle

print(f"Before del: refcount(p) = {sys.getrefcount(p)}")   # 2
del p
print("After del p — objects NOT freed (cycle)")

collected = gc.collect()
print(f"gc.collect() freed {collected} objects")
```

**How the GC handles it:** After `del p`, the `Parent` object has `ob_refcnt = 1` (held by `child.parent`), and the `Child` has `ob_refcnt = 1` (held by `parent.child`). These are both in the "container" tracked list. The GC's mark phase subtracts internal refs: `parent.gc_refs -= 1` (from child.parent), `child.gc_refs -= 1` (from parent.child). Both drop to 0, so both are garbage. The GC calls `tp_dealloc` on both, triggering `__del__`.

**Complication:** If `__del__` is defined, CPython 3.3 and earlier put cycles with finalizers in `gc.garbage` (uncollectable). CPython 3.4+ (PEP 442) handles this by calling `__del__` first, then checking if the object was resurrected.

### (e) Specialization benchmark

```python
import timeit

def add_always_int(n):
    total = 0
    for _ in range(n):
        total = total + 1   # always int + int — stays BINARY_OP_ADD_INT
    return total

x = 1
y = 1.0
flip = [x, y]

def add_mixed_types(n):
    total = 0
    for i in range(n):
        total = total + flip[i % 2]   # alternates int and float — despecializes
    return total

N = 500_000

# Warm up
add_always_int(1000)
add_mixed_types(1000)

t_int = timeit.timeit(lambda: add_always_int(N), number=10) / 10
t_mix = timeit.timeit(lambda: add_mixed_types(N), number=10) / 10

print(f"Always int (specialized):   {t_int*1000:.2f}ms")
print(f"Mixed types (despecialized): {t_mix*1000:.2f}ms")
print(f"Overhead of despecialization: {t_mix/t_int:.2f}×")
```

**Expected:** `add_always_int` benefits from `BINARY_OP_ADD_INT` — the interpreter skips type checks after specialization. `add_mixed_types` repeatedly despecializes and re-specializes, incurring the overhead of both the generic dispatch and the specialization logic. Expect 1.2–2× difference on CPython 3.12.

---

## Summary

| Mechanism | Location | Key insight |
|---|---|---|
| Object model | `Include/object.h` | Every Python object is a `PyObject` C struct |
| Refcounting | `Include/object.h`, `Py_INCREF` | Immediate, deterministic, but can't collect cycles |
| Bytecode | `Python/compile.c` | Stack-based VM, code objects are immutable |
| Eval loop | `Python/ceval.c` | Computed-goto dispatch in GCC, ~150 opcodes |
| Specialization | `Python/specialize.c` | After 8 executions, opcodes become type-specific |
| GIL | `Python/ceval_gil.c` | Mutex protecting refcounts and eval loop |
| Cyclic GC | `Modules/gcmodule.c` | Generational tricolor mark, 3 generations |
| pymalloc | `Objects/obmalloc.c` | Arena → pool → block, 64 size classes ≤512B |
| asyncio | `Lib/asyncio/` | Single-threaded epoll loop, coroutines = generators |
| PyPy JIT | Bolz et al. 2009 | Tracing the interpreter itself = meta-tracing |

**Further reading:**
- CPython source: `Python/ceval.c` (eval loop), `Modules/gcmodule.c` (GC), `Objects/obmalloc.c` (allocator)
- PEP 659 — Specializing Adaptive Interpreter
- PEP 703 — Making the GIL Optional
- Bolz, Cuni, Fijalkowski, Rigo — "Tracing the Meta-Level: PyPy's Tracing JIT Compiler" (2009)
- Victor Stinner's blog: vstinner.github.io
- Mark Shannon's optimization plans: github.com/markshannon/faster-cpython

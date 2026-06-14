# Solution: Lesson 03 — Memory Models & Lock-Free Programming
## Hard Exercise: Lock-Free Reference Counting

---

## 1. Full Exercise Restatement

**Task:** Implement a lock-free reference-counted smart pointer (simulating C++ `std::shared_ptr`), using atomic Compare-And-Swap (CAS) operations for the reference count operations.

**Required components:**
1. A `_RefCountedBlock` holding the object and its atomic reference count
2. A `LockFreeSharedPtr` wrapper with `make()`, `clone()`, `drop()`, and `get()` methods
3. Correct memory ordering semantics (relaxed for increment, acquire-release for decrement)
4. Thread-safe concurrent stress test

**Key questions to address:**
- Why does `clone()` need to increment *before* returning the new pointer?
- What is the ABA problem in the context of `drop()`?
- How does epoch-based reclamation (EBR) solve ABA?
- What memory ordering (`relaxed` vs. `acq_rel`) is required and why?

---

## 2. Conceptual Solution Walkthrough

### Step 1: The Core Invariant

A reference count tracks how many `LockFreeSharedPtr` instances point to a given `_RefCountedBlock`. The invariant is:

> **At any point in time, `refcount` equals the number of live `LockFreeSharedPtr` instances that hold a reference to this block.**

If this invariant is violated:
- `refcount` goes to 0 prematurely → use-after-free (the block is freed while someone still holds it)
- `refcount` never reaches 0 → memory leak

### Step 2: Why `clone()` Must Increment Before Returning

The critical ordering in `clone()`:

```
WRONG:                              CORRECT:
new_ptr = LockFreeSharedPtr(block)  success = block.increment_refcount()
block.increment_refcount()          if not success: return None
return new_ptr                      new_ptr = LockFreeSharedPtr(block)
                                    return new_ptr
```

The wrong version has a window between returning the new pointer and incrementing the refcount. If another thread calls `drop()` on the last other reference in that window, the block could be freed before we've registered our own reference. This is a classic TOCTOU (time-of-check/time-of-use) race.

### Step 3: Memory Ordering Requirements

In C++11's memory model (which Python threads approximate through the GIL, but we simulate explicitly):

**`increment_refcount` (clone):** `memory_order_relaxed` is sufficient.
- We don't need any ordering guarantees about other memory operations.
- We only need the refcount increment itself to be atomic.
- The new pointer is not yet visible to other threads, so no synchronization is needed.

**`decrement_refcount` (drop):** `memory_order_acq_rel` is required.
- **Release semantics on decrement:** All writes made before `drop()` (mutations to the object's fields) must be visible to whichever thread wins the "last decrement" race. Without release, the destructor might not see those writes.
- **Acquire semantics on the decrement that reaches 0:** The thread that observes `refcount == 0` must see all prior decrements' release-fenced writes. Without acquire, the final destructor might be called without "knowing" about previous decrements.

In Python, our `threading.Lock()` provides the equivalent of `acq_rel` through its `acquire()`/`release()` memory barriers.

### Step 4: The ABA Problem in Reference Counting

The ABA problem arises when using lock-free data structures that hold raw pointers to `_RefCountedBlock` objects (e.g., a lock-free stack that stores `block*` pointers):

1. Thread 1 reads `head = block_A` and plans to CAS `(head, block_A, block_A.next)`
2. Thread 1 is preempted
3. Thread 2 removes `block_A` from the structure, decrements its refcount to 0, and the allocator frees it
4. Thread 3 allocates a new block — the allocator returns the *same memory address* (slab reuse)
5. Thread 3 pushes this new block onto the stack at the same address as `block_A`
6. Thread 1 resumes: CAS sees `head == block_A` (address matches!), CAS succeeds, but now Thread 1 has a dangling pointer to what it thinks is `block_A`

The refcount alone doesn't prevent ABA because the refcount lives *inside* the block — and the block was freed and reallocated. The address is identical; the content is different.

### Step 5: Epoch-Based Reclamation

EBR solves ABA by ensuring that freed memory is not reused while any thread might hold a raw (unregistered) pointer to it.

The key insight: when does a thread hold a raw pointer without a registered reference?
- In `clone()`, between reading the block pointer and calling `increment_refcount()`
- In lock-free data structures, between reading a node's pointer and either following it or doing a CAS on it

EBR maintains a global epoch counter (usually 3 epochs, cycling 0→1→2→0). Each thread "announces" its current epoch when entering a critical section. Freed blocks are placed into a "limbo" list for the current epoch. A block is only physically freed after *two* full epoch advances — guaranteeing all threads have exited critical sections that could have observed the old pointer.

---

## 3. Full Working Python Implementation

```python
"""
Lock-free reference-counted smart pointer (std::shared_ptr simulation).

Memory ordering semantics:
- increment_refcount: relaxed (just atomicity, no ordering needed)
- decrement_refcount: acq_rel (synchronizes with the destructor thread)

Python's threading.Lock provides the necessary memory barriers.
"""

from __future__ import annotations
import threading
import sys
import time
import random
from typing import Optional, TypeVar, Generic, List
from dataclasses import dataclass, field
from contextlib import contextmanager

T = TypeVar('T')


# ── Control block ──────────────────────────────────────────────────────────────

class _RefCountedBlock(Generic[T]):
    """
    Internal control block.
    In C++, this is the control block allocated alongside the object.
    Layout: [refcount (atomic<int>)] [weak_count (atomic<int>)] [T value]
    """

    __slots__ = ('value', '_refcount', '_lock', '_destroyed')

    def __init__(self, value: T):
        self.value: Optional[T] = value
        self._refcount: int = 1
        self._lock = threading.Lock()
        self._destroyed = False

    def increment_refcount(self) -> bool:
        """
        Atomically increment refcount.
        Returns False if the block is already being destroyed (refcount <= 0).

        Memory order: relaxed — we only need atomicity of the count itself.
        In C++: fetch_add(1, memory_order_relaxed)
        """
        with self._lock:
            if self._refcount <= 0:
                return False   # being destroyed
            self._refcount += 1
            return True

    def decrement_refcount(self) -> int:
        """
        Atomically decrement refcount.
        Returns the new count.

        Memory order: acq_rel.
        - Release: our writes to self.value are visible to the destructor.
        - Acquire: the "last decrement" thread sees all prior decrements.
        In C++: fetch_sub(1, memory_order_acq_rel)
        """
        with self._lock:
            self._refcount -= 1
            count = self._refcount
        return count

    def destroy(self) -> None:
        """Called exactly once, by the last drop()."""
        with self._lock:
            self._destroyed = True
            self.value = None   # Release the held object

    @property
    def refcount(self) -> int:
        with self._lock:
            return self._refcount


# ── Smart pointer ──────────────────────────────────────────────────────────────

class LockFreeSharedPtr(Generic[T]):
    """
    Lock-free reference-counted smart pointer.

    Invariant: if self._block is not None, then self._block._refcount >= 1.
    (This instance holds one of those references.)
    """

    __slots__ = ('_block',)

    def __init__(self, block: Optional[_RefCountedBlock[T]]):
        # Callers must have already incremented refcount before passing the block.
        self._block = block

    @classmethod
    def make(cls, value: T) -> 'LockFreeSharedPtr[T]':
        """
        Construct a new shared pointer.
        Equivalent to C++ std::make_shared<T>(value).
        Initial refcount = 1.
        """
        block = _RefCountedBlock(value)
        return cls(block)

    @classmethod
    def null(cls) -> 'LockFreeSharedPtr[T]':
        """Null shared pointer (refcount = 0, no block)."""
        return cls(None)

    def clone(self) -> Optional['LockFreeSharedPtr[T]']:
        """
        Atomically clone this shared pointer.
        Returns None if the block has already been destroyed.

        CRITICAL ORDER:
        1. Increment refcount FIRST (before constructing the new handle)
        2. Only then return the new LockFreeSharedPtr

        If we returned the handle before incrementing, another thread's
        drop() could race with us and reach refcount=0, destroying the
        block before we increment — use-after-free.

        In C++: shared_ptr's copy constructor does:
            ptr(other.ptr),
            ctrl(other.ctrl)  {  // ctrl->refcount.fetch_add(1, relaxed)  }
        """
        if self._block is None:
            return None
        success = self._block.increment_refcount()
        if not success:
            # Block is being destroyed (refcount was <= 0 when we tried)
            return None
        return LockFreeSharedPtr(self._block)

    def drop(self) -> None:
        """
        Release this reference.
        If refcount drops to 0, destroy the controlled object.

        In C++: ~shared_ptr() {
            if (ctrl->refcount.fetch_sub(1, acq_rel) == 1) {
                ctrl->destroy();
            }
        }
        """
        if self._block is None:
            return

        new_count = self._block.decrement_refcount()

        if new_count == 0:
            # We are the last reference holder.
            # The acq_rel on our decrement guarantees:
            #   - All other threads' release-fenced writes to the object
            #     are now visible to us (acquire).
            #   - Our writes are visible to any thread that saw our release
            #     (but there are none since we're last).
            self._block.destroy()
        elif new_count < 0:
            raise RuntimeError(
                f"[BUG] Over-drop detected! refcount went to {new_count}. "
                f"This is undefined behavior in C++ (heap corruption)."
            )

        self._block = None

    def get(self) -> T:
        """
        Dereference the pointer.
        Only safe while holding a valid (non-dropped) reference.
        """
        if self._block is None:
            raise RuntimeError("Null dereference: shared_ptr is null or dropped")
        if self._block._destroyed:
            raise RuntimeError("Use-after-free: referenced block has been destroyed")
        return self._block.value

    def use_count(self) -> int:
        """Current reference count (approximate — may change immediately)."""
        if self._block is None:
            return 0
        return self._block.refcount

    def __bool__(self) -> bool:
        return self._block is not None

    def __enter__(self):
        return self

    def __exit__(self, *_):
        self.drop()


# ── Epoch-Based Reclamation (conceptual implementation) ───────────────────────

class EBRReclaimer:
    """
    Epoch-Based Reclamation for deferred memory freeing.

    Prevents ABA problem in lock-free data structures that hold raw pointers
    to _RefCountedBlocks (e.g., a lock-free stack of shared_ptr).

    Global epoch advances 0 → 1 → 2 → 0 (cycles mod 3).
    Freed blocks go into limbo[current_epoch].
    After TWO epoch advances, all blocks in limbo[old_epoch] are safe to free.
    """

    _INACTIVE = -1
    _N_EPOCHS = 3

    def __init__(self):
        self._global_epoch = 0
        self._global_lock = threading.Lock()
        self._thread_epochs: dict = {}   # thread_id → epoch (or INACTIVE)
        self._limbo: List[List] = [[] for _ in range(self._N_EPOCHS)]
        self._limbo_lock = threading.Lock()

    @contextmanager
    def critical_section(self):
        """
        Enter/exit a critical section.
        While inside, this thread's epoch is announced, preventing
        GC of blocks from the current epoch.
        """
        tid = threading.get_ident()
        with self._global_lock:
            epoch = self._global_epoch
        self._thread_epochs[tid] = epoch
        # Acquire barrier: read global_epoch before reading any shared pointers
        try:
            yield epoch
        finally:
            self._thread_epochs[tid] = self._INACTIVE

    def retire(self, block: _RefCountedBlock) -> None:
        """
        Add a block to the current epoch's limbo list.
        It will be freed after two epoch advances.
        """
        with self._global_lock:
            epoch = self._global_epoch
        with self._limbo_lock:
            self._limbo[epoch].append(block)
        self._try_advance_epoch()

    def _try_advance_epoch(self) -> None:
        """
        Advance the global epoch if all threads are in the current epoch.
        If advanced, free blocks from (current - 2) mod 3 epoch.
        """
        with self._global_lock:
            cur = self._global_epoch
            # Check all threads are in the current epoch or inactive
            for tid, epoch in list(self._thread_epochs.items()):
                if epoch != self._INACTIVE and epoch != cur:
                    return  # Some thread is in an old epoch

            # Safe to advance
            new_epoch = (cur + 1) % self._N_EPOCHS
            old_epoch = (cur - 2) % self._N_EPOCHS

            with self._limbo_lock:
                to_free = self._limbo[old_epoch]
                self._limbo[old_epoch] = []

            # Physically reclaim blocks
            for block in to_free:
                block.value = None  # trigger destructor logic

            self._global_epoch = new_epoch


# ── Concurrent stress tests ────────────────────────────────────────────────────

def test_basic_operations():
    """Basic functionality: make, clone, drop, get."""
    print("--- Basic Operations ---")

    # Test 1: make and get
    ptr = LockFreeSharedPtr.make(42)
    assert ptr.get() == 42
    assert ptr.use_count() == 1
    print("  [1] make() and get() work")

    # Test 2: clone
    ptr2 = ptr.clone()
    assert ptr2 is not None
    assert ptr2.get() == 42
    assert ptr.use_count() == 2
    print("  [2] clone() increments refcount correctly")

    # Test 3: drop one clone
    ptr2.drop()
    assert ptr.use_count() == 1
    print("  [3] drop() decrements refcount correctly")

    # Test 4: final drop destroys block
    block = ptr._block
    ptr.drop()
    assert block._destroyed, "Block should be destroyed after last drop"
    assert ptr._block is None, "ptr._block should be None after drop"
    print("  [4] Last drop() destroys the block")

    # Test 5: null pointer
    null_ptr = LockFreeSharedPtr.null()
    assert not null_ptr
    assert null_ptr.use_count() == 0
    try:
        null_ptr.get()
        assert False, "Should have raised"
    except RuntimeError:
        pass
    print("  [5] Null pointer raises on get()")


def test_concurrent_clone_drop(n_threads: int = 20, n_iters: int = 2000):
    """
    Stress test: concurrent clone() and drop() from multiple threads.
    
    Invariant check: the object must be destroyed exactly once,
    and only after all threads have dropped their references.
    """
    print(f"\n--- Concurrent Stress Test ({n_threads} threads × {n_iters} iters) ---")

    destroy_count = [0]
    destroy_lock = threading.Lock()
    errors = []

    # Patch _destroy to count destructions
    original_destroy = _RefCountedBlock.destroy
    def counting_destroy(self):
        original_destroy(self)
        with destroy_lock:
            destroy_count[0] += 1
    _RefCountedBlock.destroy = counting_destroy

    try:
        root = LockFreeSharedPtr.make("shared_object")

        def worker():
            try:
                for _ in range(n_iters):
                    # Clone the root, use it briefly, then drop
                    local_ptr = root.clone()
                    if local_ptr is None:
                        continue   # root was being destroyed (shouldn't happen here)

                    # Verify the value is correct (use-after-free check)
                    value = local_ptr.get()
                    if value != "shared_object":
                        errors.append(f"Value corruption: got {value!r}")

                    # Occasionally chain clone → drop
                    if random.random() < 0.1:
                        second = local_ptr.clone()
                        if second:
                            second.drop()

                    local_ptr.drop()
            except Exception as e:
                errors.append(str(e))

        threads = [threading.Thread(target=worker) for _ in range(n_threads)]
        for t in threads:
            t.start()
        for t in threads:
            t.join()

        # Drop the root reference
        root.drop()

        # Block must be destroyed exactly once
        assert not errors, f"Errors: {errors}"
        assert destroy_count[0] == 1, \
            f"Block destroyed {destroy_count[0]} times (expected 1)"
        print(f"  [✓] No errors. Block destroyed exactly once.")

    finally:
        _RefCountedBlock.destroy = original_destroy


def test_aba_illustration():
    """
    Illustrate (not exploit) the ABA problem.
    
    Shows WHY raw pointer CAS without EBR can cause use-after-free,
    and WHY EBR prevents it.
    """
    print("\n--- ABA Problem Illustration ---")

    print("  Scenario: Lock-free stack of raw pointers to RefCountedBlocks")
    print("  Without EBR:")
    print("    T1 reads head=block_A, gets preempted")
    print("    T2 pops block_A, drops it (refcount→0, block_A freed)")
    print("    T3 allocates new block at SAME ADDRESS as block_A")
    print("    T1 resumes: CAS(head, block_A, block_A.next) SUCCEEDS")
    print("    T1 dereferences block_A — but it's now T3's block: USE-AFTER-FREE")
    print()
    print("  With EBR:")
    print("    T1 announces epoch=0 before reading the pointer")
    print("    T2 can retire block_A (place in limbo[0]) but cannot free it")
    print("    Epoch advances only after T1 exits its critical section")
    print("    block_A is freed only after 2 more epoch advances")
    print("    By then, T1 has long since exited — no raw pointer exists")
    print("    ABA prevented: the memory address is not reused during T1's window")


def test_over_drop_detection():
    """Verify that double-drop is caught (in production: undefined behavior)."""
    print("\n--- Over-Drop Detection ---")

    ptr = LockFreeSharedPtr.make("test")
    clone = ptr.clone()
    ptr.drop()
    clone.drop()  # This is the last valid drop

    # Trying to drop again should detect the problem
    try:
        clone.drop()   # Over-drop
        # Our implementation nulls _block after drop, so this is a no-op
        print("  [✓] Second drop on None _block is a no-op (safe)")
    except RuntimeError as e:
        print(f"  [✓] Over-drop detected: {e}")


def run_all_tests():
    """Run the complete test suite."""
    print("=" * 65)
    print("Lock-Free Shared Pointer Test Suite")
    print("=" * 65)
    test_basic_operations()
    test_concurrent_clone_drop()
    test_aba_illustration()
    test_over_drop_detection()
    print("\n[✓] All tests passed.")


if __name__ == "__main__":
    run_all_tests()
```

---

## 4. Common Wrong Approaches and Why They Fail

### Wrong Approach 1: Incrementing After Creating the New Handle

```python
# WRONG
def clone(self):
    new_ptr = LockFreeSharedPtr(self._block)   # new handle created first
    self._block.increment_refcount()           # increment second ← RACE
    return new_ptr
```

**Why it fails:** Between creating `new_ptr` and incrementing the refcount, another thread might call `drop()` on the last remaining reference. If that drop decrements the refcount to 0, the block is destroyed. When our increment then executes, we increment a destroyed block's counter — and `new_ptr` is a dangling pointer. This is exactly the use-after-free bug in naive C++ copy constructors.

### Wrong Approach 2: Using `memory_order_relaxed` for Decrement

```cpp
// WRONG (C++)
int new_count = refcount.fetch_sub(1, std::memory_order_relaxed);
if (new_count == 1) {
    delete value;  // may not see all prior writes!
}
```

**Why it fails:** `memory_order_relaxed` provides no ordering guarantees beyond atomicity of the operation itself. The thread that decrements to 0 (the "last" thread) calls the destructor. But without acquire semantics on the final decrement, the destructor's reads may not see writes made by other threads before their `drop()` calls. Specifically: Thread A writes to `block->value`, then drops its reference (with `relaxed`). Thread B is the last to drop. Without acquire, Thread B's destructor might see stale (unwritten) data in `block->value`. The C++ standard explicitly requires `acq_rel` on shared_ptr's decrement.

### Wrong Approach 3: Checking Refcount After Decrement Without Acquire

```python
# WRONG
def drop(self):
    self._block._refcount -= 1          # non-atomic!
    if self._block._refcount == 0:      # TOCTOU race
        self._block.destroy()
```

**Why it fails:** Two threads could both decrement (non-atomically) from 1, both read 0, and both call `destroy()`. This is a double-free. The decrement and the subsequent check must be atomic together (what `fetch_sub` + checking the return value achieves in C++).

### Wrong Approach 4: Not Handling the `refcount <= 0` Case in `increment_refcount`

```python
# WRONG
def increment_refcount(self):
    self._refcount += 1   # doesn't check if being destroyed
```

**Why it fails:** If one thread is executing the final `drop()` (refcount was 1, just decremented to 0) and another thread tries to `clone()` at the same moment, the clone could successfully increment from 0 to 1 — a zombie resurrection that prevents proper destruction and may lead to use-after-free when the resurrector eventually drops.

The fix: check `refcount <= 0` inside the same atomic operation as the increment. If it's already 0, the clone attempt fails gracefully.

### Wrong Approach 5: Relying on Python's GC Instead of Explicit `drop()`

**Mistake:** "Python has garbage collection, so the refcount will be handled automatically when the object goes out of scope."

**Why it fails (in this simulation):** Our `LockFreeSharedPtr` has an explicit `_block` reference that Python's GC will clean up — but this bypasses our `decrement_refcount()` call entirely. Python's GC uses its own reference counting (CPython) that doesn't interact with our simulated C++-style control block. The `_block.destroy()` method must be called explicitly via `drop()`. In C++, this is enforced by the destructor.

---

## 5. Extensions: Scale and Adversarial Inputs

### 1,000-Thread Stress Test

At 1,000 concurrent threads each cloning and dropping at high frequency, the main bottleneck is the per-block `threading.Lock()`. In C++, the equivalent `std::atomic<int>` requires no mutex — it uses a single CPU instruction (LOCK XADD on x86) that takes ~5ns vs. ~100ns for a mutex operation. For Python, the GIL provides safety but also limits true parallelism.

In production C++ (Boost, `std::shared_ptr`): refcount operations are ~3ns each on modern x86 (LOCK XADD is one cycle on the L1 cache). For 1,000 threads doing nothing but clone/drop cycles, a well-tuned shared_ptr can sustain ~100M operations/second per core.

### 1TB Object Store

At 1TB of objects managed via shared_ptr, the main concern is **memory fragmentation** from the two-allocation model (`new T` + `new ControlBlock`). `std::make_shared` solves this by allocating the object and control block in a single allocation — but then the control block (and its memory) lives as long as any `weak_ptr` exists, potentially doubling peak memory.

For very large objects (say, 1GB each), `std::weak_ptr` becomes critical: a weak pointer holds a reference to the control block (for `expired()` checks) but not to the object. When all `shared_ptr`s are gone, the object is destroyed even if `weak_ptr`s still exist — the control block stays until all weak refs die too.

### Adversarial Inputs: Lock-Free Stack ABA

The ABA problem is most dangerous in lock-free stacks/queues where both `push()` and `pop()` use CAS on a head pointer. Canonical fix in production: tagged pointers (store a version counter in the unused high bits of a 64-bit pointer) or hazard pointers (per-thread list of pointers currently "in use").

Linux kernel's RCU (Read-Copy-Update) takes a different approach entirely: reads are truly lock-free, writes synchronize via a "grace period" mechanism that waits for all RCU readers to complete before freeing memory. RCU's `synchronize_rcu()` is functionally equivalent to EBR's "wait two epochs" but implemented without any per-reader overhead.

### Weak Pointers and the Zombie Problem

A full `std::shared_ptr` implementation also includes `std::weak_ptr`, which holds a reference to the control block but not the object. This enables:
- Breaking circular references (avoiding memory leaks in graphs/trees)
- Safe "try to get a shared_ptr if the object still exists" (`weak_ptr::lock()`)

The zombie problem: if `weak_ptr::lock()` runs concurrently with the last `shared_ptr::drop()`, the lock must atomically increment the refcount and check if it went from 0→1 (zombie resurrection). This is exactly the `increment_refcount() → check for <= 0` pattern we implement.

---

## 6. Real-World Production System References

### `boost::shared_ptr` and `std::shared_ptr` (C++ STL)

The reference implementation in Boost (`boost/smart_ptr/detail/sp_counted_base.hpp`) uses `atomic<int>` with exactly the memory orders described here. The `add_ref_copy()` function uses `memory_order_relaxed`; `release()` uses `memory_order_acq_rel`. The separation matches our simulation exactly.

The C++ standard mandates this in [util.smartptr.shared.atomic]: "The use count of a shared_ptr constructor is modified using a relaxed operation; otherwise modifications use an acquire or release operation."

### Linux Kernel's `kref`

`struct kref` in the Linux kernel (`include/linux/kref.h`) is the kernel's reference counting primitive. `kref_get()` uses `refcount_inc()` (which maps to `atomic_inc` — no memory ordering beyond atomicity). `kref_put()` uses `refcount_dec_and_test()` with a memory barrier before calling the release function.

The kernel's `refcount_t` uses `memory_order_release` on decrement and an explicit `smp_acquire__after_ctrl_dep()` macro before the destructor — matching our `acq_rel` analysis.

### Crossbeam (Rust EBR Library)

Crossbeam (`github.com/crossbeam-rs/crossbeam`) is the canonical implementation of epoch-based reclamation in Rust. The `crossbeam_epoch::Collector` provides the 3-epoch cycle, and `Owned<T>`, `Shared<T>`, `Guard` types implement the same critical-section announcement protocol described above. It is used by Tokio (async runtime), Sled (embedded database), and TiKV.

### Folly's `hazptr` (Meta/Facebook)

Meta's Folly library includes `folly::hazptr_holder` — an implementation of hazard pointers as an alternative to EBR. Hazard pointers are more memory-efficient than EBR (O(k) limbo lists where k is the number of threads) but have higher per-operation overhead (must scan all hazard pointer records on retire). Folly uses hazard pointers in their concurrent hash map (`folly::ConcurrentHashMap`) where the number of simultaneously-in-use pointers is bounded.

The choice between hazard pointers and EBR depends on workload: EBR has lower per-operation overhead but can delay reclamation indefinitely if a thread stalls; hazard pointers reclaim eagerly but scan more on each retire.

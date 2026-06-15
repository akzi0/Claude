# Solution: Lock-Free Reference Counting (Hard Exercise — Lesson 03)

## The Question

Implement a CAS-based shared pointer with `clone()` and `drop()`. Then:

1. Give a concrete ABA scenario where naive CAS-based refcounting fails.
2. Describe epoch-based reclamation (EBR) as the solution.

---

## Part 1: Full Implementation

The implementation uses Python's `threading.Lock` as a stand-in for hardware CAS. Comments explain the corresponding C++ atomic operations.

```python
"""
Lock-free reference-counted shared pointer.
Python simulation using threading.Lock as CAS substitute.
"""
import threading
import sys
import time
from typing import Optional, TypeVar, Generic, List

T = TypeVar('T')


class RefCountBlock(Generic[T]):
    """
    Internal control block: holds the managed object and its reference count.
    In C++: the 'control block' allocated separately from the object.
    """
    def __init__(self, value: T):
        self.value: Optional[T] = value
        self._refcount: int = 1
        self._lock = threading.Lock()  # In C++: std::atomic<int>

    def increment(self) -> bool:
        """
        Atomically increment refcount. Returns False if object is being destroyed.
        
        C++ equivalent:
            int old = refcount.fetch_add(1, std::memory_order_relaxed);
            return old > 0;
        
        WHY relaxed? increment() is only called when we already hold a reference
        (or are racing to acquire one — that case is handled in clone()). The
        increment itself doesn't need to synchronize any other memory.
        """
        with self._lock:
            if self._refcount <= 0:
                return False  # Already being destroyed — too late to clone
            self._refcount += 1
            return True

    def decrement(self) -> int:
        """
        Atomically decrement refcount. Returns new count.
        
        C++ equivalent:
            return refcount.fetch_sub(1, std::memory_order_acq_rel) - 1;
        
        WHY acq_rel?
        - RELEASE: ensures all writes to the managed object made before this
          drop() are visible to the thread that will do the final free.
        - ACQUIRE: the final decrement (reaching 0) must see all prior decrements
          so it knows it truly is the last reference.
        
        Without acq_rel, two threads could both see refcount drop to 0 and both
        try to free the object (double-free). Or the freeing thread might not see
        the full state of the object as written by the last user.
        """
        with self._lock:
            self._refcount -= 1
            return self._refcount

    @property
    def refcount(self) -> int:
        with self._lock:
            return self._refcount


class SharedPtr(Generic[T]):
    """
    Reference-counted smart pointer.
    clone() increments the refcount and returns a new handle.
    drop() decrements and frees if it reaches zero.
    """

    def __init__(self, block: Optional[RefCountBlock[T]] = None):
        self._block = block

    @classmethod
    def make(cls, value: T) -> 'SharedPtr[T]':
        """Create a new shared pointer with refcount=1."""
        return cls(RefCountBlock(value))

    def clone(self) -> Optional['SharedPtr[T]']:
        """
        Atomically clone: increment refcount before returning new handle.
        Returns None if the block has already been destroyed.
        
        CRITICAL ORDER: increment BEFORE constructing the new handle.
        If we construct first and then increment, another thread's drop()
        could reach 0 and free the object between our construction and increment.
        """
        if self._block is None:
            return None
        success = self._block.increment()
        if not success:
            # Object is being destroyed — we raced with the last drop()
            return None
        return SharedPtr(self._block)

    def drop(self) -> None:
        """
        Release this reference. If refcount reaches zero, destroy the object.
        
        After drop(), this SharedPtr becomes null and must not be used.
        """
        if self._block is None:
            return
        block = self._block
        self._block = None  # Null out BEFORE decrement to prevent double-drop
        new_count = block.decrement()
        if new_count == 0:
            # We are the last reference: destroy the managed object
            block.value = None  # In C++: ~T() + deallocate
        elif new_count < 0:
            raise RuntimeError(f"BUG: refcount went negative ({new_count}). Double-drop!")

    def get(self) -> T:
        """Dereference — only valid while holding a reference."""
        if self._block is None or self._block.value is None:
            raise RuntimeError("Dereferencing null or destroyed SharedPtr")
        return self._block.value

    @property
    def refcount(self) -> int:
        return self._block.refcount if self._block else 0

    def is_null(self) -> bool:
        return self._block is None


# ─── Basic smoke tests ─────────────────────────────────────────────────────────

def test_basic_lifecycle():
    """Verify refcount increments and decrements correctly."""
    ptr = SharedPtr.make(42)
    assert ptr.refcount == 1
    assert ptr.get() == 42

    ptr2 = ptr.clone()
    assert ptr.refcount == 2
    assert ptr2.refcount == 2  # same block

    ptr.drop()
    assert ptr2.refcount == 1  # ptr was dropped, ptr2 still alive

    ptr2.drop()
    # Block is now destroyed; ptr2._block is None after drop
    assert ptr2.is_null()

    print("✓ Basic lifecycle test passed")


def test_concurrent_clones_and_drops():
    """Stress test: many threads clone and drop simultaneously."""
    root = SharedPtr.make("shared data")
    errors: List[str] = []
    N_THREADS = 20
    N_OPS = 1000

    def worker(thread_id: int):
        local_ptrs: List[SharedPtr] = []
        for _ in range(N_OPS):
            clone = root.clone()
            if clone:
                local_ptrs.append(clone)
            if local_ptrs and (thread_id * _ % 3 == 0):
                p = local_ptrs.pop()
                try:
                    p.drop()
                except RuntimeError as e:
                    errors.append(str(e))
        for p in local_ptrs:
            try:
                p.drop()
            except RuntimeError as e:
                errors.append(str(e))

    threads = [threading.Thread(target=worker, args=(i,)) for i in range(N_THREADS)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

    root.drop()
    assert not errors, f"Errors in concurrent test: {errors}"
    print(f"✓ Concurrent stress test passed ({N_THREADS} threads × {N_OPS} ops)")


# ─── Run basic tests ────────────────────────────────────────────────────────────
if __name__ == "__main__":
    test_basic_lifecycle()
    test_concurrent_clones_and_drops()
```

---

## Part 2: Concrete ABA Scenario

### Setup

Consider a **lock-free stack** where each stack node IS a `SharedPtr` control block, and the stack's head pointer is a raw address (not a wrapped SharedPtr). This is a common pattern in lock-free data structures where you need CAS on the pointer itself.

```
Lock-free stack (using raw pointers, CAS on head):

Time 0: head → [Block_A, rc=2] → [Block_B, rc=1] → NULL
         Thread 1 holds a reference to Block_A (that's why rc=2)
         Thread 1 reads head = Block_A_addr
         Thread 1 plans: CAS(head, Block_A_addr, Block_A.next = Block_B_addr)
         Thread 1 is PREEMPTED before executing the CAS.
```

### The ABA Problem Unfolds

```
Time 1: Thread 1 is sleeping.

Time 2: Thread 2 pops Block_A from the stack:
         - CAS(head, Block_A_addr, Block_B_addr) → head = Block_B_addr
         - Thread 2 drops its stack reference: Block_A.rc = 2 → 1
         - Thread 2 pops Block_B from the stack:
           - head = NULL
           - Block_B.rc = 1 → 0 → Block_B is FREED, memory returned to allocator
         - Thread 2 drops its own held reference to Block_A: rc = 1 → 0
         - Block_A is FREED, memory returned to allocator.
         
Time 3: Thread 3 allocates a new block. The allocator returns
        the SAME PHYSICAL ADDRESS that Block_A used to occupy.
        (Common with slab/pool allocators — they recycle recently freed blocks.)
        Call this new_Block at addr = Block_A_addr.
        Thread 3 pushes new_Block: CAS(head, NULL, Block_A_addr) → head = Block_A_addr.

Time 4: Thread 1 wakes up.
        Thread 1 reads head: still == Block_A_addr! ← THE LIE
        Thread 1 executes: CAS(head, Block_A_addr, Block_B_addr)
        CAS SUCCEEDS (head == Block_A_addr is true — but it's new_Block, not old Block_A!)
        head is now set to Block_B_addr — DANGLING POINTER (Block_B was freed at Time 2!)
        Thread 3's new_Block is now orphaned (nobody points to it, but it won't be freed).
        
RESULT: Use-after-free on Block_B (anyone who dereferences head now accesses freed memory).
         Memory leak of new_Block.
```

### Why the Refcount Doesn't Save Us

The critical insight: the reference count lives **inside** the block. Once the block is freed and the allocator reuses that memory, the bytes at that address now belong to `new_Block`. The old refcount value is meaningless. Thread 1 is holding a pointer to what it **thinks** is Block_A, but the object at that address is now `new_Block`. The CAS on the head pointer checks the address — not the identity — so it succeeds even though the world changed underneath.

**The refcount only protects the object it lives in, not the pointer to that object.** The ABA problem arises at the pointer level, not the object level.

---

## Part 3: Epoch-Based Reclamation (EBR)

EBR is one of the three standard solutions to the ABA/free-too-early problem (the others are hazard pointers and tagged pointers).

### Core Idea

Delay freeing a block until we are **certain** no thread holds a raw (unprotected) pointer to it — even temporarily.

```
GLOBAL_EPOCH: atomic int (0, 1, or 2 — cycles mod 3)
THREAD_EPOCH[tid]: announced epoch per thread (or INACTIVE)
LIMBO[0,1,2]: per-epoch lists of blocks pending reclamation
```

### Pseudocode

```python
"""
Epoch-Based Reclamation pseudocode.
Real implementation: libcds, crossbeam (Rust), or folly::hazptr.
"""
import threading
from typing import Optional, List
from dataclasses import dataclass, field

INACTIVE = -1
N_EPOCHS = 3

@dataclass
class EBRState:
    global_epoch: int = 0
    thread_epochs: dict = field(default_factory=dict)
    limbo: List[list] = field(default_factory=lambda: [[], [], []])
    _lock: threading.Lock = field(default_factory=threading.Lock)


_ebr = EBRState()
_tid_counter = 0
_tls = threading.local()


def _get_tid() -> int:
    global _tid_counter
    if not hasattr(_tls, 'tid'):
        with _ebr._lock:
            _tid_counter += 1
            _tls.tid = _tid_counter
            _ebr.thread_epochs[_tls.tid] = INACTIVE
    return _tls.tid


def enter_critical_section():
    """
    Thread announces it is entering a critical section at the current epoch.
    Must be called before reading any shared pointer.
    
    Corresponding C++ (crossbeam): Guard = epoch::pin()
    """
    tid = _get_tid()
    epoch = _ebr.global_epoch  # relaxed load
    _ebr.thread_epochs[tid] = epoch   # announce (release store in C++)


def exit_critical_section():
    """Thread exits the critical section — may no longer hold raw pointers."""
    tid = _get_tid()
    _ebr.thread_epochs[tid] = INACTIVE


def try_advance_epoch():
    """
    Try to advance the global epoch. Succeeds only if ALL active threads
    have announced an epoch >= current global epoch (i.e., everyone has
    entered a critical section at least once since the last advancement).
    
    When the epoch advances, blocks in LIMBO[old_epoch] can be freed.
    """
    with _ebr._lock:
        current = _ebr.global_epoch
        # Check all active threads
        for tid, announced in _ebr.thread_epochs.items():
            if announced == INACTIVE:
                continue  # Not in critical section — fine
            if announced != current:
                return False  # This thread is still in an old epoch
        # Safe to advance
        _ebr.global_epoch = (current + 1) % N_EPOCHS
        # Free the blocks in the newly-recycled epoch slot
        freed_epoch = (_ebr.global_epoch - 1) % N_EPOCHS  # the epoch we just left
        # Wait: actually we free LIMBO[global_epoch] (the slot about to be reused)
        # which is guaranteed to have been fully observed by all threads
        reclaim_epoch = _ebr.global_epoch
        blocks_to_free = _ebr.limbo[reclaim_epoch]
        _ebr.limbo[reclaim_epoch] = []
        return blocks_to_free  # caller frees these


def retire_block(block):
    """
    Schedule a block for deferred reclamation.
    The block will be freed once all threads have advanced past the current epoch.
    """
    current_epoch = _ebr.global_epoch
    with _ebr._lock:
        _ebr.limbo[current_epoch].append(block)

    # Try to advance the epoch (may or may not succeed)
    freed = try_advance_epoch()
    if freed:
        for b in freed:
            b.value = None  # Actually free the block
            # In C++: allocator.deallocate(b)


# ─── How EBR fixes the ABA problem ─────────────────────────────────────────────

"""
Revised scenario with EBR:

Time 0: Thread 1 calls enter_critical_section() and announces epoch=0.
         Thread 1 reads head = Block_A_addr.
         Thread 1 is PREEMPTED.

Time 1: Thread 2 pops Block_A and Block_B.
         Thread 2 calls retire_block(Block_A) → LIMBO[0].append(Block_A)
         Thread 2 calls retire_block(Block_B) → LIMBO[0].append(Block_B)
         Thread 2 calls try_advance_epoch():
           - Thread 1 is still at epoch=0 (announced).
           - Cannot advance: Thread 1 has not exited and re-entered.
         BLOCKS ARE NOT FREED.

Time 2: Thread 3 allocates a new block.
         allocator.allocate() returns NEW memory (not Block_A's address)
         because Block_A is in LIMBO — NOT freed yet.
         Thread 3 pushes new_Block at a DIFFERENT address.

Time 3: Thread 1 wakes up.
         Thread 1 executes CAS(head, Block_A_addr, Block_B_addr).
         BUT head ≠ Block_A_addr now (Thread 3 pushed new_Block, head = new_Block_addr).
         CAS FAILS. Thread 1 retries from the start (reads current head).

RESULT: No ABA. No use-after-free. The retired blocks stay alive until
        ALL threads have exited and re-entered critical sections, proving
        no thread holds a stale pointer to them.
"""


# ─── EBR-protected clone() ──────────────────────────────────────────────────────

def ebr_clone(head_ptr_ref: list) -> Optional[object]:
    """
    Safe clone with EBR protection.
    head_ptr_ref is a list wrapping the head pointer (mutable reference).
    """
    enter_critical_section()  # Announce: I'm about to read a pointer
    try:
        block = head_ptr_ref[0]  # Read the pointer inside the critical section
        if block is None:
            return None
        # Increment refcount — safe because block is in our epoch's limbo if retired
        success = block.increment()
        if not success:
            return None
        return SharedPtr(block)
    finally:
        exit_critical_section()  # Release: no longer reading raw pointers
```

### Why EBR Works: The Key Invariant

```
If a block is in LIMBO[epoch E], it can only be freed when all threads
have announced epoch >= E+1. This means:

1. All threads that read a pointer BEFORE retirement (in epoch E or earlier)
   have since exited their critical section and re-entered at epoch > E.
   
2. They cannot still be holding a raw pointer to the retired block.

3. The allocator cannot reuse the block's memory until it's freed from LIMBO.

4. Therefore: no thread can see the same address after reuse.
   The "A" (old value) and the second "A" (reused address) cannot coexist
   in a single thread's critical section.
```

### Three EBR Variants in Practice

| Technique | When to Free | Overhead | Use Case |
|-----------|-------------|----------|----------|
| **EBR** (epoch-based) | After all threads advance epoch | Low (amortized) | Most lock-free DS |
| **Hazard Pointers** | After all hazard pointers release it | Per-pointer scanning | Precise, lower memory |
| **Quiescent State** (RCU) | After all threads hit a quiescent state | Very low | Linux kernel, RCU-protected reads |

---

## Common Mistakes and Traps

1. **Thinking refcounts prevent ABA entirely.** They prevent premature freeing of the *object*, but not ABA on the *pointer* to that object. These are different levels.

2. **Forgetting the "load, then increment" order matters.** In `clone()`, you MUST increment the refcount before returning the new handle. If you return first and then increment, another thread's `drop()` could race you to zero and free the object.

3. **The "acquire" in decrement is critical.** The final `drop()` that reaches `refcount=0` must "acquire" all prior "releases" (other drops). Without `acquire`, the freeing thread might not see writes made by previous holders.

4. **Python's GIL doesn't save you.** The GIL prevents Python bytecode from interleaving, but `threading.Lock()` is at the Python level. In C/C++, these operations are machine instructions and CAN interleave. The pseudocode here is correct for C++.

5. **EBR can starve if a thread stays in a critical section.** If Thread 1 never calls `exit_critical_section()`, the global epoch can never advance and retired blocks pile up forever (memory leak). The fix: make critical sections short (only around pointer loads).

---

## Summary

| Concept | Key Rule |
|---------|---------|
| `clone()` order | Increment refcount BEFORE returning new handle |
| `drop()` memory order | `acq_rel` on decrement: release=sync writes, acquire=see all prior releases |
| ABA problem | Reuse of same memory address fools CAS, even with correct refcount |
| EBR fix | Delay free until all threads have observed epoch advancement (no live raw pointers) |
| Hazard pointers | Alternative: per-thread announcement of specific protected addresses |

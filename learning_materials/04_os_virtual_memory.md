# Lesson 04: OS Internals — Virtual Memory, Page Tables, TLB, and NUMA

> **Level:** Linux kernel source reader. All code samples require Python 3.8+ with `mmap`, `ctypes`, and `os` modules.

---

## 1. The Virtual Memory Abstraction

### Why Virtual Memory Exists

Modern operating systems present every process with an illusion: that it owns the entire machine's address space. This illusion is called **virtual memory**, and it rests on three pillars:

**1. Isolation.** Each process has its own virtual address space. Process A at virtual address `0x7fff_0000` and Process B at the same virtual address `0x7fff_0000` map to completely different physical memory. Without this, a buggy process could corrupt any other process — or the kernel itself. This is the defining safety property of modern multitasking.

**2. Overcommit.** A process can allocate more virtual memory than the machine has physical RAM plus swap. The kernel promises the pages exist; it only commits physical backing when the pages are actually written. A process calling `malloc(200GB)` on a machine with 64GB RAM + 64GB swap will succeed — as long as it never writes more than 128GB. Linux calls this "overcommit" (controlled by `/proc/sys/vm/overcommit_memory`).

**3. Security.** Page table entries carry permission bits. The kernel marks its own pages non-executable from user space (and vice versa in the post-Meltdown world). Hardware enforces these restrictions; no user-space instruction can bypass them without the kernel's cooperation.

### Virtual Address → Physical Address Translation

Without hardware support, translating every virtual address to a physical address would require a software lookup on every memory instruction — every load, every store, every instruction fetch. At ~1 billion memory operations per second, software translation is a deal-breaker.

The **MMU (Memory Management Unit)** is a hardware unit baked into the CPU die. It intercepts every memory access and translates virtual addresses to physical ones using **page tables** stored in RAM. The translation result is cached in the **TLB (Translation Lookaside Buffer)** — a small, fast, associative memory inside the CPU.

### Page Granularity

Memory is managed in fixed-size chunks called **pages**. On x86-64, the default page size is **4KB** (4096 bytes = 2¹²). This is the granularity at which permissions, present/absent bits, and dirty flags are tracked.

- **48-bit virtual address space** (current x86-64 canonical form): 2⁴⁸ = 256TB of virtual space per process. Linux splits this into lower half (user) and upper half (kernel): 128TB each.
- **52-bit physical address space** (current x86-64 hardware limit): 2⁵² = 4PB of addressable physical memory.
- **57-bit virtual addresses** (5-level paging, available since Skylake with `LA57`): 2⁵⁷ = 128PB of virtual space — relevant for memory-intensive ML clusters.

The number of 4KB pages in a 48-bit address space: 2⁴⁸ / 2¹² = **2³⁶ pages = 64 billion pages**. We cannot store a table of 64 billion entries for every process (that would be terabytes per process). This is why page tables are hierarchical and sparse.

---

## 2. x86-64 Four-Level Page Tables

### Virtual Address Breakdown (48-bit)

A 48-bit canonical virtual address is split into five fields:

```
Bits:   [63:48]   [47:39]   [38:30]   [29:21]   [20:12]   [11:0]
Field:  sign-ext  PGD idx   PUD idx   PMD idx   PTE idx   offset
Width:  (16)      9 bits    9 bits    9 bits    9 bits    12 bits
```

- **PGD (Page Global Directory):** Top-level table. 512 entries × 8 bytes = 4KB per process. The physical address of the PGD is stored in **CR3**.
- **PUD (Page Upper Directory):** Second level. Each PGD entry that is present points to a 4KB PUD.
- **PMD (Page Middle Directory):** Third level. Each PUD entry points to a PMD. A PMD entry with bit 7 (PS) set is a **2MB huge page** mapping.
- **PTE (Page Table Entry):** Leaf level. Maps a 4KB physical page.
- **Offset (bits 11:0):** Byte offset within the physical page.

### PTE Format (64-bit entry)

```
Bit 63:    NX (No-Execute) — requires IA32_EFER.NXE=1
Bits 51:12 Physical Frame Number (PFN) — the physical page address >> 12
Bit 11:9   Available to OS (Linux uses for mm flags)
Bit 8:     Global — don't flush on CR3 change (used for kernel pages)
Bit 7:     PS (Page Size) — at PMD level: 1=2MB, at PUD level: 1=1GB
Bit 6:     Dirty — CPU sets when page is written
Bit 5:     Accessed — CPU sets on read or write
Bit 4:     PCD (Page-level Cache Disable)
Bit 3:     PWT (Page-level Write-Through)
Bit 2:     U/S (User/Supervisor) — 0=kernel only, 1=user accessible
Bit 1:     R/W (Read/Write) — 0=read-only, 1=writable
Bit 0:     P (Present) — 0=not mapped (triggers page fault), 1=mapped
```

The hardware page walker reads these bits automatically. The kernel also reads and writes them when managing memory.

### The CR3 Register and Context Switches

**CR3** holds the physical address of the current process's PGD (aligned to 4KB, so only bits 51:12 are used as the address; lower bits carry PCID or flags).

A **context switch** from Process A to Process B requires:

1. Save Process A's register state.
2. Load Process B's CR3 — this atomically switches the entire virtual address space.
3. On older CPUs: this flushes the **entire TLB** (because the old virtual-to-physical mappings are now stale).
4. On CPUs with PCID support: the old ASID/PCID is retained; TLB entries tagged with Process A's PCID remain but are not used by Process B.

### The Page Walk Cost

Without a TLB hit, one memory access requires **5 memory reads**:

```
1 read: PGD entry    (CR3 + PGD_index × 8)
2 read: PUD entry    (PGD.PFN << 12 + PUD_index × 8)
3 read: PMD entry    (PUD.PFN << 12 + PMD_index × 8)
4 read: PTE entry    (PMD.PFN << 12 + PTE_index × 8)
5 read: actual data  (PTE.PFN << 12 + offset)
```

At ~100ns per DRAM access (uncached), a cold page walk costs **~500ns**. At 3GHz, that is **~1500 cycles** — per memory access. This is why the TLB exists, and why TLB misses are catastrophic for performance.

### 5-Level Page Tables (Linux 5.5+, `CONFIG_X86_5LEVEL`)

Intel "Ice Lake" and later CPUs support `LA57` (57-bit Linear Addresses), adding a fifth level called **P4D** between PGD and PUD:

```
Bits: [56:48] P4D idx  (9 bits)
```

This expands virtual address space to **2⁵⁷ = 128PB** per process — the requirement for future HPC and AI workloads with massive datasets. Linux 5.5 enables this at boot if the CPU supports it.

```python
import ctypes
import os

def parse_virtual_address_48bit(va: int) -> dict:
    """Decompose a 48-bit x86-64 virtual address into page table indices."""
    return {
        "pgd_index": (va >> 39) & 0x1FF,
        "pud_index": (va >> 30) & 0x1FF,
        "pmd_index": (va >> 21) & 0x1FF,
        "pte_index": (va >> 12) & 0x1FF,
        "page_offset": va & 0xFFF,
        "raw_hex": hex(va),
    }

# Example: parsing a typical user-space stack address
addr = 0x7FFF_A3B4_C5D6
info = parse_virtual_address_48bit(addr)
for k, v in info.items():
    print(f"  {k:15s}: {v if isinstance(v, str) else hex(v)} ({v if isinstance(v, str) else v})")
```

---

## 3. Translation Lookaside Buffer (TLB)

### TLB Architecture

The TLB is a **hardware associative cache** inside the CPU that stores recently used virtual→physical translations. A modern Intel CPU (e.g., Skylake) has:

- **L1 iTLB:** 128 entries for 4KB pages, 8 entries for 2MB pages (instructions)
- **L1 dTLB:** 64 entries for 4KB pages, 32 entries for 2MB pages (data)
- **L2 STLB (Shared TLB):** 1536 entries for 4KB + 2MB pages

A TLB entry maps `(PCID, virtual_page_number) → (physical_frame_number, permissions)`. On a TLB hit, translation costs **~1 cycle**. On a miss, the hardware page table walker takes over.

### Hardware vs. Software TLB Miss Handling

On **x86-64**, TLB misses trigger the hardware page table walker, which reads CR3 and walks the four levels entirely in microcode. The OS is not invoked for normal TLB misses.

On **MIPS** and some older SPARC processors, TLB misses trap into the kernel. The kernel's TLB refill handler (written in carefully optimized assembly) reads the page table and writes the TLB entry. This adds a few hundred cycles per miss but gives the kernel complete flexibility over the TLB structure.

### PCID (Process Context ID) — The ASID of x86

Without PCID (pre-Haswell or when disabled), every CR3 write flushes all TLB entries. On a server doing thousands of context switches per second, this is an enormous tax.

**PCID** (introduced in Intel Westmere, enabled by Linux 4.14+) is a 12-bit tag stored in the low bits of CR3. TLB entries are tagged with a PCID. When CR3 is loaded with a new PCID, only entries tagged with PCIDs not in use are eligible for replacement — entries from other processes remain valid.

```
CR3 layout with PCID:
Bits [63]:    NOFLUSH — if set, don't invalidate TLB on CR3 write
Bits [51:12]: Physical address of PGD
Bits [11:0]:  PCID (0–4095)
```

Linux uses a pool of PCIDs, recycling them when exhausted (with a generation counter to detect stale entries).

### TLB Shootdown

When the kernel changes a page table mapping (via `munmap`, `mprotect`, `madvise(MADV_DONTNEED)`), it must ensure **all CPUs** that might have cached the old translation flush it. This is called a **TLB shootdown**:

1. CPU 0 modifies PTE (marks page absent or changes permissions).
2. CPU 0 executes `INVLPG <vaddr>` to flush its own TLB for that address.
3. CPU 0 sends an **IPI (Inter-Processor Interrupt)** to all CPUs that may have the mapping (tracked by `mm->cpu_vm_mask`).
4. Each receiving CPU executes its IPI handler, which calls `INVLPG` on the same address.
5. CPU 0 waits for all CPUs to acknowledge before returning from `munmap`.

**Scalability impact:** On a 128-CPU NUMA server, one `munmap()` of a shared mapping triggers 127 IPIs. If 100 threads are doing `munmap` per millisecond (e.g., a memory allocator using `mmap`/`munmap` for large allocations like glibc's `MMAP_THRESHOLD`), that's **12,700 IPIs per millisecond** — a measurable bottleneck.

Solutions: `jemalloc` and `tcmalloc` avoid per-malloc `mmap`/`munmap` by maintaining their own slabs. Linux 5.18 introduced `DEFER_TLB_FLUSH` optimizations.

```python
import mmap
import time
import os

def benchmark_mmap_munmap(n_iterations: int = 1000, size_mb: int = 4) -> float:
    """Measure the average cost of mmap+munmap, which includes TLB shootdown if on SMP."""
    size = size_mb * 1024 * 1024
    times = []
    for _ in range(n_iterations):
        t0 = time.perf_counter_ns()
        m = mmap.mmap(-1, size, flags=mmap.MAP_PRIVATE | mmap.MAP_ANONYMOUS)
        m.close()
        times.append(time.perf_counter_ns() - t0)
    avg_us = sum(times) / len(times) / 1000
    print(f"mmap+munmap ({size_mb}MB): avg {avg_us:.1f}µs over {n_iterations} iterations")
    return avg_us

benchmark_mmap_munmap()
```

---

## 4. Huge Pages

### 2MB Pages: PMD-Level Mapping

When the PS (Page Size) bit is set in a PMD entry, the PMD entry directly maps a **2MB contiguous physical region**. The page walk terminates at PMD level — no PTE lookup.

```
PMD entry (huge page):
Bits [51:21]: Physical address of 2MB-aligned frame
Bit 7 (PS):  1 — "this is a huge page, stop here"
Bits 1, 0:   R/W, Present
```

Benefits:
- **TLB coverage:** One L1 dTLB entry covers 2MB instead of 4KB — 512× more memory per TLB entry. For a process with a 1GB working set: 4KB pages require 262,144 TLB entries (impossible); 2MB pages require 512 entries (fits in L2 STLB).
- **Reduced page walk depth:** 3 levels instead of 4.
- **Fewer page faults:** Fewer PTEs to set up on initial access.

### 1GB Pages (PUD-Level)

Setting PS at the PUD level maps a **1GB** region. Used almost exclusively for NUMA-aware large allocations and high-performance networking (DPDK, RDMA).

### Transparent Huge Pages (THP)

**THP** is the kernel feature that automatically promotes 2MB-aligned groups of 4KB pages into a single 2MB huge page without application changes.

The **`khugepaged`** kernel thread periodically scans VMAs looking for opportunities:
1. Find a 2MB-aligned VMA region where all 512 pages are present and young (recently accessed).
2. Allocate a contiguous 2MB physical region (requires compacting fragmented memory if unavailable).
3. Copy all 512 pages into the new 2MB region.
4. Atomically replace 512 PTEs with one PMD-level entry.

**When THP hurts:**
- **Compaction stalls:** Finding a contiguous 2MB physical region can trigger `kcompactd` to run, moving pages around — causing latency spikes of 10–100ms. This is fatal for latency-sensitive services.
- **False sharing:** A database process reading page 0 of a 2MB region causes the entire 2MB to be faulted in, evicting 512 pages of other data from L3.
- **PostgreSQL, Redis** explicitly disable THP: `echo never > /sys/kernel/mm/transparent_hugepage/enabled`.

```python
def check_thp_status() -> str:
    """Read THP configuration from sysfs."""
    path = "/sys/kernel/mm/transparent_hugepage/enabled"
    try:
        with open(path) as f:
            return f.read().strip()
    except FileNotFoundError:
        return "THP sysfs not available (non-Linux or unprivileged container)"

print("THP status:", check_thp_status())
```

---

## 5. Demand Paging and Page Fault Handling

### The Demand Paging Contract

When a process calls `mmap` or `brk` (heap growth), the kernel creates a **VMA (Virtual Memory Area)** — a descriptor saying "virtual addresses [start, end) with these permissions exist" — but does **not** allocate physical pages. Physical pages are allocated only on first access.

This is **demand paging**: the kernel "demands" proof of access before spending physical memory.

### The Page Fault Path (Linux `do_page_fault`)

When the CPU encounters a virtual address with `PTE.Present = 0` (or the PTE doesn't exist at all), it generates a **page fault exception** (interrupt vector 14 on x86). The CPU saves state and jumps to `do_page_fault` in `arch/x86/mm/fault.c`.

The handler (simplified):

```
1. Read CR2 (faulting virtual address) and error code.
2. Look up the VMA covering the faulting address in mm->mmap (red-black tree).
3. If no VMA: SIGSEGV — invalid access.
4. If VMA has insufficient permissions (write to read-only): SIGSEGV or CoW.
5. Call handle_mm_fault(vma, address, flags):
   a. Walk the page table levels, allocating missing PGD/PUD/PMD entries.
   b. If this is an anonymous page: allocate a new zeroed physical page, write PTE.
   c. If file-backed (mmap of a file): look up the page in the page cache.
      - Found: map it (minor fault).
      - Not found: read from disk, add to page cache, map it (major fault).
6. Return to user space; the faulting instruction is re-executed.
```

### Fault Types

| Fault Type | Cause | Cost | Example |
|---|---|---|---|
| **Minor** | Page in page cache, not mapped | ~1µs | Second access to mmap'd file |
| **Major** | Page must be read from disk | ~5–10ms | First access to cold mmap'd file |
| **Demand zero** | Anonymous page, first write | ~1µs | First write to heap/stack |
| **CoW** | Write to shared read-only page | ~2µs | fork() + first write |

### Prefetching (Readahead)

When a major fault occurs on file-backed pages, the kernel reads ahead adjacent pages into the page cache (`do_page_cache_ra`). The readahead size is controlled by `/proc/sys/vm/page-cluster` (default 3 = 2³ × 4KB = 32KB ahead).

```python
import mmap
import os
import time

def measure_fault_cost(size_bytes: int = 100 * 1024 * 1024) -> dict:
    """
    Measure demand-zero page fault cost by first-touching an anonymous mmap.
    Each write to a new 4KB-aligned offset triggers one page fault.
    """
    m = mmap.mmap(-1, size_bytes, flags=mmap.MAP_PRIVATE | mmap.MAP_ANONYMOUS)
    n_pages = size_bytes // 4096

    start = time.perf_counter_ns()
    for i in range(0, size_bytes, 4096):
        m[i] = 0xFF
    elapsed_ns = time.perf_counter_ns() - start

    result = {
        "total_pages": n_pages,
        "total_ms": elapsed_ns / 1e6,
        "ns_per_fault": elapsed_ns / n_pages,
    }
    m.close()
    print(f"Demand-zero faults: {n_pages} pages, "
          f"{result['total_ms']:.1f}ms total, "
          f"{result['ns_per_fault']:.0f}ns/fault")
    return result

measure_fault_cost()
```

---

## 6. Copy-on-Write (CoW) and `fork()`

### How `fork()` Works with CoW

`fork()` creates an exact copy of the calling process. Naively, this means copying every mapped page — potentially gigabytes. CoW avoids this:

1. `fork()` is called. The kernel clones the parent's page tables.
2. **All writable pages in both parent and child are marked read-only.** Each page's `reference count` (in `struct page`) is incremented to 2.
3. Both processes return from `fork()` — instantly, regardless of address space size.
4. When either process writes to a shared page:
   - Page fault fires (write to read-only page).
   - Kernel allocates a **new physical page**, copies the original's content.
   - The writer's PTE is updated to point to the new page, marked read-write.
   - The original page's refcount drops to 1 and is also marked read-write again (single owner).

The key insight: **`fork()` is O(1) in time and space**. The cost is deferred until actual writes.

### Measuring CoW with `os.fork()`

```python
import mmap
import os
import struct
import time

def measure_cow_fork():
    """
    Demonstrate CoW: parent forks, child writes to shared pages.
    Measure how long CoW page splitting takes for 100MB of memory.
    """
    SIZE = 100 * 1024 * 1024  # 100MB

    # Allocate and first-touch all pages in parent
    m = mmap.mmap(-1, SIZE, flags=mmap.MAP_SHARED | mmap.MAP_ANONYMOUS)
    for i in range(0, SIZE, 4096):
        m[i] = 0xAA  # Populate all pages in parent

    pid = os.fork()
    if pid == 0:
        # CHILD: write to every page — each triggers a CoW fault
        start = time.perf_counter_ns()
        for i in range(0, SIZE, 4096):
            m[i] = 0xBB
        elapsed_ms = (time.perf_counter_ns() - start) / 1e6
        print(f"[child pid={os.getpid()}] CoW split {SIZE//4096} pages: {elapsed_ms:.1f}ms")
        m.close()
        os._exit(0)
    else:
        # PARENT: verify data unchanged
        _, status = os.waitpid(pid, 0)
        assert m[0] == 0xAA, "CoW failed: parent's page was corrupted!"
        print(f"[parent] Data integrity verified after child's CoW writes.")
        m.close()

measure_cow_fork()
```

### The Redis CoW Problem (in Detail)

Redis does persistence via `BGSAVE`: it calls `fork()` and the child serializes all data to disk while the parent continues serving writes.

Timeline:
- `t=0`: Redis parent has 10GB RSS. Fork is instant.
- `t=0+ε`: Child begins iterating all keys for serialization.
- `t=1s–60s`: Parent continues writing (SET commands). Each modified key's slab triggers CoW. New physical pages are allocated.
- `t=60s`: BGSAVE completes. Parent may now have **5–8GB of additional RSS** due to CoW copies.

This is why Redis recommends: `vm.overcommit_memory = 1` (so fork doesn't fail) and monitoring for `mem_allocator:jemalloc` CoW metrics. Redis 7+ attempts to use `fork()` less aggressively (replication offset tracking).

```python
import mmap
import os
import time

def simulate_redis_bgsave_cow(data_mb: int = 50, write_fraction: float = 0.3):
    """
    Simulate Redis's BGSAVE CoW overhead.
    Parent keeps writing while child 'saves' (reads all pages).
    Measure total physical memory inflation.
    """
    SIZE = data_mb * 1024 * 1024
    m = mmap.mmap(-1, SIZE, flags=mmap.MAP_SHARED | mmap.MAP_ANONYMOUS)

    # Pre-populate (simulate Redis loaded state)
    for i in range(0, SIZE, 4096):
        m[i] = 0xDD

    pid = os.fork()
    if pid == 0:
        # CHILD: "serialize" — just read all pages
        checksum = 0
        for i in range(0, SIZE, 4096):
            checksum ^= m[i]
        print(f"[child] BGSAVE complete. Checksum={checksum}")
        m.close()
        os._exit(0)
    else:
        # PARENT: simulate write traffic during BGSAVE
        write_pages = int((SIZE // 4096) * write_fraction)
        cow_start = time.perf_counter_ns()
        for i in range(0, write_pages * 4096, 4096):
            m[i] = 0xEE  # Each write triggers a CoW fault
        cow_ms = (time.perf_counter_ns() - cow_start) / 1e6
        print(f"[parent] Wrote {write_pages} pages during BGSAVE: {cow_ms:.1f}ms CoW overhead")
        os.waitpid(pid, 0)
        m.close()

simulate_redis_bgsave_cow()
```

---

## 7. Python `mmap` Module — Deep Dive

### Anonymous vs. File-Backed mmaps

```python
import mmap
import os
import struct
import time

# --- 1. Anonymous mmap (heap-like, backed by swap) ---
anon = mmap.mmap(-1, 100 * 1024 * 1024,  # 100MB
                 flags=mmap.MAP_PRIVATE | mmap.MAP_ANONYMOUS)
# Pages are demand-zeroed — no physical memory consumed yet
anon[0:4] = b'\xDE\xAD\xBE\xEF'
assert anon[0:4] == b'\xDE\xAD\xBE\xEF'
anon.close()

# --- 2. File-backed mmap for IPC ---
IPC_FILE = '/tmp/mmap_ipc_demo'
with open(IPC_FILE, 'w+b') as f:
    f.write(b'\x00' * 4096)
    f.flush()
    shared = mmap.mmap(f.fileno(), 4096, flags=mmap.MAP_SHARED)

# Writer: pack a 32-bit big-endian counter at offset 0
struct.pack_into('>I', shared, 0, 999)
shared.flush()  # msync(MS_SYNC) — ensure kernel writes to page cache

# Reader (same process, but could be another process opening same file)
val = struct.unpack_from('>I', shared, 0)[0]
print(f"IPC value read back: {val}")  # 999

shared.close()
os.unlink(IPC_FILE)

# --- 3. mmap.ACCESS_READ for memory-efficient file reads ---
# Map a large file read-only; pages loaded on access via page cache
with open('/proc/self/maps', 'rb') as f:
    content = f.read()
    # For a real large file: mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ)
    print(f"Process maps size: {len(content)} bytes")

# --- 4. Page fault timing — demand zero ---
def measure_first_touch(size_mb: int = 100) -> None:
    size = size_mb * 1024 * 1024
    m = mmap.mmap(-1, size)
    start = time.perf_counter()
    for i in range(0, size, 4096):
        m[i] = 1
    elapsed = time.perf_counter() - start
    n_pages = size // 4096
    print(f"First-touch {size_mb}MB: {elapsed*1000:.1f}ms, "
          f"{elapsed*1e9/n_pages:.0f}ns/page, "
          f"{n_pages} faults")
    m.close()

measure_first_touch(100)

# --- 5. madvise equivalents in Python ---
def madvise_sequential(m: mmap.mmap, size: int) -> None:
    """Hint to kernel: sequential access pattern (triggers aggressive readahead)."""
    try:
        m.madvise(mmap.MADV_SEQUENTIAL, 0, size)
    except AttributeError:
        pass  # madvise not available on this platform

# --- 6. Pinning pages with mlock (requires root or CAP_IPC_LOCK) ---
def try_mlock_demo():
    """mlock prevents pages from being swapped. Critical for real-time processes."""
    import ctypes
    libc = ctypes.CDLL("libc.so.6", use_errno=True)
    m = mmap.mmap(-1, 4096)
    m[0] = 1
    ptr = ctypes.c_void_p(ctypes.addressof(ctypes.c_char.from_buffer(m)))
    ret = libc.mlock(ptr, 4096)
    if ret == 0:
        print("mlock succeeded — page is wired in RAM")
    else:
        errno = ctypes.get_errno()
        print(f"mlock returned errno={errno} (EPERM=1, need CAP_IPC_LOCK or ulimit -l)")
    m.close()

try_mlock_demo()
```

### Reading `/proc/self/smaps` for Page Accounting

```python
def parse_smaps(pid: int = None) -> list[dict]:
    """
    Parse /proc/<pid>/smaps to get per-VMA memory breakdown.
    Key fields: Size, Rss (resident), Pss (proportional shared), Private_Dirty.
    """
    pid_str = str(pid) if pid else "self"
    path = f"/proc/{pid_str}/smaps"
    regions = []
    current = {}
    try:
        with open(path) as f:
            for line in f:
                line = line.rstrip()
                if not line:
                    continue
                if '-' in line.split()[0]:  # VMA header line
                    if current:
                        regions.append(current)
                    parts = line.split()
                    current = {
                        "range": parts[0],
                        "perms": parts[1],
                        "name": parts[-1] if len(parts) > 5 else "[anon]",
                    }
                elif ':' in line:
                    key, _, val = line.partition(':')
                    val_stripped = val.strip().split()[0] if val.strip() else '0'
                    try:
                        current[key.strip()] = int(val_stripped)
                    except ValueError:
                        current[key.strip()] = val.strip()
        if current:
            regions.append(current)
    except PermissionError:
        print(f"Cannot read {path}: need same UID or root")
        return []
    return regions

def summarize_smaps():
    """Print top VMAs by Private_Dirty (CoW-copied or modified anonymous pages)."""
    regions = parse_smaps()
    # Filter to regions with non-zero Private_Dirty
    dirty = [(r.get('Private_Dirty', 0), r) for r in regions]
    dirty.sort(reverse=True)
    print(f"\n{'Private_Dirty':>15} {'RSS':>10} {'PSS':>10} {'Name'}")
    print("-" * 60)
    for kb, r in dirty[:10]:
        print(f"{kb:>12} KB  {r.get('Rss',0):>7} KB  {r.get('Pss',0):>7} KB  {r.get('name','?')}")

summarize_smaps()
```

---

## 8. NUMA (Non-Uniform Memory Access)

### The Hardware Reality

A dual-socket server has two CPUs, each with their own DDR4/DDR5 memory controllers and local DIMMs. Memory latency is **not uniform**:

| Access Type | Latency (typical) |
|---|---|
| L1 cache | ~4 cycles (~1.3ns) |
| L2 cache | ~12 cycles (~4ns) |
| L3 cache | ~40 cycles (~13ns) |
| Local DRAM (same NUMA node) | ~80ns |
| Remote DRAM (cross-socket, 2-hop QPI/UPI) | ~130–160ns |
| Remote DRAM (3-hop, 4-socket server) | ~200–250ns |

The ratio of remote to local DRAM latency is the **NUMA factor** (~2× for dual-socket, up to 3× for 4-socket).

### Linux NUMA Policies

The kernel tracks NUMA topology in `struct pg_data_t` (one per NUMA node). Each page has a "home node" where it physically resides.

**Allocation policies** (set per-VMA via `mbind()` syscall or per-process via `set_mempolicy()`):

| Policy | Description |
|---|---|
| `MPOL_DEFAULT` / `MPOL_LOCAL` | Allocate on the node local to the allocating CPU (best for most workloads) |
| `MPOL_BIND` | Hard-bind: only allocate from specified node(s); fail if unavailable |
| `MPOL_PREFERRED` | Try specified node first, fall back to others |
| `MPOL_INTERLEAVE` | Round-robin across nodes (good for large shared data accessed by all CPUs) |

**NUMA and process migration:** The Linux scheduler may migrate a thread to a less-loaded CPU on another NUMA node. The thread's memory stays on the original node. Now every memory access is remote. `numactl --cpunodebind=0 --membind=0` prevents this.

### Detecting NUMA Topology in Python

```python
import os
import ctypes
import mmap
import time

def get_numa_topology() -> dict:
    """Read NUMA topology from sysfs."""
    topology = {}
    numa_path = "/sys/devices/system/node"
    if not os.path.isdir(numa_path):
        return {"numa_available": False}
    
    nodes = sorted(d for d in os.listdir(numa_path) if d.startswith("node"))
    topology["n_nodes"] = len(nodes)
    topology["nodes"] = {}
    
    for node in nodes:
        node_path = f"{numa_path}/{node}"
        node_info = {}
        
        # Read CPUs on this node
        cpu_list_path = f"{node_path}/cpulist"
        if os.path.exists(cpu_list_path):
            with open(cpu_list_path) as f:
                node_info["cpus"] = f.read().strip()
        
        # Read memory size
        meminfo_path = f"{node_path}/meminfo"
        if os.path.exists(meminfo_path):
            with open(meminfo_path) as f:
                for line in f:
                    if "MemTotal" in line:
                        parts = line.split()
                        node_info["mem_kb"] = int(parts[-2])
                        break
        
        topology["nodes"][node] = node_info
    
    return topology

numa_info = get_numa_topology()
print(f"NUMA nodes: {numa_info.get('n_nodes', 'N/A')}")
for node, info in numa_info.get("nodes", {}).items():
    print(f"  {node}: CPUs={info.get('cpus','?')}, "
          f"Mem={info.get('mem_kb',0)//1024}MB")

def measure_numa_local_vs_remote_latency():
    """
    Measure the cost of pointer chasing on local vs. remote NUMA memory.
    (Approximation using mmap — true NUMA measurement requires mbind() and numactl.)
    """
    SIZE = 64 * 1024 * 1024  # 64MB — larger than L3, forces DRAM access
    STEPS = 10_000_000
    STRIDE = 64  # cache line size

    m = mmap.mmap(-1, SIZE)
    # First touch (allocates on local NUMA node by default)
    for i in range(0, SIZE, 4096):
        m[i] = 0

    # Pointer chase to measure DRAM latency (not cache)
    # Access pattern: stride through mmap to avoid L1/L2/L3 cache hits
    start = time.perf_counter_ns()
    idx = 0
    for _ in range(STEPS):
        _ = m[idx % SIZE]
        idx = (idx + STRIDE) & (SIZE - 1)  # stride access
    elapsed_ns = time.perf_counter_ns() - start
    ns_per_access = elapsed_ns / STEPS
    print(f"Strided DRAM access: {ns_per_access:.1f}ns/access (local NUMA expected ~60-100ns)")
    m.close()

measure_numa_local_vs_remote_latency()
```

### NUMA and Python Runtime Pitfalls

1. **GIL and NUMA:** CPython's GIL is a mutex in the interpreter's data segment. On a 4-socket machine, if the allocating CPU was on node 0, every thread on nodes 1–3 contends for a lock on remote memory.

2. **Process pool and NUMA:** `multiprocessing.Pool` with 64 workers on a 4-socket server will have workers on nodes 1, 2, 3 whose parent-allocated objects live on node 0. Use `numactl` to pin each worker to its local node.

3. **NumPy arrays and NUMA:** `np.zeros((10_000, 10_000))` allocated by the main thread on node 0 is slow for worker threads on node 1. Use `numa_alloc_local()` via `ctypes` + `libnuma` for critical arrays.

```python
def try_mbind_interleave(size_mb: int = 100) -> None:
    """
    Use mbind() to interleave a large allocation across all NUMA nodes.
    This is the ideal policy for large shared read-mostly data.
    Requires libc with NUMA support.
    """
    libc = ctypes.CDLL("libc.so.6", use_errno=True)
    size = size_mb * 1024 * 1024
    
    m = mmap.mmap(-1, size)
    for i in range(0, size, 4096):
        m[i] = 0  # First touch (current policy)
    
    # mbind with MPOL_INTERLEAVE would be called here for true NUMA control
    # Requires: ctypes setup for sys_mbind with nodemask
    # Example pseudocode (actual implementation requires careful nodemask setup):
    # MPOL_INTERLEAVE = 3
    # nodemask = ctypes.c_ulong(0x3)  # nodes 0 and 1
    # ret = libc.syscall(237, ptr, size, MPOL_INTERLEAVE, ctypes.byref(nodemask), 2, 0)
    print(f"mbind interleave for {size_mb}MB: requires CAP_SYS_NICE or MPOL_MF_MOVE rights")
    m.close()

try_mbind_interleave()
```

---

## 9. What Textbooks Don't Tell You

### `struct mm_struct` — The Kernel's View of a Process's Memory

Every process has a `struct mm_struct` (in `include/linux/mm_types.h`) that describes its entire virtual address space:

```c
// Simplified from linux/mm_types.h
struct mm_struct {
    struct maple_tree  mm_mt;       // VMA tree (Linux 6.1+: maple tree, was red-black)
    unsigned long      mmap_base;   // Base of mmap region
    unsigned long      task_size;   // Size of user address space
    pgd_t             *pgd;         // PGD — loaded into CR3 on context switch
    atomic_t           mm_users;    // Number of threads using this mm
    atomic_t           mm_count;    // Number of references (including kernel)
    unsigned long      hiwater_rss; // High-water mark of RSS
    unsigned long      total_vm;    // Total pages mapped
    unsigned long      locked_vm;   // Pages locked with mlock()
    unsigned long      data_vm;     // Pages for data segments
    unsigned long      stack_vm;    // Pages for stack
    // ... 200+ more fields
};
```

`mmap()` inserts a `struct vm_area_struct` (VMA) into this tree. `munmap()` removes or splits VMAs. Lookup on page fault is **O(log n)** where n is the number of distinct VMAs. A typical process has ~50–200 VMAs. A complex process (Python with many libraries) may have 500–2000.

### Stack Growth: `VM_GROWSDOWN`

The stack VMA has the flag `VM_GROWSDOWN`. When the CPU generates a page fault at an address just below the current stack bottom:

1. Kernel checks: is the faulting address within `STACK_GUARD_GAP` (1MB) below the stack VMA?
2. If yes: extend the VMA downward by one page, allocate the page, return.
3. If the stack would exceed `ulimit -s` (default 8MB): SIGSEGV — stack overflow.

This is why the stack "grows" without the programmer doing anything — the OS extends the VMA on each page fault.

### Overcommit and the OOM Killer

```
/proc/sys/vm/overcommit_memory:
  0: heuristic overcommit (default) — allow reasonable overcommit
  1: always overcommit — any allocation succeeds regardless of RAM
  2: never overcommit — only allocate up to RAM + swap * overcommit_ratio/100

/proc/sys/vm/overcommit_ratio: default 50 (50% of swap can be overcommitted)
```

When physical memory is truly exhausted (all pages allocated, swap full), the kernel runs the **OOM (Out-Of-Memory) killer**. It scores each process with `oom_score` (based on RSS, swap usage, runtime, `oom_score_adj`). The highest-scored process is sent `SIGKILL`.

```python
def read_oom_scores(top_n: int = 5) -> list[tuple[int, str, int]]:
    """Read OOM scores for all processes. Higher = more likely to be killed."""
    results = []
    for pid_str in os.listdir('/proc'):
        if not pid_str.isdigit():
            continue
        try:
            score_path = f"/proc/{pid_str}/oom_score"
            comm_path = f"/proc/{pid_str}/comm"
            with open(score_path) as f:
                score = int(f.read().strip())
            with open(comm_path) as f:
                comm = f.read().strip()
            results.append((score, comm, int(pid_str)))
        except (FileNotFoundError, PermissionError, ProcessLookupError):
            pass
    results.sort(reverse=True)
    print(f"\n{'Score':>8} {'PID':>8} {'Command'}")
    for score, comm, pid in results[:top_n]:
        print(f"{score:>8} {pid:>8} {comm}")
    return results[:top_n]

read_oom_scores()
```

### KPTI (Kernel Page Table Isolation) — The Meltdown Mitigation

**Meltdown** (CVE-2017-5754): A CPU speculative execution bug allowed user-space code to read kernel memory via cache timing side channels. The kernel's page table (including kernel mappings) was previously accessible to user-space processes (just not executable — but readable by speculation).

**KPTI fix:** Split the page table into two:

1. **User-space PT:** Contains user pages + minimal kernel stubs (syscall entry points, interrupt handlers). No kernel data.
2. **Kernel PT:** Full mapping of all kernel memory.

On every syscall/interrupt: `CR3` switches from user PT to kernel PT. On return to user space: `CR3` switches back. This is **2 CR3 writes per syscall** = 2 TLB flushes (or 2 PCID switches with KPTI+PCID optimization).

**Performance cost:** ~5–30% overhead on syscall-heavy workloads (databases, servers). Intel CPUs with hardware Meltdown mitigation (Ice Lake+) remove most of this overhead.

```python
def check_kpti_status() -> str:
    """Check if KPTI is active on this Linux system."""
    # Method 1: dmesg
    try:
        with open('/proc/cmdline') as f:
            cmdline = f.read()
            if 'nopti' in cmdline:
                return "KPTI DISABLED (nopti kernel parameter)"
    except:
        pass
    
    # Method 2: CPU vulnerabilities sysfs (Linux 4.14.2+)
    meltdown_path = "/sys/devices/system/cpu/vulnerabilities/meltdown"
    if os.path.exists(meltdown_path):
        with open(meltdown_path) as f:
            return f"Meltdown mitigation: {f.read().strip()}"
    
    return "Cannot determine KPTI status"

print(check_kpti_status())
```

### `userfaultfd` — Kernel-Level Page Fault Handling in User Space

Linux 4.3 introduced `userfaultfd`, a file descriptor that lets a user-space handler receive page fault notifications and supply pages:

```python
import ctypes
import os

def userfaultfd_demo_concept():
    """
    Conceptual demonstration of userfaultfd.
    Real usage requires: ioctl(UFFDIO_API), ioctl(UFFDIO_REGISTER), poll().
    Used by: QEMU (VM live migration), Redis (background save optimization),
             checkpoint/restore (CRIU), copy-on-fork optimizations.
    """
    # userfaultfd syscall number on x86-64
    SYS_USERFAULTFD = 323
    libc = ctypes.CDLL("libc.so.6", use_errno=True)
    
    # Create userfaultfd (O_NONBLOCK | O_CLOEXEC = 0x800 | 0x80000)
    fd = libc.syscall(SYS_USERFAULTFD, 0x800 | 0x80000)
    if fd < 0:
        errno = ctypes.get_errno()
        print(f"userfaultfd failed: errno={errno} (need kernel 4.3+ and CAP_SYS_PTRACE or sysctl)")
        return
    
    print(f"userfaultfd fd={fd} created")
    print("Use cases: QEMU live migration, CRIU checkpointing, lazy page population")
    os.close(fd)

userfaultfd_demo_concept()
```

---

## 10. Hard Exercise

### Problem Statement

A Python web server (Gunicorn-style) forks **8 worker processes** on startup. The parent process has already loaded a 200MB machine learning model into memory (a large `dict` mapping strings to `np.ndarray` objects). Each worker immediately begins handling HTTP requests. After 5 minutes under load, `top` shows each worker using **250MB RSS**.

**Questions:**

**(a) Explain why each worker uses 250MB RSS, not 200MB.**

The base 200MB comes from the model loaded in the parent. The additional 50MB comes from:
- Worker-local objects: request state, response buffers, thread-local storage, Python interpreter overhead per-worker.
- **Python reference counting CoW-copy:** Every time Python access a dict key or array element, it increments `ob_refcnt` (the reference count field in `PyObject`). This is a **write** to the object's memory — even for a read-only lookup. This write triggers a CoW page fault: the page containing the `ob_refcnt` field is copied. Since model objects are accessed constantly (every request), almost every page gets CoW-copied.

**(b) Why is memory being copied even though workers are read-only after startup?**

Python's garbage collector is reference-count based. Every attribute access, every function call, every container lookup does `Py_INCREF` / `Py_DECREF`. `ob_refcnt` is the first 8 bytes of every Python object (`PyObject.ob_refcnt`). When a worker accesses even a single key in the model dict, the `PyObject` representing that key has its refcount incremented — dirtying the page. Since the page was shared CoW (read-only), this triggers a page fault and a copy.

Even "reading" the model requires refcount mutations, making pure read-only sharing of Python objects impossible in the standard CPython interpreter.

**(c) Two solutions to reduce total memory usage:**

**Solution 1: Use `mmap` + serialized format (Pickle/NumPy `.npy` memmap)**
Instead of a Python dict in RAM, store the model as memory-mapped files:
```python
import numpy as np
import mmap

# Parent pre-saves model as memory-mapped NumPy arrays
np.save('/tmp/model_weights.npy', weights_array)

# Each worker maps it read-only — shared pages, no CoW
weights = np.load('/tmp/model_weights.npy', mmap_mode='r')
# Accessing weights reads from the shared file-backed mmap — no refcount mutations
# because it's a raw NumPy buffer, not a Python object per element
```
This sidesteps CPython refcounting: NumPy arrays backed by `mmap` with `mmap_mode='r'` use read-only mmaps. As long as workers only read (no Python-level attribute access per element), pages stay shared.

**Solution 2: Use `multiprocessing.shared_memory` (Python 3.8+)**
```python
from multiprocessing import shared_memory
import numpy as np

# Parent creates shared memory block
shm = shared_memory.SharedMemory(create=True, size=model_array.nbytes)
shared_arr = np.ndarray(model_array.shape, dtype=model_array.dtype, buffer=shm.buf)
shared_arr[:] = model_array

# Workers attach to the same block — one physical copy in RAM
shm_worker = shared_memory.SharedMemory(name=shm.name)
worker_arr = np.ndarray(model_array.shape, dtype=model_array.dtype, buffer=shm_worker.buf)
# worker_arr is a view into the same physical pages — no CoW
```

**(d) Measuring shared vs. private pages using `/proc/<pid>/smaps`:**

```python
def analyze_process_sharing(pid: int) -> dict:
    """
    Read /proc/<pid>/smaps and compute shared vs. private memory.
    
    Key metrics:
    - Rss:          Total resident pages (shared + private)
    - Shared_Clean: Pages shared with other processes, not modified
    - Shared_Dirty: Pages shared but modified (rare)
    - Private_Clean: Private pages not modified (e.g., read-only mapped files)
    - Private_Dirty: Pages that have been CoW-copied and written
    - Pss:          Proportional Share Size = Private + (Shared / n_sharers)
    """
    path = f"/proc/{pid}/smaps"
    stats = {
        "Rss": 0, "Pss": 0,
        "Shared_Clean": 0, "Shared_Dirty": 0,
        "Private_Clean": 0, "Private_Dirty": 0,
    }
    try:
        with open(path) as f:
            for line in f:
                for key in stats:
                    if line.startswith(key + ":"):
                        val = line.split()[1]
                        stats[key] += int(val)
    except PermissionError:
        return {"error": f"Cannot read {path}: need same UID or root"}
    
    total_shared = stats["Shared_Clean"] + stats["Shared_Dirty"]
    total_private = stats["Private_Clean"] + stats["Private_Dirty"]
    
    print(f"\nProcess {pid} memory breakdown (all values in KB):")
    print(f"  RSS (total resident): {stats['Rss']:>10,}")
    print(f"  PSS (proportional):   {stats['Pss']:>10,}")
    print(f"  Shared (unmodified):  {stats['Shared_Clean']:>10,}")
    print(f"  Private CoW-dirtied:  {stats['Private_Dirty']:>10,}  ← This is your CoW cost")
    print(f"  CoW overhead:         {stats['Private_Dirty'] / max(stats['Rss'],1) * 100:.1f}%")
    
    return stats

# Analyze current process
result = analyze_process_sharing(os.getpid())

# To compare workers, run:
# for worker_pid in worker_pids:
#     analyze_process_sharing(worker_pid)
# The delta between RSS and PSS × n_workers tells you total memory waste from CoW
```

### Practical Debugging Workflow

```python
def full_memory_audit(pid: int = None) -> None:
    """Complete memory audit: smaps + status + maps."""
    pid = pid or os.getpid()
    
    # 1. High-level stats from /proc/pid/status
    status_path = f"/proc/{pid}/status"
    vm_stats = {}
    with open(status_path) as f:
        for line in f:
            for key in ("VmRSS", "VmSize", "VmSwap", "VmPeak", "RssAnon", "RssFile", "RssShmem"):
                if line.startswith(key + ":"):
                    vm_stats[key] = line.split()[1] + " " + (line.split()[2] if len(line.split())>2 else "kB")
    
    print(f"\n=== Memory Audit: PID {pid} ===")
    for k, v in vm_stats.items():
        print(f"  {k:15}: {v}")
    
    # 2. Top VMAs by RSS
    regions = parse_smaps(pid)
    regions_by_rss = sorted(regions, key=lambda r: r.get("Rss", 0), reverse=True)
    print(f"\n  Top VMAs by RSS:")
    for r in regions_by_rss[:5]:
        print(f"    {r.get('range','?'):30} {r.get('perms','?')} "
              f"RSS={r.get('Rss',0):6}KB  Private_Dirty={r.get('Private_Dirty',0):6}KB  "
              f"{r.get('name','[anon]')}")

full_memory_audit()
```

---

## Quick-Reference: Key System Parameters

| Parameter | Path | Default | Effect |
|---|---|---|---|
| Overcommit mode | `/proc/sys/vm/overcommit_memory` | `0` | 0=heuristic, 1=always, 2=never |
| THP | `/sys/kernel/mm/transparent_hugepage/enabled` | `madvise` | `always`/`madvise`/`never` |
| Dirty ratio | `/proc/sys/vm/dirty_ratio` | `20` | Max % of RAM as dirty pages before sync |
| Swappiness | `/proc/sys/vm/swappiness` | `60` | 0=avoid swap, 100=aggressive swap |
| OOM adj | `/proc/<pid>/oom_score_adj` | `0` | -1000=never kill, 1000=kill first |
| Readahead pages | `/proc/sys/vm/page-cluster` | `3` | 2^N pages read ahead on major fault |
| NUMA balancing | `/proc/sys/kernel/numa_balancing` | `1` | Auto NUMA page migration |

---

## Summary

Virtual memory is the foundational abstraction of modern OS design. Understanding its implementation — four-level page tables, TLB shootdowns, huge pages, CoW semantics, NUMA topology — is essential for reasoning about performance in any language, including Python. The `mmap` module exposes these primitives directly: anonymous mmaps exercise demand paging, shared mmaps enable zero-copy IPC, and `os.fork()` demonstrates CoW in action. Profiling tools like `/proc/<pid>/smaps` let you measure exactly how well your architecture exploits physical memory sharing.

The performance limits of Python web servers, ML serving systems, and databases all ultimately trace back to the mechanisms described in this lesson: CoW-induced refcount copies, TLB pressure from fragmented virtual address spaces, NUMA-remote allocations from the wrong CPU, and kernel overhead from KPTI. Knowing the kernel source gives you the vocabulary — and the Python tools above give you the instruments — to measure, understand, and fix them.

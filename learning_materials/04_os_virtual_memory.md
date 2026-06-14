# Lesson 04: OS Internals — Virtual Memory, Page Tables, TLB, and NUMA

> **Level:** Linux kernel source reader. All code samples require Python 3.8+ with `mmap`, `ctypes`, and `os` modules.

---

## 0. Complete Virtual Address Space Map (x86-64, Linux)

Before diving into mechanisms, here is the full picture of a 64-bit Linux process's virtual address space:

```
Virtual Address Space Layout (x86-64, 48-bit canonical, ASLR enabled)
=======================================================================

0xFFFF_FFFF_FFFF_FFFF ┐
        ...            │  KERNEL SPACE (upper half, inaccessible to user)
0xFFFF_8000_0000_0000 ┘  (~128TB kernel virtual space)

  ── CANONICAL HOLE (non-canonical addresses — hardware fault on access) ──

0x0000_7FFF_FFFF_FFFF ┐
        ...            │  USER SPACE (lower half, 128TB)
                       │
   ~0x7FFF_xxxx_xxxx  │  [stack]         grows DOWN (VM_GROWSDOWN)
                       │  [stack guard]   1MB gap (STACK_GUARD_GAP)
        ...            │  [mmap region]   grows DOWN from mmap_base
                       │    ├─ anonymous mmaps (malloc large allocations)
                       │    ├─ shared libraries (.so text + data segments)
                       │    ├─ file-backed mmaps
                       │    └─ VDSO, VVAR (kernel-provided user pages)
                       │
   ~0x5600_xxxx_xxxx  │  [heap]          grows UP via brk()
   ~0x5555_xxxx_xxxx  │  [BSS]           uninitialized static data (demand zero)
   ~0x5555_xxxx_xxxx  │  [data]          initialized static data
   ~0x5555_xxxx_xxxx  │  [text]          executable code (r-xp, PIE-randomized)
0x0000_0000_0000_0000 ┘  (NULL — page 0 is never mapped, catches null deref)

KEY: addresses are ASLR-randomized at each exec(); shown as typical ranges.
```

```python
import os, re

def print_address_space_layout() -> None:
    """Print this process's actual VMA layout with annotations."""
    category_map = {
        "[stack]": "STACK (grows down)",
        "[heap]":  "HEAP  (brk grows up)",
        "[vdso]":  "VDSO  (kernel-provided syscall stubs)",
        "[vvar]":  "VVAR  (kernel variables: clock, vsyscall data)",
        "[vsyscall]": "VSYSCALL (legacy fixed mapping)",
    }
    
    with open("/proc/self/maps") as f:
        lines = f.readlines()
    
    print(f"\n{'Address Range':>42}  {'Perms':<5}  {'Size':>7}  Description")
    print("─" * 85)
    for line in lines:
        parts = line.split()
        addr_range, perms = parts[0], parts[1]
        name = parts[-1] if len(parts) > 5 else ""
        start, end = [int(x, 16) for x in addr_range.split("-")]
        size_kb = (end - start) // 1024
        
        # Annotate
        desc = category_map.get(name, "")
        if not desc:
            if name.endswith(".so") or ".so." in name:
                desc = f"shared lib: {os.path.basename(name)}"
            elif name == "" and "x" in perms:
                desc = "anon exec (JIT?)"
            elif name == "":
                desc = "anon mmap"
            else:
                desc = os.path.basename(name)
        
        print(f"  {addr_range:>42}  {perms:<5}  {size_kb:>5}KB  {desc[:40]}")
    print(f"\nTotal VMAs: {len(lines)}")

print_address_space_layout()
```

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

### Reading Physical Addresses via `/proc/self/pagemap`

Linux exposes a binary interface at `/proc/<pid>/pagemap` that maps virtual page numbers to physical frame numbers — the same information the hardware uses during a page walk. This lets you verify that two processes share the same physical page (CoW before first write) or have diverged (after CoW split).

```python
import struct
import os
import mmap

def virt_to_phys(vaddr: int, pid: int = None) -> dict:
    """
    Translate a virtual address to physical address using /proc/pid/pagemap.
    
    /proc/pid/pagemap layout: 8 bytes per page entry, indexed by VPN.
      Bit 63:    page present in RAM
      Bit 62:    page swapped out
      Bits 54:0: page frame number (PFN) if present; swap info if swapped
    
    Requires same UID or CAP_SYS_ADMIN (PFN access gated since Linux 4.0).
    """
    pid_str = str(pid) if pid else "self"
    page_size = os.sysconf("SC_PAGE_SIZE")
    vpn = vaddr // page_size
    offset_in_file = vpn * 8  # 8 bytes per entry

    result = {"vaddr": hex(vaddr), "vpn": vpn, "pfn": None, "present": False}
    try:
        with open(f"/proc/{pid_str}/pagemap", "rb") as f:
            f.seek(offset_in_file)
            data = f.read(8)
            if len(data) < 8:
                result["error"] = "short read"
                return result
            entry = struct.unpack("<Q", data)[0]
            present = bool(entry >> 63)
            swapped = bool((entry >> 62) & 1)
            pfn = entry & ((1 << 55) - 1)
            result.update({
                "present": present,
                "swapped": swapped,
                "pfn": pfn if present else None,
                "phys_addr": hex(pfn * page_size + (vaddr % page_size)) if present else None,
            })
    except PermissionError:
        result["error"] = "EPERM: need CAP_SYS_ADMIN for PFN (Linux 4.0+)"
    return result

def demonstrate_cow_physical_sharing():
    """
    Fork a process and compare physical addresses before and after a CoW write.
    Before the child writes: parent and child should share the same PFN.
    After the child writes: PFNs diverge — CoW split has occurred.
    """
    SIZE = 4096  # one page
    m = mmap.mmap(-1, SIZE, flags=mmap.MAP_SHARED | mmap.MAP_ANONYMOUS)
    m[0] = 0xAA

    vaddr = ctypes.addressof(ctypes.c_char.from_buffer(m))
    parent_phys_before = virt_to_phys(vaddr)
    print(f"[parent before fork] PFN={parent_phys_before.get('pfn')}, "
          f"phys={parent_phys_before.get('phys_addr')}")

    pid = os.fork()
    if pid == 0:
        child_before = virt_to_phys(vaddr)
        print(f"[child before write] PFN={child_before.get('pfn')} "
              f"(should match parent — shared CoW page)")
        m[0] = 0xBB  # Trigger CoW split
        child_after = virt_to_phys(vaddr)
        print(f"[child after write]  PFN={child_after.get('pfn')} "
              f"(should differ from parent — new physical page)")
        m.close()
        os._exit(0)
    else:
        os.waitpid(pid, 0)
        parent_phys_after = virt_to_phys(vaddr)
        print(f"[parent after fork]  PFN={parent_phys_after.get('pfn')} "
              f"(unchanged — parent keeps original page)")
        m.close()

import ctypes
demonstrate_cow_physical_sharing()
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

### Measuring TLB Misses with Hardware Performance Counters

The Linux `perf` tool exposes CPU performance counters that directly count TLB misses. These are architectural events wired to the PMU (Performance Monitoring Unit) inside the CPU.

```bash
# Count dTLB load misses for a command (requires perf + kernel.perf_event_paranoid <= 1)
perf stat -e dTLB-load-misses,dTLB-loads,iTLB-load-misses,iTLB-loads \
    python3 -c "
import mmap, os
m = mmap.mmap(-1, 256*1024*1024)
for i in range(0, 256*1024*1024, 4096):
    m[i] = 1
m.close()
"

# Expected output for 4KB page run (TLB thrashing):
#   dTLB-load-misses:  ~6,500,000   (one per page — TLB can't cover 256MB with 4KB pages)
#   dTLB-loads:        ~650,000,000

# With 2MB huge pages (madvise MADV_HUGEPAGE), dTLB-load-misses drops 512x
perf stat -e dTLB-load-misses python3 -c "
import mmap, ctypes
m = mmap.mmap(-1, 256*1024*1024)
libc = ctypes.CDLL('libc.so.6')
# MADV_HUGEPAGE = 14
ptr = ctypes.addressof(ctypes.c_char.from_buffer(m))
libc.madvise(ptr, 256*1024*1024, 14)
for i in range(0, 256*1024*1024, 4096):
    m[i] = 1
m.close()
"
# Expected: dTLB-load-misses ~12,500 (512 pages per huge page entry, so 512x fewer misses)
```

```python
import subprocess
import os
import mmap

def measure_tlb_misses_via_perf(size_mb: int = 128, use_huge: bool = False) -> dict:
    """
    Spawn a subprocess that touches pages and counts TLB misses via perf.
    Requires: perf installed, /proc/sys/kernel/perf_event_paranoid <= 1.
    """
    script = f"""
import mmap, ctypes, os
SIZE = {size_mb} * 1024 * 1024
m = mmap.mmap(-1, SIZE)
{"# Enable huge pages via madvise" if use_huge else "# Using 4KB pages"}
{"libc = ctypes.CDLL('libc.so.6'); ptr = ctypes.addressof(ctypes.c_char.from_buffer(m)); libc.madvise(ptr, SIZE, 14)" if use_huge else ""}
for i in range(0, SIZE, 4096):
    m[i] = 1
m.close()
"""
    cmd = [
        "perf", "stat", "-e", "dTLB-load-misses,dTLB-loads",
        "--", "python3", "-c", script
    ]
    try:
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=60)
        output = result.stderr  # perf writes to stderr
        misses = loads = None
        for line in output.splitlines():
            if "dTLB-load-misses" in line:
                misses = line.split()[0].replace(",", "")
            elif "dTLB-loads" in line and "misses" not in line:
                loads = line.split()[0].replace(",", "")
        return {
            "size_mb": size_mb,
            "huge_pages": use_huge,
            "dTLB_load_misses": int(misses) if misses else None,
            "dTLB_loads": int(loads) if loads else None,
            "miss_rate_pct": f"{int(misses)/int(loads)*100:.2f}%" if misses and loads else None,
        }
    except (FileNotFoundError, subprocess.TimeoutExpired) as e:
        return {"error": str(e), "note": "Install perf: apt install linux-tools-generic"}

# Run comparison: 4KB vs huge pages
r4k = measure_tlb_misses_via_perf(128, use_huge=False)
rhuge = measure_tlb_misses_via_perf(128, use_huge=True)
print("4KB pages:  ", r4k)
print("Huge pages: ", rhuge)
# Typical result: 4KB miss rate ~0.8%, huge page miss rate ~0.002% (400x improvement)
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

### Explicit Huge Pages: `MAP_HUGETLB` and the Hugetlbfs

For latency-critical applications (HFT, databases, ML inference), waiting for `khugepaged` to promote pages is unacceptable. Use `MAP_HUGETLB` to request huge pages directly from the kernel's **huge page pool**:

```python
import mmap
import ctypes
import os

# MAP_HUGETLB = 0x40000 on Linux x86-64
# MAP_HUGE_2MB = (21 << MAP_HUGE_SHIFT) where MAP_HUGE_SHIFT = 26
MAP_HUGETLB  = 0x40000
MAP_HUGE_2MB = (21 << 26)
MAP_HUGE_1GB = (30 << 26)
MAP_ANONYMOUS = 0x20
MAP_PRIVATE   = 0x02

# Pre-requisite: echo 512 > /proc/sys/vm/nr_hugepages
# This reserves 512 × 2MB = 1GB of huge pages at boot time

def allocate_huge_page_mmap(size_mb: int = 2) -> mmap.mmap | None:
    """
    Allocate using MAP_HUGETLB. Fails with ENOMEM if huge page pool is empty.
    Unlike THP, this never causes compaction stalls — pages are pre-reserved.
    """
    size = size_mb * 1024 * 1024
    # Ensure size is a multiple of 2MB
    size = (size + (2 * 1024 * 1024 - 1)) & ~(2 * 1024 * 1024 - 1)
    
    libc = ctypes.CDLL("libc.so.6", use_errno=True)
    # void* mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset)
    ptr = libc.mmap(None, size, 3,  # PROT_READ|PROT_WRITE=3
                    MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB | MAP_HUGE_2MB,
                    -1, 0)
    
    MAP_FAILED = ctypes.c_void_p(-1).value
    if ctypes.c_ulong(ptr).value == ctypes.c_ulong(MAP_FAILED).value:
        errno = ctypes.get_errno()
        if errno == 12:  # ENOMEM
            print("MAP_HUGETLB failed: ENOMEM — run: echo 64 > /proc/sys/vm/nr_hugepages")
        else:
            print(f"MAP_HUGETLB failed: errno={errno}")
        return None
    
    print(f"Huge page mmap succeeded at {hex(ptr)}, size={size // (1024*1024)}MB")
    # First touch to populate
    buf = (ctypes.c_char * size).from_address(ptr)
    buf[0] = b'\x01'
    libc.munmap(ptr, size)
    return None  # demonstrating allocation only

allocate_huge_page_mmap(2)

def check_hugepage_pool() -> dict:
    """Read huge page pool status from /proc/meminfo."""
    stats = {}
    with open("/proc/meminfo") as f:
        for line in f:
            for key in ("HugePages_Total", "HugePages_Free", "HugePages_Rsvd",
                        "Hugepagesize", "Hugetlb"):
                if line.startswith(key + ":"):
                    val = line.split()[1]
                    stats[key] = int(val)
    if stats:
        print(f"Huge page pool: {stats.get('HugePages_Free',0)}/{stats.get('HugePages_Total',0)} "
              f"free ({stats.get('Hugepagesize',0)} kB each)")
    return stats

check_hugepage_pool()
```

### `madvise` Reference: Hints That Change Kernel Behavior

```python
import mmap
import ctypes

# madvise(2) constants and their kernel-side effects
MADVISE_HINTS = {
    # Performance hints
    "MADV_NORMAL":     (0,  "Default: mild readahead"),
    "MADV_RANDOM":     (1,  "Disable readahead — random access pattern"),
    "MADV_SEQUENTIAL": (2,  "Aggressive readahead — sequential access"),
    "MADV_WILLNEED":   (3,  "Prefetch pages now — fadvise equivalent for mmap"),
    # Memory reclaim
    "MADV_DONTNEED":   (4,  "Drop pages from RSS (stays mapped, re-faulted on access)"),
    "MADV_FREE":       (8,  "Lazy free: pages dropped under pressure, reused if not touched"),
    # Huge page control
    "MADV_HUGEPAGE":   (14, "Enable THP for this region (overrides system setting)"),
    "MADV_NOHUGEPAGE": (15, "Disable THP for this region"),
    # Memory safety / hardening
    "MADV_DONTDUMP":   (16, "Exclude from core dumps (for sensitive data)"),
    "MADV_DOFORK":     (11, "Inherit mapping across fork"),
    "MADV_DONTFORK":   (10, "Exclude from child after fork (for DMA-pinned memory)"),
    # Linux 5.14+
    "MADV_COLLAPSE":   (25, "Synchronously collapse to THP (no khugepaged wait)"),
}

def apply_madvise(m: mmap.mmap, size: int, advice_name: str) -> bool:
    """Apply madvise hint to an mmap region."""
    if advice_name not in MADVISE_HINTS:
        return False
    advice_val, desc = MADVISE_HINTS[advice_name]
    libc = ctypes.CDLL("libc.so.6", use_errno=True)
    ptr = ctypes.addressof(ctypes.c_char.from_buffer(m))
    ret = libc.madvise(ctypes.c_void_p(ptr), size, advice_val)
    if ret != 0:
        print(f"madvise({advice_name}={advice_val}) failed: errno={ctypes.get_errno()}")
        return False
    print(f"madvise({advice_name}): {desc}")
    return True

# Demonstrate: allocate 100MB, first touch, then MADV_FREE (reclaim under pressure)
m = mmap.mmap(-1, 100 * 1024 * 1024)
for i in range(0, 100 * 1024 * 1024, 4096):
    m[i] = 0x42
apply_madvise(m, 100 * 1024 * 1024, "MADV_FREE")
# After MADV_FREE: pages are still accessible but kernel may reclaim them
# under memory pressure. Next access re-faults them demand-zero.
m.close()
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

### `struct vm_area_struct` — The Per-Mapping Descriptor

Each VMA is described by a `struct vm_area_struct` (abbreviated `vma`), defined in `include/linux/mm_types.h`:

```c
// Simplified for readability
struct vm_area_struct {
    unsigned long vm_start;    // Start virtual address (inclusive)
    unsigned long vm_end;      // End virtual address (exclusive)
    struct vm_area_struct *vm_next, *vm_prev;  // Linked list (pre-6.1)
    pgprot_t      vm_page_prot;  // Page protection (maps to PTE permission bits)
    unsigned long vm_flags;      // VM_READ, VM_WRITE, VM_EXEC, VM_SHARED, VM_GROWSDOWN...
    struct mm_struct *vm_mm;     // Back-pointer to owning mm_struct
    
    // For file-backed mappings:
    struct file *vm_file;         // File being mapped (NULL for anonymous)
    unsigned long vm_pgoff;       // Offset within file (in pages)
    
    // VMA operations (fault handler, open, close, etc.)
    const struct vm_operations_struct *vm_ops;
};
```

Key `vm_flags` bits:
```
VM_READ       = 0x00000001  // Readable
VM_WRITE      = 0x00000002  // Writable
VM_EXEC       = 0x00000004  // Executable
VM_SHARED     = 0x00000008  // MAP_SHARED — changes visible across processes
VM_MAYWRITE   = 0x00000020  // Allowed to set VM_WRITE (for mprotect)
VM_GROWSDOWN  = 0x00000100  // Stack: extend downward on page fault
VM_PFNMAP     = 0x00000400  // Special: maps physical addresses directly (device mmaps)
VM_LOCKED     = 0x00002000  // Pages are mlock'd — never swap
VM_HUGETLB    = 0x00400000  // Uses huge pages
VM_WIPEONFORK = 0x02000000  // Zero-fill on fork (for sensitive keys, Linux 4.14+)
```

Reading VMAs from Python (equivalent to `/proc/self/maps`):

```python
import os
import re

def parse_proc_maps(pid: int = None) -> list[dict]:
    """
    Parse /proc/<pid>/maps — the human-readable VMA list.
    Format per line: start-end perms offset dev inode [pathname]
    """
    pid_str = str(pid) if pid else "self"
    vmas = []
    pattern = re.compile(
        r"([0-9a-f]+)-([0-9a-f]+)\s+([rwxsp-]{4})\s+([0-9a-f]+)\s+"
        r"([0-9a-f:]+)\s+(\d+)\s*(.*)"
    )
    try:
        with open(f"/proc/{pid_str}/maps") as f:
            for line in f:
                m = pattern.match(line.strip())
                if m:
                    start, end, perms, offset, dev, inode, name = m.groups()
                    size_kb = (int(end, 16) - int(start, 16)) // 1024
                    vmas.append({
                        "start": int(start, 16),
                        "end":   int(end, 16),
                        "size_kb": size_kb,
                        "perms": perms,
                        "offset": int(offset, 16),
                        "name": name.strip() or "[anon]",
                    })
    except PermissionError:
        print(f"Cannot read /proc/{pid_str}/maps")
    return vmas

def print_vma_summary(pid: int = None):
    """Print VMA layout — useful for understanding Python's address space."""
    vmas = parse_proc_maps(pid)
    print(f"\n{'Start':>18} {'End':>18} {'Size':>8} {'Perms'} {'Name'}")
    print("-" * 80)
    total_kb = 0
    for v in vmas:
        print(f"  {v['start']:#018x}  {v['end']:#018x}  "
              f"{v['size_kb']:>6}KB  {v['perms']}  {v['name'][:40]}")
        total_kb += v["size_kb"]
    print(f"\nTotal VMAs: {len(vmas)}, Total virtual: {total_kb // 1024}MB")

print_vma_summary()
```

### Linux 6.1+ Maple Tree: Why the VMA Data Structure Changed

Prior to Linux 6.1, VMAs were stored in two redundant structures:
1. A **red-black tree** (`mm_struct.mm_rb`) for O(log n) lookup by address.
2. A **doubly-linked list** (`vm_area_struct.vm_next/prev`) for O(1) traversal.

Maintaining two structures for every `mmap`/`munmap`/`mprotect` operation caused lock contention under high concurrency. Linux 6.1 replaced both with a single **Maple Tree** — a B-tree variant optimized for range operations, RCU-safe reads, and cache efficiency. The maple tree stores ranges (VMA start–end pairs) as keys, enabling O(log n) lookup, O(log n) insertion, and lockless concurrent reads via RCU.

### `struct page` — The Kernel's Per-Page Metadata

Every physical page frame has a `struct page` (in `include/linux/mm_types.h`). On a machine with 64GB RAM and 4KB pages: **16 million** `struct page` instances. The kernel must keep `struct page` small — modern Linux uses a union to overlap fields for different page types, keeping it at **64 bytes**.

```c
// Heavily simplified struct page (linux/mm_types.h)
struct page {
    unsigned long flags;        // PG_locked, PG_dirty, PG_referenced, PG_active, etc.
    union {
        // For page cache pages (file-backed):
        struct {
            struct address_space *mapping;  // Points to inode's page cache
            pgoff_t index;                  // Offset within file (in pages)
        };
        // For anonymous pages (heap/stack):
        struct {
            struct anon_vma *anon_vma;  // For reverse mapping (finding all PTEs)
        };
        // For slab allocator pages:
        struct {
            struct kmem_cache *slab_cache;
            void *freelist;
        };
    };
    _refcount;  // atomic_t — number of references (page table entries + kernel refs)
    _mapcount;  // atomic_t — number of PTEs pointing here (-1 = not mapped)
    // ... union for compound pages (huge pages), buddy allocator pointers, etc.
};
```

**Key `page->flags` bits:**

| Flag | Value | Meaning |
|---|---|---|
| `PG_locked` | bit 0 | Page is locked for I/O — other accessors sleep |
| `PG_referenced` | bit 2 | Recently accessed — used by LRU aging |
| `PG_dirty` | bit 4 | Modified since last writeback to disk |
| `PG_lru` | bit 5 | On the LRU list (active or inactive) |
| `PG_active` | bit 6 | On the active LRU list |
| `PG_swapcache` | bit 8 | Has a swap entry (being evicted) |
| `PG_mappedtodisk` | bit 9 | Blocks allocated on disk |
| `PG_reclaim` | bit 10 | Being reclaimed by kswapd |
| `PG_compound` | bit 14 | Part of a compound (huge) page |
| `PG_hwpoison` | bit 31 | Hardware memory error — kernel will kill accessor |

The 16 million `struct page` instances take **~1GB of RAM** on a 64GB machine — purely for metadata. This is called the `mem_map` and is allocated at boot.

### The Buddy Allocator: Physical Page Allocation

When the kernel needs to allocate N contiguous physical pages, it uses the **buddy allocator** (`mm/page_alloc.c`). Pages are organized into free lists by order (power of 2):

```
Order 0: 1 page  (4KB)    — free list of individual pages
Order 1: 2 pages (8KB)    — free list of 2-page blocks
Order 2: 4 pages (16KB)
...
Order 9: 512 pages (2MB)  — one huge page
Order 10: 1024 pages (4MB)
```

Allocation of a 2MB huge page: request from order-9 free list. If order-9 is empty, split an order-10 block into two order-9 blocks, use one, put other on order-9 list.

**External fragmentation:** After hours of mixed small and large allocations, the order-9 free list may be empty even if 2MB of total free RAM exists — it's just not contiguous. This is why THP promotion can fail and trigger compaction (`kcompactd`).

```python
def read_buddy_allocator_stats() -> dict:
    """Read buddy allocator free list counts from /proc/buddyinfo."""
    buddy_path = "/proc/buddyinfo"
    stats = {}
    try:
        with open(buddy_path) as f:
            for line in f:
                # Format: Node N, zone <name>, <count per order>
                parts = line.split(",")
                if len(parts) < 2:
                    continue
                zone_part = parts[1].strip()
                counts_part = parts[2].strip() if len(parts) > 2 else ""
                counts = [int(x) for x in counts_part.split() if x.isdigit()]
                node_zone = f"{parts[0].strip()},{zone_part}"
                stats[node_zone] = {
                    f"order_{i}": count
                    for i, count in enumerate(counts)
                }
        
        print("Buddy allocator free pages per order:")
        for zone, orders in stats.items():
            print(f"  {zone}:")
            for order, count in orders.items():
                size_kb = (4 * (2 ** int(order.split('_')[1])))
                print(f"    {order} ({size_kb:>6}KB blocks): {count:>8} free "
                      f"({count * size_kb // 1024:>6}MB)")
    except FileNotFoundError:
        print("buddyinfo not available")
    return stats

read_buddy_allocator_stats()
```

### Stack Growth: `VM_GROWSDOWN`

The stack VMA has the flag `VM_GROWSDOWN`. When the CPU generates a page fault at an address just below the current stack bottom:

1. Kernel checks: is the faulting address within `STACK_GUARD_GAP` (1MB) below the stack VMA?
2. If yes: extend the VMA downward by one page, allocate the page, return.
3. If the stack would exceed `ulimit -s` (default 8MB): SIGSEGV — stack overflow.

This is why the stack "grows" without the programmer doing anything — the OS extends the VMA on each page fault.

### ASLR (Address Space Layout Randomization)

ASLR randomizes the base addresses of key memory regions at process startup to make exploit address prediction infeasible:

| Region | Without ASLR | With ASLR (typical) |
|---|---|---|
| Stack base | `0x7fffffffe000` (fixed) | Random 28-bit offset |
| mmap region | Fixed base | Random 28-bit offset |
| Heap (brk) | Right after BSS | Random 13-bit offset |
| Executable (PIE) | `0x400000` (fixed) | Random 28-bit offset |
| VDSO/VVAR | Fixed | Randomized with mmap |

```
/proc/sys/kernel/randomize_va_space:
  0: ASLR disabled (for debugging or deterministic fuzzing)
  1: Randomize stack, mmap, VDSO (not heap)
  2: Full ASLR — randomize everything including heap (default)
```

```python
import os
import re

def measure_aslr_entropy() -> dict:
    """
    Measure ASLR randomness by reading stack and mmap addresses from /proc/self/maps.
    Run multiple times to see how much the addresses vary.
    """
    maps_path = "/proc/self/maps"
    addresses = {}
    
    with open(maps_path) as f:
        for line in f:
            parts = line.split()
            if not parts:
                continue
            addr_range = parts[0]
            name = parts[-1] if len(parts) > 5 else ""
            start = int(addr_range.split("-")[0], 16)
            
            if "[stack]" in name:
                addresses["stack"] = start
            elif "[heap]" in name:
                addresses["heap"] = start
            elif "[vdso]" in name:
                addresses["vdso"] = start
            elif name.endswith("libc.so.6") and "r-xp" in line:
                addresses["libc_text"] = start
    
    aslr_status_path = "/proc/sys/kernel/randomize_va_space"
    with open(aslr_status_path) as f:
        aslr_level = int(f.read().strip())
    
    print(f"ASLR level: {aslr_level} ({'disabled' if aslr_level==0 else 'partial' if aslr_level==1 else 'full'})")
    for region, addr in addresses.items():
        print(f"  {region:15s}: {addr:#018x}")
    print("  (Run again to see different addresses — that's ASLR working)")
    return addresses

measure_aslr_entropy()
```

### W^X (Write XOR Execute) Policy

A fundamental security property: **no page should be simultaneously writable and executable**. If a page is writable (attacker can inject shellcode) and executable (CPU will run it), a buffer overflow can execute arbitrary code.

The Linux kernel enforces W^X through PTE bits:
- `VM_WRITE | VM_EXEC` combination in a VMA is rejected by security-hardened kernels.
- `mprotect(PROT_READ | PROT_WRITE | PROT_EXEC)` is allowed by default but blocked by `CONFIG_STRICT_W_EXEC` and security modules (SELinux, grsecurity).

JIT compilers (Python's Numba, PyPy, Java HotSpot) legitimately need W^X:
1. Allocate a writable page: `mmap(PROT_READ|PROT_WRITE)`.
2. Write compiled machine code into it.
3. `mprotect(PROT_READ|PROT_EXEC)` — flip to executable, remove write permission.
4. Execute the JIT code via a function pointer.

```python
import ctypes
import mmap
import struct

def execute_jit_shellcode():
    """
    W^X demonstration: write x86-64 machine code, flip to executable, call it.
    The function computes 1 + 2 and returns 3.
    
    x86-64 machine code for: int add(int a, int b) { return a + b; }
    System V AMD64 ABI: args in rdi, rsi. Return in rax.
    """
    # mov eax, edi   (eax = first arg)
    # add eax, esi   (eax += second arg)
    # ret
    machine_code = bytes([
        0x89, 0xF8,  # mov eax, edi
        0x01, 0xF0,  # add eax, esi
        0xC3,        # ret
    ])
    
    # Step 1: Allocate writable (not executable) page
    code_page = mmap.mmap(-1, 4096,
                          flags=mmap.MAP_PRIVATE | mmap.MAP_ANONYMOUS,
                          prot=mmap.PROT_READ | mmap.PROT_WRITE)
    
    # Step 2: Write machine code
    code_page[:len(machine_code)] = machine_code
    
    # Step 3: Flip to executable (remove write permission — W^X)
    libc = ctypes.CDLL("libc.so.6", use_errno=True)
    ptr = ctypes.addressof(ctypes.c_char.from_buffer(code_page))
    PROT_READ_EXEC = 0x1 | 0x4
    ret = libc.mprotect(ctypes.c_void_p(ptr), 4096, PROT_READ_EXEC)
    if ret != 0:
        print(f"mprotect failed: errno={ctypes.get_errno()}")
        code_page.close()
        return
    
    # Step 4: Cast to callable function pointer and call
    FuncType = ctypes.CFUNCTYPE(ctypes.c_int, ctypes.c_int, ctypes.c_int)
    jit_func = FuncType(ptr)
    result = jit_func(40, 2)
    print(f"JIT function(40, 2) = {result}  (expected 42)")
    
    code_page.close()

execute_jit_shellcode()
```

### The SLUB Allocator: Kernel's `malloc`

When the kernel allocates small objects (inodes, VMAs, `struct page` metadata, network buffers), it uses the **SLUB allocator** (`mm/slub.c`) — the default since Linux 2.6.23. SLUB organizes objects into fixed-size **caches** (slabs), amortizing page allocation overhead:

```
struct kmem_cache "vm_area_struct" (192 bytes each):
┌──────────────────────────────────────────────────────┐
│  Slab page 1 (4KB):   [VMA_0][VMA_1][VMA_2]...[VMA_21]  │
│  Slab page 2 (4KB):   [VMA_22][VMA_23]...[VMA_43]        │
│  freelist → VMA_5 → VMA_17 → VMA_31 → NULL               │
└──────────────────────────────────────────────────────┘
```

- `kmalloc(size)` → picks the cache for size ≤ 8, 16, 32, 64 ... 8192 bytes.
- Per-CPU partial slabs avoid lock contention on multi-core.
- SLUB stores free-list pointers directly in free object space (no separate header).

**SLUB statistics** (visible from Python):

```python
def read_slub_stats(cache_name: str = "vm_area_struct") -> dict:
    """
    Read SLUB allocator statistics for a specific cache from /proc/slabinfo.
    Format: name, active_objs, num_objs, obj_size, obj_per_slab, pages_per_slab
    """
    slabinfo_path = "/proc/slabinfo"
    result = {}
    try:
        with open(slabinfo_path) as f:
            header = f.readline()  # skip version line
            f.readline()  # skip column header
            for line in f:
                parts = line.split()
                if not parts or parts[0] != cache_name:
                    continue
                name = parts[0]
                active_objs = int(parts[1])
                num_objs = int(parts[2])
                obj_size = int(parts[3])
                objperslab = int(parts[4])
                pages_per_slab = int(parts[5])
                result = {
                    "name": name,
                    "active_objs": active_objs,
                    "num_objs": num_objs,
                    "obj_size_bytes": obj_size,
                    "objs_per_slab": objperslab,
                    "pages_per_slab": pages_per_slab,
                    "total_kb": num_objs * obj_size // 1024,
                    "fragmentation_pct": (1 - active_objs / max(num_objs, 1)) * 100,
                }
                break
    except (FileNotFoundError, PermissionError):
        return {"error": "Cannot read /proc/slabinfo (need root or CONFIG_SLUB_STATS)"}
    
    if result:
        print(f"SLUB cache '{result['name']}':")
        print(f"  Objects: {result['active_objs']}/{result['num_objs']} active, "
              f"{result['obj_size_bytes']}B each, {result['total_kb']}KB total")
        print(f"  Fragmentation: {result['fragmentation_pct']:.1f}%")
    return result

# Interesting caches to inspect:
for cache in ["vm_area_struct", "mm_struct", "task_struct", "filp", "dentry", "inode_cache"]:
    r = read_slub_stats(cache)
    if r and "error" not in r:
        print()

def read_top_slabs_by_memory() -> list[dict]:
    """Find the top SLUB caches by total memory consumed."""
    slabinfo_path = "/proc/slabinfo"
    caches = []
    try:
        with open(slabinfo_path) as f:
            f.readline(); f.readline()
            for line in f:
                parts = line.split()
                if len(parts) < 6:
                    continue
                try:
                    num_objs = int(parts[2])
                    obj_size = int(parts[3])
                    total_kb = num_objs * obj_size // 1024
                    caches.append({"name": parts[0], "total_kb": total_kb,
                                   "num_objs": num_objs, "obj_size": obj_size})
                except ValueError:
                    pass
        caches.sort(key=lambda x: x["total_kb"], reverse=True)
        print(f"\n{'Cache Name':>30} {'Objects':>10} {'Size':>6}B {'Total':>8}KB")
        for c in caches[:10]:
            print(f"  {c['name']:>30} {c['num_objs']:>10} {c['obj_size']:>6} {c['total_kb']:>8}")
    except (FileNotFoundError, PermissionError):
        print("Cannot read /proc/slabinfo")
    return caches[:10]

read_top_slabs_by_memory()
```

### Swap Internals: How Pages Leave RAM

When physical RAM is exhausted, the kernel's **page reclaim** subsystem (`mm/vmscan.c`) runs. It selects **victim pages** using a two-list LRU:

1. **Active list:** Pages recently accessed (Accessed bit set in PTE).
2. **Inactive list:** Pages not recently accessed — candidates for reclaim.

Pages migrate: Active → Inactive (on second LRU scan) → Reclaimed.

**For anonymous pages** (heap, stack — not file-backed): they are written to **swap space** (a raw partition or swapfile). The PTE is updated:
```
Bits [63:0]: swap_type (5 bits) | swap_offset (50 bits)
Bit 0 (Present): 0  ← triggers page fault on next access
```

On next access, the page fault handler reads the swap entry, issues an I/O to read the page back, and updates the PTE. This is a **major fault** (~5–15ms).

**For file-backed pages** (mmap of a file, shared libraries): they are simply evicted (the file is their backing store). Re-reading is also a major fault, but reads from the file rather than swap.

### `zswap` and `zram`: Compressed In-Memory Swap

Modern systems often use **compressed swap** to avoid writing to disk entirely:

- **`zram`** (Linux 3.14+): A RAM-based block device that compresses pages using LZ4/zstd before storing them. Effectively extends RAM capacity 2–3× at the cost of CPU compression cycles. Used by Android, ChromeOS, and cloud VMs by default.
- **`zswap`** (Linux 3.11+): A compressed page pool that intercepts swap-outs and stores compressed pages in RAM. If the pool fills, pages are evicted to the real swap device. Acts as a swap cache, reducing I/O.

```python
def check_zswap_status() -> dict:
    """Read zswap statistics from debugfs."""
    stats = {}
    zswap_path = "/sys/kernel/debug/zswap"
    if not os.path.isdir(zswap_path):
        return {"zswap_available": False, "note": "Requires debugfs + CONFIG_ZSWAP"}
    for fname in os.listdir(zswap_path):
        try:
            with open(f"{zswap_path}/{fname}") as f:
                val = f.read().strip()
                stats[fname] = int(val) if val.isdigit() else val
        except:
            pass
    if "pool_pages" in stats and "stored_pages" in stats:
        ratio = stats.get("stored_pages", 0) / max(stats.get("pool_pages", 1), 1)
        stats["compression_ratio"] = f"{ratio:.2f}x"
    return stats

def check_swap_usage() -> dict:
    """Read swap usage from /proc/meminfo."""
    result = {}
    with open("/proc/meminfo") as f:
        for line in f:
            for key in ("SwapTotal", "SwapFree", "SwapCached"):
                if line.startswith(key + ":"):
                    result[key] = int(line.split()[1])
    used = result.get("SwapTotal", 0) - result.get("SwapFree", 0)
    result["SwapUsed"] = used
    print(f"Swap: {used//1024}MB used / {result.get('SwapTotal',0)//1024}MB total, "
          f"Cached={result.get('SwapCached',0)//1024}MB")
    return result

check_swap_usage()
print("zswap:", check_zswap_status())
```

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

### Spectre, Meltdown, and Memory Isolation

The 2018 hardware vulnerabilities exposed deep interactions between virtual memory, speculative execution, and microarchitectural state:

**Meltdown (CVE-2017-5754) — "rogue data cache load":**
The CPU's out-of-order execution engine speculatively reads kernel memory before the page permission check completes. The data never commits to registers, but it leaves a trace in the CPU cache. An attacker measures cache timing (`RDTSC` + memory probe) to reconstruct the secret. Linux's fix: **KPTI** — keep kernel pages unmapped in user-space page tables entirely.

**Spectre Variant 1 (CVE-2017-5753) — bounds check bypass:**
A speculative execution path reads past an array bound while the bounds check is "in flight." The out-of-bounds data trains the branch predictor and leaks via cache timing. Fix: `array_index_nospec()` barrier in kernel; compilers insert `LFENCE` at speculation barriers.

**Spectre Variant 2 (CVE-2017-5715) — branch target injection:**
The attacker poisons the Branch Target Buffer (BTB) to redirect the victim's speculative execution to an attacker-chosen gadget. Fix: **Retpoline** — a return-based trampoline that avoids indirect branches:

```asm
; Retpoline for indirect call through register rax:
; Instead of: call *%rax   (speculates to attacker-controlled target)
; Use:
    call set_up_target
    .Lspec_trap:
        pause           ; serialize speculation
        lfence
        jmp .Lspec_trap  ; infinite loop that the speculator hits
    set_up_target:
        mov %rax, (%rsp) ; overwrite return address with true target
        ret              ; speculator: predicts return to .Lspec_trap (safe)
                         ; actual CPU:  returns to *rax (correct target)
```

The CPU speculates into the `pause/lfence` loop (harmless), while actually executing the correct target.

**eIBRS (Enhanced IBRS):** Hardware fix in Tiger Lake+. The hardware tracks privilege levels in the BTB — user-space cannot poison kernel-mode branch predictors. Linux uses eIBRS instead of software retpoline on supporting CPUs.

```python
def check_spectre_mitigations() -> dict:
    """Read CPU vulnerability mitigations from sysfs (Linux 4.14.2+)."""
    vuln_path = "/sys/devices/system/cpu/vulnerabilities"
    mitigations = {}
    if not os.path.isdir(vuln_path):
        return {"error": "Vulnerability sysfs not available"}
    for vuln in sorted(os.listdir(vuln_path)):
        try:
            with open(f"{vuln_path}/{vuln}") as f:
                status = f.read().strip()
            mitigations[vuln] = status
        except PermissionError:
            mitigations[vuln] = "<permission denied>"
    
    print("\nCPU Vulnerability Status:")
    for vuln, status in mitigations.items():
        indicator = "✓" if "Mitigation" in status or "Not affected" in status else "✗"
        print(f"  {indicator} {vuln:30s}: {status}")
    return mitigations

check_spectre_mitigations()
```

**Performance impact of mitigations on Python:**

| Mitigation | Overhead | When it fires |
|---|---|---|
| KPTI (Meltdown) | 5–30% for syscall-heavy code | Every syscall/interrupt crossing |
| Retpoline | 1–5% | Every indirect branch (function ptr, vtable) |
| IBRS/eIBRS | 2–8% | Kernel indirect branches |
| MDS/L1TF clears | 1–3% | Context switches, hyperthreading |
| `SWAPGS` barrier | <1% | Syscall entry path |

Python is particularly sensitive to KPTI because CPython makes many syscalls (memory allocation via `brk`/`mmap`, file operations, timers). A Python web server may see 10–20% throughput reduction on pre-Ice Lake CPUs.

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

### Live Memory Inspection: Reading Another Process's Memory

The kernel exposes `/proc/<pid>/mem` — a virtual file that, with the right permissions, lets one process read/write another's memory as if doing `ptrace`. This is how `gdb`, `strace`, and memory sanitizers work.

```python
import os
import struct
import ctypes

def read_process_memory(pid: int, vaddr: int, size: int) -> bytes | None:
    """
    Read 'size' bytes from virtual address 'vaddr' in process 'pid'.
    Requires: same UID, or CAP_SYS_PTRACE, or ptrace attachment.
    
    This is exactly how gdb's 'x/Nx' command works internally.
    """
    mem_path = f"/proc/{pid}/mem"
    try:
        with open(mem_path, "rb") as f:
            f.seek(vaddr)
            data = f.read(size)
            return data
    except (PermissionError, OSError) as e:
        print(f"Cannot read /proc/{pid}/mem at {hex(vaddr)}: {e}")
        return None

def introspect_python_object(obj) -> dict:
    """
    Read a Python object's raw memory layout — specifically ob_refcnt and ob_type.
    CPython object header: [ob_refcnt (8 bytes)] [ob_type* (8 bytes)] [ob_val...]
    """
    addr = id(obj)  # id() returns the object's virtual address in CPython
    
    # Read 24 bytes: ob_refcnt (8) + ob_type* (8) + first 8 bytes of object body
    data = read_process_memory(os.getpid(), addr, 24)
    if data is None or len(data) < 16:
        return {}
    
    ob_refcnt = struct.unpack_from("<q", data, 0)[0]   # signed 64-bit
    ob_type_ptr = struct.unpack_from("<Q", data, 8)[0]  # unsigned 64-bit pointer
    
    result = {
        "addr": hex(addr),
        "ob_refcnt": ob_refcnt,
        "ob_type_ptr": hex(ob_type_ptr),
        "type_name": type(obj).__name__,
        "sys_getrefcount": __import__("sys").getrefcount(obj),
    }
    print(f"PyObject at {result['addr']}:")
    print(f"  ob_refcnt     = {ob_refcnt}  (sys.getrefcount={result['sys_getrefcount']})")
    print(f"  ob_type*      = {result['ob_type_ptr']} ({result['type_name']})")
    print(f"  Note: sys.getrefcount is 1 higher (counts the argument reference)")
    return result

# Inspect a Python integer object
x = 12345
introspect_python_object(x)

# Demonstrate CoW by checking ob_refcnt mutation on dict access:
demo_dict = {"key": "value" * 1000}
print("\nBefore access:")
introspect_python_object(demo_dict["key"])
_ = demo_dict["key"]  # This Py_INCREFs "key" temporarily
print("\nThe ob_refcnt mutation is WHY Python dicts CoW-copy pages even on read access")
```

### Cache Coherence: The Hardware Contract Under Virtual Memory

Modern multi-core CPUs cache data in L1/L2/L3 caches. When two cores share a physical page (after fork, or via `MAP_SHARED`), a write by Core 0 must become visible to Core 1. The hardware guarantees this via the **MESI cache coherence protocol**:

| State | Meaning |
|---|---|
| **M** (Modified) | Core 0 has the cache line, it's dirty — other caches have invalid copies |
| **E** (Exclusive) | Core 0 has the only copy, unmodified — can silently promote to M |
| **S** (Shared) | Multiple cores have valid read-only copies |
| **I** (Invalid) | Cache line is stale — must fetch from memory or another cache |

**False sharing** — the CoW of caches: Two threads write to different variables that happen to be on the same 64-byte cache line. Every write by Thread A invalidates Thread B's cache line copy, forcing a re-fetch — even though they're writing completely different data.

```python
import ctypes
import threading
import time

def demonstrate_false_sharing() -> None:
    """
    False sharing: two threads update adjacent counters in the same cache line.
    Counter A is at offset 0, Counter B at offset 8 — same 64-byte cache line.
    Both threads fight over the cache line, causing massive cache coherence traffic.
    """
    ITERATIONS = 10_000_000
    
    # Layout 1: Counters packed (false sharing — same cache line)
    class PackedCounters(ctypes.Structure):
        _fields_ = [("a", ctypes.c_int64), ("b", ctypes.c_int64)]
    
    # Layout 2: Counters padded to separate cache lines (no false sharing)
    class PaddedCounters(ctypes.Structure):
        _fields_ = [
            ("a",     ctypes.c_int64),
            ("pad_a", ctypes.c_int8 * 56),  # pad to 64 bytes
            ("b",     ctypes.c_int64),
            ("pad_b", ctypes.c_int8 * 56),
        ]
    
    def bench(CounterType: type, label: str) -> float:
        counters = CounterType()
        done = [False]
        
        def thread_a():
            for _ in range(ITERATIONS):
                counters.a += 1
        
        def thread_b():
            for _ in range(ITERATIONS):
                counters.b += 1
        
        t0 = time.perf_counter()
        ta = threading.Thread(target=thread_a)
        tb = threading.Thread(target=thread_b)
        ta.start(); tb.start()
        ta.join(); tb.join()
        elapsed = time.perf_counter() - t0
        print(f"  {label}: {elapsed*1000:.0f}ms  a={counters.a} b={counters.b}")
        return elapsed
    
    print(f"False sharing benchmark ({ITERATIONS:,} increments per thread):")
    t_packed = bench(PackedCounters, "Packed (false sharing)   ")
    t_padded = bench(PaddedCounters, "Padded (no false sharing)")
    print(f"  Speedup from padding: {t_packed/t_padded:.2f}x")
    print("  (Note: CPython GIL may reduce parallelism; use PyPy or C extensions for true parallel)")

demonstrate_false_sharing()
```

**Connection to virtual memory:** False sharing occurs at the physical cache line level, regardless of virtual address layout. Two processes sharing a `MAP_SHARED` page where their respective data lands on the same cache line will suffer the same coherence traffic across NUMA nodes. This is more expensive — coherence messages cross the QPI/UPI interconnect rather than the L3 ring bus.

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

## 11. `io_uring` and Zero-Copy I/O: Where Virtual Memory Meets I/O

### The Classic `read()` Path and Double Copying

The traditional `read(fd, buf, len)` syscall:
1. User process calls `read()` → traps to kernel (KPTI: CR3 switch).
2. Kernel issues DMA to read from disk into a **kernel page cache** page.
3. Kernel `copy_to_user()`: copies data from page cache into the user buffer. This crosses the kernel/user boundary — two separate virtual address spaces, one physical copy.
4. Return to user space (CR3 switch back).

Two context switches + one memcpy per `read()`. For a web server reading 10,000 files per second, this is significant overhead.

### `mmap` as "Zero-Copy Read" from the Page Cache

`mmap(fd, ...)` maps the file's page cache pages directly into the process's virtual address space. No `copy_to_user()` — the user process reads directly from the same physical pages the kernel uses. The VMA's `vm_file` and `vm_ops->fault` handler map page cache pages on demand:

```python
import mmap
import os
import time

def compare_read_vs_mmap(path: str, size: int = None) -> None:
    """Compare traditional read() vs mmap for file I/O throughput."""
    if not os.path.exists(path):
        # Create a temp file for benchmarking
        path = "/tmp/bench_io.bin"
        with open(path, "wb") as f:
            f.write(os.urandom(64 * 1024 * 1024))  # 64MB

    file_size = os.path.getsize(path)
    size = size or file_size

    # --- Traditional read() ---
    t0 = time.perf_counter_ns()
    with open(path, "rb") as f:
        data_read = f.read(size)
    read_ms = (time.perf_counter_ns() - t0) / 1e6

    # --- mmap (zero-copy from page cache) ---
    t0 = time.perf_counter_ns()
    with open(path, "rb") as f:
        m = mmap.mmap(f.fileno(), size, access=mmap.ACCESS_READ)
        # Force all pages to load (simulate sequential read)
        checksum = 0
        for i in range(0, size, 4096):
            checksum ^= m[i]
        m.close()
    mmap_ms = (time.perf_counter_ns() - t0) / 1e6

    print(f"File I/O benchmark ({size // (1024*1024)}MB):")
    print(f"  read() syscall:    {read_ms:.1f}ms  ({size/read_ms/1e6*1000:.0f} MB/s)")
    print(f"  mmap (zero-copy):  {mmap_ms:.1f}ms  ({size/mmap_ms/1e6*1000:.0f} MB/s)")
    print(f"  Note: on second run, page cache is warm — both should be ~equal (no disk)")

compare_read_vs_mmap("/tmp/bench_io.bin")
```

### `io_uring` (Linux 5.1+): Asynchronous I/O Without Syscall Overhead

`io_uring` is a shared-memory ring buffer between user space and the kernel. Submission and completion of I/O operations do not require individual syscalls:

```
User space:                    Kernel space:
┌──────────────────────────┐  ┌──────────────────────────┐
│  Submission Queue (SQ)   │  │  SQ entries processed    │
│  [mmap'd into user space]│→ │  by kernel I/O subsystem │
└──────────────────────────┘  └──────────────────────────┘
┌──────────────────────────┐  ┌──────────────────────────┐
│  Completion Queue (CQ)   │← │  Results written here    │
│  [mmap'd into user space]│  │  by kernel (no syscall)  │
└──────────────────────────┘  └──────────────────────────┘
```

Both ring buffers are **shared mmaps** — no data copying, no syscall per operation. `IORING_SETUP_SQPOLL` even lets the kernel poll for submissions without any user-space syscall at all.

```python
import ctypes
import os

def setup_io_uring_concept():
    """
    Conceptual io_uring setup using raw syscalls.
    Production: use the 'liburing' C library or 'uring' Python package.
    """
    libc = ctypes.CDLL("libc.so.6", use_errno=True)
    
    # io_uring_setup(entries=256, params) — syscall 425 on x86-64
    SYS_IO_URING_SETUP = 425
    
    class IoUringParams(ctypes.Structure):
        _fields_ = [
            ("sq_entries",     ctypes.c_uint32),
            ("cq_entries",     ctypes.c_uint32),
            ("flags",          ctypes.c_uint32),
            ("sq_thread_cpu",  ctypes.c_uint32),
            ("sq_thread_idle", ctypes.c_uint32),
            ("features",       ctypes.c_uint32),
            ("wq_fd",          ctypes.c_uint32),
            ("resv",           ctypes.c_uint32 * 3),
            ("sq_off",         ctypes.c_uint8 * 40),
            ("cq_off",         ctypes.c_uint8 * 40),
        ]
    
    params = IoUringParams()
    fd = libc.syscall(SYS_IO_URING_SETUP, 256, ctypes.byref(params))
    
    if fd < 0:
        errno = ctypes.get_errno()
        print(f"io_uring_setup failed: errno={errno} (need Linux 5.1+, CONFIG_IO_URING)")
        return
    
    print(f"io_uring fd={fd} created")
    print(f"  SQ entries: {params.sq_entries}, CQ entries: {params.cq_entries}")
    print(f"  Features bitmask: {params.features:#010x}")
    print("  Next: mmap the SQ/CQ rings into user space for zero-syscall submission")
    os.close(fd)

setup_io_uring_concept()
```

**Why `io_uring` matters for virtual memory:** The submission and completion queues are `mmap`'d into user space — the same physical pages are accessible from both user space and the kernel. This is **zero-copy communication**, enabled entirely by the virtual memory abstraction and shared page mappings.

---

## 12. `cgroups v2` Memory Controller

Containers (Docker, Kubernetes) use **cgroups v2** to limit and monitor process memory. The memory controller exposes a hierarchy of files under `/sys/fs/cgroup/<group>/`:

| File | Purpose |
|---|---|
| `memory.max` | Hard limit: processes exceeding this are OOM-killed |
| `memory.high` | Soft limit: triggers reclaim before hitting max |
| `memory.current` | Current memory usage (RSS + page cache) |
| `memory.stat` | Detailed breakdown: anon, file, shmem, swap |
| `memory.pressure` | PSI (Pressure Stall Information) — time stalled on memory |
| `memory.oom.group` | Kill entire cgroup on OOM (not just one process) |
| `memory.swap.max` | Limit swap usage for the cgroup |

### PSI (Pressure Stall Information) — Memory Pressure Monitoring

PSI (Linux 4.20+, enabled by default in Linux 5.13+) measures the fraction of time tasks are stalled waiting for memory. Unlike RSS-based metrics, PSI captures actual performance impact:

```python
import os
import time
import threading

def read_memory_psi(cgroup_path: str = "/sys/fs/cgroup") -> dict:
    """
    Read PSI memory pressure. Returns fraction of time stalled.
    'some': at least one task stalled (workload impacted)
    'full': ALL tasks stalled (workload completely halted)
    """
    psi_path = f"{cgroup_path}/memory.pressure"
    if not os.path.exists(psi_path):
        # Fall back to system-wide PSI
        psi_path = "/proc/pressure/memory"
    
    result = {}
    try:
        with open(psi_path) as f:
            for line in f:
                parts = line.strip().split()
                if not parts:
                    continue
                kind = parts[0]  # "some" or "full"
                fields = {}
                for kv in parts[1:]:
                    k, v = kv.split("=")
                    fields[k] = float(v)
                result[kind] = fields
        
        some = result.get("some", {})
        full = result.get("full", {})
        print(f"Memory PSI:")
        print(f"  some (any task stalled): avg10={some.get('avg10',0):.2f}% "
              f"avg60={some.get('avg60',0):.2f}% avg300={some.get('avg300',0):.2f}%")
        print(f"  full (all tasks stalled): avg10={full.get('avg10',0):.2f}% "
              f"avg60={full.get('avg60',0):.2f}%")
        if some.get("avg10", 0) > 10:
            print("  ⚠ HIGH PRESSURE: >10% stall rate — consider increasing memory limit")
    except FileNotFoundError:
        print(f"PSI not available at {psi_path} (need Linux 4.20+ with CONFIG_PSI)")
    return result

read_memory_psi()

def read_cgroup_memory_stats(cgroup_path: str = "/sys/fs/cgroup") -> dict:
    """Read cgroup v2 memory statistics."""
    stats = {}
    
    # Current usage
    for fname in ("memory.current", "memory.max", "memory.high"):
        fpath = f"{cgroup_path}/{fname}"
        if os.path.exists(fpath):
            with open(fpath) as f:
                val = f.read().strip()
                stats[fname] = int(val) if val.isdigit() else val
    
    # Detailed stats from memory.stat
    stat_path = f"{cgroup_path}/memory.stat"
    if os.path.exists(stat_path):
        with open(stat_path) as f:
            for line in f:
                parts = line.strip().split()
                if len(parts) == 2 and parts[1].isdigit():
                    stats[f"stat.{parts[0]}"] = int(parts[1])
    
    if stats:
        current_mb = stats.get("memory.current", 0) / (1024*1024)
        limit = stats.get("memory.max", "unlimited")
        limit_mb = int(limit) / (1024*1024) if str(limit).isdigit() else "unlimited"
        print(f"cgroup memory: {current_mb:.0f}MB / {limit_mb}MB limit")
        
        anon_mb = stats.get("stat.anon", 0) / (1024*1024)
        file_mb = stats.get("stat.file", 0) / (1024*1024)
        print(f"  anon={anon_mb:.0f}MB  file_cache={file_mb:.0f}MB  "
              f"oom_kill={stats.get('stat.oom_kill', 0)}")
    return stats

read_cgroup_memory_stats()
```

### Kubernetes Memory Limits and the OOM Kill Loop

When a Kubernetes pod exceeds `resources.limits.memory`, the cgroup's hard limit triggers. The kernel OOM-kills the process whose `oom_score` is highest within the cgroup. If the pod keeps restarting and hitting the same limit, it enters **OOMKill loop** (`CrashLoopBackOff`).

Practical implications for Python services:
- `sys.setrecursionlimit` controls stack depth but not stack memory (each frame is ~few KB of heap objects).
- Python's `gc.collect()` releases cyclic garbage but doesn't release memory to the OS immediately — the allocator keeps it for future use. Use `ctypes.CDLL("libc.so.6").malloc_trim(0)` to return free memory to the kernel.
- Set `MALLOC_TRIM_THRESHOLD_=-1` or use `jemalloc` via `LD_PRELOAD` for more aggressive memory return.

```python
import ctypes
import gc

def release_memory_to_os() -> int:
    """
    Attempt to return free Python heap memory to the OS.
    Works by combining gc.collect() with glibc malloc_trim().
    Returns approximate bytes freed (estimated from RSS delta).
    """
    import resource
    rss_before = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss
    
    gc.collect(2)  # Full collection including long-lived objects
    
    libc = ctypes.CDLL("libc.so.6")
    # malloc_trim(0): release all free glibc heap memory back to the OS via madvise(MADV_DONTNEED)
    libc.malloc_trim(0)
    
    rss_after = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss
    freed_kb = rss_before - rss_after
    print(f"malloc_trim: RSS {rss_before}KB → {rss_after}KB, freed ~{freed_kb}KB")
    return freed_kb * 1024

release_memory_to_os()
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
| PSI enabled | `/proc/sys/kernel/psi` | `1` (5.13+) | Pressure Stall Information |
| Zswap enabled | `/sys/module/zswap/parameters/enabled` | `Y` | Compressed in-memory swap cache |

---

## 13. Profiling Workflow: From Symptom to Root Cause

This section ties all the tools together into a practical diagnostic runbook.

### Symptom: High RSS, Suspected CoW Waste

```bash
# Step 1: Identify the problem process
ps aux --sort=-%mem | head -10

# Step 2: Compare RSS vs PSS across workers (PSS accounts for sharing)
for pid in $(pgrep -f "gunicorn"); do
    rss=$(cat /proc/$pid/status | grep VmRSS | awk '{print $2}')
    pss=$(awk '/^Pss:/{sum+=$2} END{print sum}' /proc/$pid/smaps 2>/dev/null)
    echo "PID $pid: RSS=${rss}kB, PSS=${pss}kB, CoW_waste=$((rss-pss))kB"
done

# Step 3: Find which VMAs are CoW-dirtied
awk '/Private_Dirty/{dirty+=$2} /^[0-9a-f]/{if(dirty>0){print dirty" "$0; dirty=0}}' \
    /proc/<pid>/smaps | sort -rn | head -20
```

### Symptom: TLB Pressure, High CPU% on Memory-Intensive Code

```bash
# Step 1: Profile with perf — measure dTLB miss rate
perf stat -e dTLB-load-misses,dTLB-loads,LLC-load-misses,LLC-loads \
    -p <pid> -- sleep 10

# Step 2: Find the hot virtual addresses causing TLB misses
perf record -e dTLB-load-misses -c 1000 -p <pid> -- sleep 10
perf report --sort=dso,symbol

# Step 3: Check huge page promotion rate
grep -E "thp_fault_alloc|thp_collapse_alloc|thp_split_page" /proc/vmstat
# thp_fault_alloc high = THP working
# thp_split_page high = THP being split (fragmentation, database disabling it)
```

### Symptom: NUMA-Remote Memory Access (High Latency, Low Throughput)

```bash
# Step 1: Check NUMA statistics
numastat -p <pid>   # Per-NUMA-node memory breakdown for a process
numastat            # System-wide NUMA hit/miss rates

# Step 2: perf c2c (cache-to-cache) — detect NUMA false sharing
perf c2c record -p <pid> -- sleep 10
perf c2c report --call-graph --stdio

# Step 3: Pin the process to a NUMA node
numactl --cpunodebind=0 --membind=0 python3 server.py
```

```python
import subprocess
import os

def run_memory_diagnostic(pid: int = None) -> None:
    """Full memory diagnostic suite. Run as root for complete output."""
    pid = pid or os.getpid()
    
    print(f"\n{'='*60}")
    print(f"MEMORY DIAGNOSTIC FOR PID {pid}")
    print(f"{'='*60}")
    
    # 1. RSS / PSS / Private_Dirty summary
    try:
        rss = pss = private_dirty = 0
        with open(f"/proc/{pid}/smaps") as f:
            for line in f:
                if line.startswith("Rss:"):
                    rss += int(line.split()[1])
                elif line.startswith("Pss:"):
                    pss += int(line.split()[1])
                elif line.startswith("Private_Dirty:"):
                    private_dirty += int(line.split()[1])
        print(f"\n[Memory Sharing]")
        print(f"  RSS:          {rss:>8,} KB  (total resident)")
        print(f"  PSS:          {pss:>8,} KB  (proportional — accounts for sharing)")
        print(f"  Private_Dirty:{private_dirty:>8,} KB  (CoW-copied pages you exclusively own)")
        sharing_pct = (rss - pss) / max(rss, 1) * 100
        print(f"  Sharing:      {sharing_pct:.1f}% of RSS is shared with other processes")
    except (FileNotFoundError, PermissionError) as e:
        print(f"  Cannot read smaps: {e}")
    
    # 2. Page fault rates from /proc/pid/stat
    try:
        with open(f"/proc/{pid}/stat") as f:
            fields = f.read().split()
        minflt = int(fields[9])   # minor faults
        majflt = int(fields[11])  # major faults
        print(f"\n[Page Fault History]")
        print(f"  Minor faults: {minflt:>10,}  (page in cache, just mapped)")
        print(f"  Major faults: {majflt:>10,}  (required disk I/O — expensive)")
    except (FileNotFoundError, IndexError) as e:
        print(f"  Cannot read stat: {e}")
    
    # 3. Memory pressure
    try:
        with open("/proc/pressure/memory") as f:
            for line in f:
                parts = dict(kv.split("=") for kv in line.split()[1:]
                             if "=" in kv)
                kind = line.split()[0]
                print(f"  PSI {kind}:  avg10={float(parts.get('avg10',0)):.2f}%  "
                      f"avg60={float(parts.get('avg60',0)):.2f}%") \
                    if kind in ("some", "full") else None
    except FileNotFoundError:
        pass
    
    # 4. THP stats
    try:
        with open("/proc/vmstat") as f:
            vmstat = dict(line.split() for line in f if len(line.split()) == 2)
        thp_fault = int(vmstat.get("thp_fault_alloc", 0))
        thp_split = int(vmstat.get("thp_split_page", 0))
        print(f"\n[THP Activity]")
        print(f"  THP allocated: {thp_fault:>10,}")
        print(f"  THP split:     {thp_split:>10,}  (splits are expensive — check THP config)")
    except FileNotFoundError:
        pass

run_memory_diagnostic()
```

### Interview Questions on Virtual Memory (Senior Engineer Level)

**Q1:** A process calls `mmap(NULL, 1GB, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)`. How much physical memory is consumed immediately after this call returns?

*Answer:* Zero. The kernel creates a VMA but does not allocate physical pages. The pages are demand-zeroed — allocated one-at-a-time when first written.

**Q2:** Two processes share a `MAP_SHARED` anonymous mmap. Process A writes to page N. Does Process B see the write?

*Answer:* Yes, immediately (on the same CPU) or after a cache coherence event (cross-CPU). `MAP_SHARED` means both processes' PTEs point to the same physical page. The MESI protocol ensures cache coherence. There is no CoW — writes are visible to all sharers.

**Q3:** Why does `mprotect(PROT_READ)` on a 1GB anonymous region take longer on a 128-CPU server than on a 1-CPU VM?

*Answer:* `mprotect` must change the PTE permission bits for all pages in the range. After updating PTEs, it must issue a TLB shootdown to all CPUs that might have the old mapping cached. On a 128-CPU server, this sends 127 IPIs. On a 1-CPU system, no IPIs are needed — just a local `INVLPG`.

**Q4:** A Python ML server loads a 10GB model, then forks 16 workers. The kernel shows 10GB RSS for the parent and 10GB RSS per worker = 170GB total RSS. The machine has 200GB RAM and is now OOM. Explain and fix.

*Answer:* After fork, parent and workers share the model pages (CoW, not yet copied). However, Python's reference counting mutates `ob_refcnt` on every object access, dirtying pages. Each worker's 10GB gradually becomes private. Fix: Store the model as a read-only `np.memmap` or `multiprocessing.shared_memory` block — these do not have per-element Python objects, so no refcount mutations, so no CoW copies.

**Q5:** Explain why Linux's OOM killer sometimes kills the "wrong" process.

*Answer:* The OOM killer uses `oom_score = RSS + swap_usage + (time_running penalty)`, adjusted by `oom_score_adj`. It picks the highest score. But "most memory used" is not the same as "most memory to reclaim" — a process might have 90% of its RSS shared (CoW, not private), while another process with lower RSS is entirely private. The OOM killer's score doesn't account for page sharing, so it may kill a shared-memory-heavy process when killing a purely-private process would recover more RAM. Linux 5.18+ improves this heuristic slightly.

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
| PSI enabled | `/proc/sys/kernel/psi` | `1` (5.13+) | Pressure Stall Information |
| Zswap enabled | `/sys/module/zswap/parameters/enabled` | `Y` | Compressed in-memory swap cache |

---

## Summary

Virtual memory is the foundational abstraction of modern OS design. Understanding its implementation — four-level page tables, TLB shootdowns, huge pages, CoW semantics, NUMA topology, speculative execution mitigations, and the buddy + SLUB allocators — is essential for reasoning about performance in any language, including Python. The `mmap` module exposes these primitives directly: anonymous mmaps exercise demand paging, shared mmaps enable zero-copy IPC, `os.fork()` demonstrates CoW in action, and `ctypes` + `/proc` give you a live window into page table state.

The performance limits of Python web servers, ML serving systems, and databases all trace back to the mechanisms described in this lesson: CoW-induced refcount copies under Gunicorn forks, TLB thrashing from 4KB pages on 100GB working sets, NUMA-remote allocations when the scheduler migrates threads, cache coherence false sharing, and KPTI overhead on syscall-heavy workloads. The profiling runbook in Section 13 and the Python diagnostic tools throughout give you the instruments to measure each of these precisely — no guesswork, just hardware counters and kernel interfaces telling you exactly what is happening and why.

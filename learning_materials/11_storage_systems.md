# Lesson 11: Storage Systems — Block Devices, Filesystems, NVMe, and Erasure Coding

> References: *The Design and Implementation of the ext2 Filesystem* (Card et al. 1994), *The ext4 Filesystem* (Mathur et al. 2007), *ZFS: The Last Word in Filesystems* (Bonwick & Moore 2003), *NVM Express Base Specification 2.0*, *A Fast File System for UNIX* (McKusick et al. 1984).

---

## 1. The Block Device Layer

Everything the kernel stores on persistent media passes through the **block device layer**. A block device is any storage medium that exposes a flat, random-access array of fixed-size sectors. HDDs, SSDs, NVMe drives, RAID arrays, device-mapper logical volumes, and loop-back devices all appear to the kernel as a uniform abstraction: read or write *n* sectors starting at sector *k*.

### Sector Size and Alignment

Historically, every disk sector was 512 bytes — a constraint inherited from the IBM 3505 disk drives of the 1970s. Modern HDDs and SSDs use 4096-byte native sectors (4Kn), though most present a 512-byte emulation layer (512e) for compatibility. Misaligned I/O on a 512e drive causes a read-modify-write amplification: a 512-byte write that straddles two 4K physical sectors forces the drive firmware to read both physical sectors, merge the change, and write both back — doubling write amplification before the FTL even touches it.

The kernel exposes the physical sector size via `ioctl(BLKPBSZGET)` and the logical sector size via `ioctl(BLKSSZGET)`. Partition tools that ignore these values create misaligned partitions with measurable performance penalties (10–30% on sustained sequential workloads on HDD).

### The I/O Scheduler

The kernel's **block layer** sits between the filesystem and the device driver. It queues I/O requests and may reorder or merge them before submission. This matters enormously for HDDs (where seek latency of 3–15 ms dominates) and less so for SSDs.

| Scheduler | Target | Strategy |
|-----------|--------|----------|
| `none`/`noop` | NVMe, fast SSD | Passthrough — no reordering. The device is fast enough and has its own internal scheduler. |
| `mq-deadline` | HDD, SATA SSD | Enforces deadlines to prevent starvation; sorts by sector to minimize seek distance. Multi-queue aware. |
| `bfq` | Desktop HDD | Budget Fair Queuing — tracks per-process I/O budgets, provides responsiveness. Best for mixed workloads. |
| `kyber` | Very fast NVMe | Latency-aware; targets configurable read/write latency rather than throughput. |

CFQ (Completely Fair Queuing) was removed in Linux 5.0 after mq-deadline and BFQ proved superior in all tested workloads.

### The Full I/O Stack

```
Application (read/write syscall)
    │
    ▼
VFS (vfs_read / vfs_write)
    │
    ▼
Page Cache (check if page already in memory)
    │
    ▼
Filesystem (ext4 / xfs / btrfs — translate file offset to block address)
    │
    ▼
Block Layer (bio structure — merge small I/Os)
    │
    ▼
I/O Scheduler (reorder for seek optimization)
    │
    ▼
Device Driver (NVMe / AHCI / virtio-blk)
    │
    ▼
Hardware (PCIe / SATA)
```

Each layer adds latency but also provides value. The page cache eliminates disk I/Os entirely for hot data. The filesystem provides metadata consistency. The block layer amortizes syscall overhead by merging adjacent writes into a single large I/O request.

### Direct I/O (O_DIRECT)

`O_DIRECT` bypasses the page cache. Data moves directly between a userspace buffer and the block device. Databases (PostgreSQL, Oracle, MySQL InnoDB) use this because they implement their own buffer pools tuned to their access patterns. The kernel's page cache is a liability when the database already knows which pages are hot.

Constraints: the userspace buffer must be aligned to the logical sector size (512 or 4096 bytes), the file offset must be sector-aligned, and the transfer length must be a multiple of the sector size. Violating any of these causes `EINVAL`.

```python
import os, time, mmap, ctypes

BLOCK = 4096
PATH = '/tmp/test_io'

# Create 100 MB test file
with open(PATH, 'wb') as f:
    f.write(b'x' * (1024 * 1024 * 100))

# --- Buffered read ---
with open(PATH, 'rb') as f:
    t = time.perf_counter()
    data = f.read()
    buffered_ms = (time.perf_counter() - t) * 1000
print(f"Buffered read:  {buffered_ms:.1f} ms  ({len(data) / 1e6:.0f} MB)")

# --- Direct I/O (Linux, O_DIRECT = 0x4000) ---
# mmap with MAP_ANONYMOUS gives page-aligned memory.
# For O_DIRECT we need the buffer address to be sector-aligned.
O_DIRECT = 0x4000
try:
    fd = os.open(PATH, os.O_RDONLY | O_DIRECT)
    # Anonymous mmap is always page-aligned (≥ 4096), satisfying 512-byte alignment.
    buf = mmap.mmap(-1, 1024 * 1024 * 100)
    t = time.perf_counter()
    # os.readv reads into a list of buffers; buf supports the buffer protocol
    n = os.readv(fd, [buf])
    direct_ms = (time.perf_counter() - t) * 1000
    print(f"Direct I/O read: {direct_ms:.1f} ms  ({n / 1e6:.0f} MB)")
    os.close(fd)
    buf.close()
except OSError as e:
    print(f"Direct I/O unavailable on this path: {e}")

# Second buffered read (data now in page cache) shows cache effect
with open(PATH, 'rb') as f:
    t = time.perf_counter()
    data = f.read()
    cached_ms = (time.perf_counter() - t) * 1000
print(f"Cached read:    {cached_ms:.1f} ms  (page cache hit)")
```

The first buffered read populates the page cache. The second is served entirely from RAM — often 10–50x faster. `O_DIRECT` is always slower than a cache hit but gives consistent, cache-independent latency — important for databases that need predictable performance.

---

## 2. NVMe Protocol Internals

NVMe (Non-Volatile Memory Express) was designed from scratch for flash. The AHCI protocol that SATA uses was designed for spinning disks in 2004: it has one command queue with 32 slots. NVMe has **65535 queues × 65535 commands each**, and each queue is independent — no global lock, no head-of-line blocking.

### PCIe Bandwidth

| Interface | Bandwidth |
|-----------|-----------|
| SATA III | 600 MB/s |
| PCIe 3.0 ×4 | 3.9 GB/s |
| PCIe 4.0 ×4 | 7.8 GB/s |
| PCIe 5.0 ×4 | 15.6 GB/s |

NVMe drives saturate PCIe 4.0 ×4 (e.g., Samsung 990 Pro: 7.4 GB/s sequential read). SATA SSDs are bottlenecked by the interface, not the NAND.

### NVMe Command Submission

An NVMe command is a 64-byte structure placed in a **Submission Queue (SQ)** in host memory. After writing the command, the host increments the SQ tail doorbell — a 32-bit write to a memory-mapped PCIe BAR (Base Address Register). The controller monitors this register via PCIe DMA and processes the command. The result appears in a **Completion Queue (CQ)** entry (16 bytes), and the controller signals the host with a PCIe MSI-X interrupt (or the host can poll — polling is faster for ultra-low-latency workloads).

The doorbell mechanism means submitting an NVMe I/O requires exactly **one memory-mapped write** at the host side. Compare to a full system call (AHCI): `ioctl` → trap to kernel → AHCI driver builds FIS → write to PCI I/O space → interrupt. NVMe's submission path is deliberately thin.

### io_uring

`io_uring` (Linux 5.1, Jens Axboe 2019) takes the NVMe queue philosophy to the Linux syscall interface. Two shared ring buffers (mapped in both kernel and userspace):
- **SQ (Submission Queue)**: userspace writes I/O requests
- **CQ (Completion Queue)**: kernel writes completions

With `IORING_SETUP_SQPOLL`, a kernel thread polls the SQ — zero syscalls per I/O once the thread is running. Measured latency: ~1.5 µs per 4K NVMe read (vs ~4 µs for `epoll`-based async I/O).

```python
# io_uring via asyncio + aiofiles (Python wrapper over the kernel's async path)
import asyncio
import aiofiles
import time
import os

PATH = '/tmp/test_io'

async def read_once(path: str) -> int:
    async with aiofiles.open(path, 'rb') as f:
        data = await f.read()
        return len(data)

async def benchmark_concurrent_reads(n: int = 10):
    t = time.perf_counter()
    results = await asyncio.gather(*[read_once(PATH) for _ in range(n)])
    elapsed = (time.perf_counter() - t) * 1000
    total_mb = sum(results) / 1e6
    print(f"{n} concurrent reads: {elapsed:.1f} ms  "
          f"({total_mb / (elapsed / 1000):.0f} MB/s aggregate)")

async def benchmark_sequential_reads(n: int = 10):
    t = time.perf_counter()
    for _ in range(n):
        await read_once(PATH)
    elapsed = (time.perf_counter() - t) * 1000
    print(f"{n} sequential reads: {elapsed:.1f} ms")

async def main():
    print("=== io_uring-backed async I/O benchmark ===")
    await benchmark_sequential_reads()
    await benchmark_concurrent_reads()

if os.path.exists(PATH):
    asyncio.run(main())
```

Concurrent reads show the benefit of async I/O: the event loop issues all 10 reads simultaneously, the kernel (via io_uring internally) submits them all to the NVMe controller's multiple queues, and the drive's internal parallelism services them concurrently.

---

## 3. VFS — Virtual Filesystem Switch

VFS is the kernel's filesystem abstraction layer, introduced in SunOS 2.0 (1985) and reimplemented in Linux. Every filesystem (ext4, xfs, btrfs, tmpfs, procfs, nfs) registers itself with VFS by providing function pointer tables. A userspace `open()` call always goes through `vfs_open()`, which dispatches to the appropriate filesystem's `->open()` method.

### Core VFS Objects

**`inode`** — represents one file or directory (metadata only, no name). Fields include: `i_mode` (permissions), `i_uid`/`i_gid`, `i_size`, `i_atime`/`i_mtime`/`i_ctime`, `i_blocks` (number of 512-byte blocks allocated), `i_mapping` (pointer to the address space — the page cache for this inode). The on-disk inode and in-memory `struct inode` are different: the in-memory version carries locking, reference counting, and the page cache pointer.

**`dentry`** (directory entry) — maps one name component to an inode. The dentry cache (dcache) is a hash table of recently resolved path components. A `stat("/usr/lib/libc.so")` call does: look up `"usr"` in dcache (hit), look up `"lib"` under `usr` (hit), look up `"libc.so"` under `lib` (miss — disk read). The second `stat()` call for the same path is entirely in cache.

**`file`** — represents one open file descriptor. Contains: current position (`f_pos`), flags (`f_flags`), pointer to `file_operations` (`f_op`), and pointer to the inode (via `f_inode`). Multiple `file` objects can point to the same `inode` (multiple opens of the same file).

**`super_block`** — represents one mounted filesystem instance. Contains: block size, filesystem type pointer, list of all inodes, dirty inode list, operations (`->sync_fs`, `->put_super`, `->statfs`).

### Demonstrating the Dentry Cache

```python
import os, time, stat

def time_stat(path: str, label: str, iterations: int = 10_000) -> float:
    t = time.perf_counter()
    for _ in range(iterations):
        os.stat(path)
    elapsed_us = (time.perf_counter() - t) * 1e6 / iterations
    print(f"{label}: {elapsed_us:.2f} µs/stat")
    return elapsed_us

# Warm up dentry cache
os.stat('/usr/lib')

cold = time_stat('/proc/sys/kernel/hostname', 'procfs (no cache benefit)', 1000)
warm = time_stat('/usr/lib', 'cached path (dcache hit)', 100_000)

# stat() also returns inode fields
s = os.stat('/usr/lib')
print(f"\nInode fields for /usr/lib:")
print(f"  inode number:  {s.st_ino}")
print(f"  mode:          {oct(s.st_mode)}")
print(f"  uid/gid:       {s.st_uid}/{s.st_gid}")
print(f"  size:          {s.st_size} bytes")
print(f"  block size:    {s.st_blksize} bytes")
print(f"  blocks:        {s.st_blocks} × 512 = {s.st_blocks * 512} bytes on disk")
print(f"  mtime:         {s.st_mtime:.3f}")
```

The dentry cache typically makes the second `stat()` of a known path 5–20x faster than the first because no disk I/O is required for the path resolution.

---

## 4. ext4 Filesystem Internals

ext4 is the evolution of ext3 (which evolved from ext2, which evolved from the 1992 MINIX filesystem). It is the default filesystem on most Linux distributions.

### On-Disk Layout

The disk is divided into fixed-size **block groups** (typically 128 MB each). Every block group contains:

```
[superblock backup][group descriptor table backup][data block bitmap][inode bitmap][inode table][data blocks...]
```

The primary superblock lives at byte offset 1024 (block 0 is left for boot sectors). Block group 0's superblock is the canonical copy; groups 1, 3, 5, 7, 9, 25, 49... (powers of 3, 5, and 7) contain backups (sparse_super feature).

### Extents vs. Indirect Blocks

ext3 used *indirect block pointers*: 12 direct blocks, 1 singly-indirect block (points to a block of block addresses), 1 doubly-indirect, 1 triply-indirect. For a 4 KB block size this gives a max file size of ~16 TB but requires up to 4 disk reads just to find the block containing a given byte in a large file.

ext4 uses **extents**: a single `ext4_extent` structure records `(logical_block_start, physical_block_start, length)` — a contiguous run of up to 32,768 blocks (128 MB). Small files fit their entire extent tree directly in the inode (4 extents = 12 bytes of inode space). Large files use an extent tree with internal nodes. Seeking to an arbitrary offset in a large sequential file now costs O(log n) in tree height rather than 3 block reads.

### Journal Modes

| Mode | What's journaled | Crash safety | Performance |
|------|-----------------|--------------|-------------|
| `data=journaled` | Data + metadata | Highest | Lowest (data written twice) |
| `data=ordered` | Metadata only, data flushed first | Good | Default |
| `data=writeback` | Metadata only, data whenever | Lowest | Highest |

In `data=ordered` mode (default), ext4 ensures that file data reaches disk *before* the journal commit that updates the file size. This prevents a crash from leaving a file with a large `i_size` pointing to blocks containing stale data.

### Delayed Allocation

ext4 does not allocate physical blocks when `write()` is called. Instead, it marks the pages dirty in the page cache and defers allocation until the dirty data must actually be written out (typically 30 seconds, or when `fsync()` is called, or when memory is needed). This allows the allocator to see the full extent of a write and allocate contiguous blocks — reducing fragmentation dramatically. The risk: if the process crashes before writeback, the data is lost even though `write()` returned success.

### Parsing the Superblock

```python
import struct, os, sys

SUPERBLOCK_OFFSET = 1024

# ext4 superblock field offsets (bytes from start of superblock)
SB_FIELDS = {
    'inodes_count':       (0,   '<I'),
    'blocks_count_lo':    (4,   '<I'),
    'r_blocks_count_lo':  (8,   '<I'),
    'free_blocks_lo':     (12,  '<I'),
    'free_inodes':        (16,  '<I'),
    'first_data_block':   (20,  '<I'),
    'log_block_size':     (24,  '<I'),   # block_size = 1024 << this
    'log_cluster_size':   (28,  '<I'),
    'blocks_per_group':   (32,  '<I'),
    'inodes_per_group':   (40,  '<I'),
    'magic':              (56,  '<H'),   # must be 0xEF53
    'state':              (58,  '<H'),   # 1=clean, 2=errors
    'rev_level':          (76,  '<I'),   # 0=original, 1=dynamic
    'inode_size':         (88,  '<H'),   # 128 (ext2) or 256 (ext4)
    'feature_compat':     (92,  '<I'),
    'feature_incompat':   (96,  '<I'),
}

INCOMPAT_FEATURES = {
    0x0001: 'compression',
    0x0002: 'filetype',
    0x0004: 'recover',
    0x0008: 'journal_dev',
    0x0010: 'meta_bg',
    0x0040: 'extents',
    0x0080: '64bit',
    0x0100: 'mmp',
    0x0200: 'flex_bg',
    0x1000: 'encrypt',
}

def read_ext4_superblock(device_path: str) -> dict:
    """Read and parse an ext4 superblock from a block device or image file."""
    try:
        with open(device_path, 'rb') as f:
            f.seek(SUPERBLOCK_OFFSET)
            data = f.read(1024)
    except PermissionError:
        print(f"Need read permission on {device_path} (try sudo)")
        return {}

    parsed = {}
    for name, (offset, fmt) in SB_FIELDS.items():
        size = struct.calcsize(fmt)
        parsed[name] = struct.unpack_from(fmt, data, offset)[0]

    magic = parsed['magic']
    block_size = 1024 << parsed['log_block_size']
    total_blocks = parsed['blocks_count_lo']
    total_gb = total_blocks * block_size / (1024 ** 3)
    free_pct = 100 * parsed['free_blocks_lo'] / max(total_blocks, 1)

    features = []
    incompat = parsed['feature_incompat']
    for bit, name in INCOMPAT_FEATURES.items():
        if incompat & bit:
            features.append(name)

    print(f"=== ext4 Superblock: {device_path} ===")
    print(f"  Magic:          {hex(magic)} {'✓' if magic == 0xEF53 else '✗ INVALID'}")
    print(f"  State:          {'clean' if parsed['state'] == 1 else 'unclean (needs fsck)'}")
    print(f"  Block size:     {block_size} bytes")
    print(f"  Total blocks:   {total_blocks:,}  ({total_gb:.1f} GB)")
    print(f"  Free blocks:    {parsed['free_blocks_lo']:,}  ({free_pct:.1f}% free)")
    print(f"  Total inodes:   {parsed['inodes_count']:,}")
    print(f"  Free inodes:    {parsed['free_inodes']:,}")
    print(f"  Inode size:     {parsed['inode_size']} bytes")
    print(f"  Incompat flags: {', '.join(features)}")
    return parsed

# Example usage (requires a readable block device or ext4 image):
# read_ext4_superblock('/dev/sda1')
# read_ext4_superblock('/path/to/disk.img')

# Demonstrate on a loop device if possible
import subprocess, tempfile
try:
    img = tempfile.mktemp(suffix='.img')
    subprocess.run(['dd', 'if=/dev/zero', f'of={img}', 'bs=1M', 'count=64'],
                   capture_output=True, check=True)
    subprocess.run(['mkfs.ext4', '-q', img], capture_output=True, check=True)
    read_ext4_superblock(img)
    os.unlink(img)
except (subprocess.CalledProcessError, FileNotFoundError):
    print("mkfs.ext4 not available; skipping live superblock demo.")
```

---

## 5. ZFS — A Different Philosophy

ZFS (Zettabyte File System) was designed at Sun Microsystems by Jeff Bonwick and Bill Moore and open-sourced in 2005. It combines the volume manager and filesystem into one layer, which eliminates an entire class of RAID-5 write holes and provides end-to-end data integrity.

### Copy-on-Write Semantics

ZFS never overwrites data in place. When modifying a block:
1. Allocate a new block in free space.
2. Write the new data.
3. Update the parent block pointer atomically.
4. Mark the old block free.

Because step 3 is a single atomic pointer swap, the filesystem is **always consistent** — there is no window where the structure is partially updated. No journal is needed. A crash at any point leaves either the old or new version intact.

**Snapshots are free**: a snapshot saves the current root block pointer. Since old blocks are never overwritten, all blocks referenced by the snapshot remain valid indefinitely. Only blocks modified *after* the snapshot are allocated anew. Storage cost of a snapshot is exactly the blocks that changed, not a full copy.

### On-Disk Structure

```
Vdev (physical device)
 └── Uberblock array (256 bytes × 128 copies in first 128 KB)
      └── Meta Object Set (MOS)
           ├── Dataset directory
           │    ├── Filesystem dataset → dnode tree → data
           │    └── Snapshot dataset  → frozen dnode tree
           └── Pool configuration, history, properties
```

The **uberblock** is ZFS's equivalent of a superblock. ZFS writes 128 copies of it in the first 128 KB of each vdev (in a round-robin pattern). On mount, ZFS scans all 128 slots and picks the one with the highest valid transaction group (TXG) number. Firmware errors that corrupt one uberblock cannot prevent mount.

### Transaction Groups (TXG)

ZFS batches writes into **transaction groups** (default 5-second intervals). All dirty data for a TXG is written, then a new uberblock is atomically written. The write order is strictly enforced: leaf data blocks → indirect blocks → MOS → uberblock. This preserves the tree invariant: any valid uberblock points to a fully consistent tree.

### ARC (Adaptive Replacement Cache)

The Linux page cache uses LRU eviction. Sequential scan over 10 GB of data evicts 10 GB of hot working set. ZFS's **ARC** (Adaptive Replacement Cache, Megiddo & Modha 2003) maintains two LRU lists:
- **T1**: recently used once (recency)  
- **T2**: used more than once (frequency)

And two ghost lists (metadata only, no data) that track recently evicted entries from T1 and T2. When a ghost entry is accessed, the ARC adapts the balance between T1 and T2. Sequential scan promotes entries into T1 but they stay there (no ghost hit means no T2 promotion) and get evicted quickly — the hot T2 working set survives.

### RAID-Z

RAID-Z1/Z2/Z3 are ZFS's software RAID. The critical improvement over hardware RAID-5 is **variable stripe width**: a write of 3 blocks uses a 3-wide stripe with 1 parity block. A write of 7 blocks uses a 7-wide stripe. This eliminates the **write hole**: in RAID-5, a partial stripe write (e.g., updating 1 of 4 data blocks) must read-modify-write the parity — if power fails during this, data and parity are inconsistent. ZFS's CoW means partial stripe writes never occur: the entire new stripe is written to new blocks atomically.

### Python: CoW Filesystem Simulation

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Any, Dict, Optional, List
import copy
import hashlib

@dataclass
class BlockPointer:
    block_id: int
    checksum: bytes  # stored in PARENT, like real ZFS

@dataclass
class DataNode:
    data: Any = None
    children: Dict[str, 'DataNode'] = field(default_factory=dict)
    block_id: int = 0

    def checksum(self) -> bytes:
        payload = str(self.data).encode() + str(sorted(self.children.keys())).encode()
        return hashlib.sha256(payload).digest()[:8]

class CoWFilesystem:
    def __init__(self):
        self._block_store: Dict[int, DataNode] = {}
        self._next_block = 1
        self._root_id = self._alloc(DataNode())
        self.snapshots: Dict[str, int] = {}  # name → root block_id
        self._txg = 0

    def _alloc(self, node: DataNode) -> int:
        bid = self._next_block
        self._next_block += 1
        node.block_id = bid
        self._block_store[bid] = node
        return bid

    def _get(self, block_id: int) -> DataNode:
        return self._block_store[block_id]

    def _cow_write(self, node: DataNode) -> DataNode:
        """Clone a node (CoW): allocate new block, copy data, return new node."""
        new_node = DataNode(
            data=node.data,
            children=dict(node.children)
        )
        self._alloc(new_node)
        return new_node

    def write(self, path: str, data: Any) -> None:
        """CoW write: walk the path, clone each ancestor node, update pointers."""
        self._txg += 1
        parts = [p for p in path.strip('/').split('/') if p]

        # Collect path nodes
        path_ids: List[int] = [self._root_id]
        current = self._get(self._root_id)
        for part in parts:
            if part not in current.children:
                current.children[part] = self._alloc(DataNode())
            next_id = current.children[part]
            path_ids.append(next_id)
            current = self._get(next_id)

        # Write data node (CoW: new block)
        new_leaf = self._cow_write(current)
        new_leaf.data = data
        new_ids = [new_leaf.block_id]

        # Walk back up, cloning each parent and updating the child pointer
        for i in range(len(parts) - 1, -1, -1):
            parent = self._cow_write(self._get(path_ids[i]))
            parent.children[parts[i]] = new_ids[0]
            new_ids.insert(0, parent.block_id)

        self._root_id = new_ids[0]

    def read(self, path: str) -> Optional[Any]:
        parts = [p for p in path.strip('/').split('/') if p]
        current = self._get(self._root_id)
        for part in parts:
            if part not in current.children:
                return None
            current = self._get(current.children[part])
        return current.data

    def snapshot(self, name: str) -> None:
        """O(1): just record the current root block_id."""
        self.snapshots[name] = self._root_id
        print(f"Snapshot '{name}': root block {self._root_id} "
              f"({len(self._block_store)} blocks total)")

    def rollback(self, name: str) -> None:
        self._root_id = self.snapshots[name]
        print(f"Rolled back to '{name}': root block {self._root_id}")

    def read_from_snapshot(self, snapshot: str, path: str) -> Optional[Any]:
        root_id = self.snapshots[snapshot]
        parts = [p for p in path.strip('/').split('/') if p]
        current = self._get(root_id)
        for part in parts:
            if part not in current.children:
                return None
            current = self._get(current.children[part])
        return current.data

def demo_cow():
    fs = CoWFilesystem()
    fs.write('/etc/hostname', 'server-01')
    fs.write('/etc/passwd', 'root:x:0:0')
    fs.write('/var/log/syslog', 'boot message')
    fs.snapshot('baseline')

    fs.write('/etc/hostname', 'server-02')  # Modified
    fs.write('/var/log/syslog', 'new log entry')
    fs.snapshot('after-rename')

    print(f"Current hostname: {fs.read('/etc/hostname')}")
    print(f"Baseline hostname: {fs.read_from_snapshot('baseline', '/etc/hostname')}")
    print(f"Blocks allocated: {fs._next_block - 1} (snapshot cost = 0)")
    fs.rollback('baseline')
    print(f"After rollback:  {fs.read('/etc/hostname')}")

demo_cow()
```

---

## 6. SSD Internals and the Flash Translation Layer

### NAND Flash Cell Types

| Type | Bits/Cell | P/E Cycles | Use Case |
|------|-----------|------------|----------|
| SLC | 1 | ~100,000 | Enterprise, embedded |
| MLC | 2 | ~3,000 | Prosumer, early consumer |
| TLC | 3 | ~1,000 | Consumer (2024 mainstream) |
| QLC | 4 | ~300 | High-density archive |
| PLC | 5 | ~100 | Experimental |

NAND must be **erased before writing**. Erase granularity is a **block** (128–512 pages, typically 1–16 MB). Write granularity is a **page** (4–16 KB). You cannot overwrite a page — you erase the containing block (resetting all bits to 1), then program (write 0s into) individual pages. This fundamental asymmetry requires the Flash Translation Layer.

### Flash Translation Layer (FTL)

The FTL runs in the SSD's controller firmware. It maintains a **Logical-to-Physical (L2P) map** translating host LBAs to physical page addresses (PPAs). Key functions:

- **Address translation**: every host read/write is remapped through L2P.
- **Out-of-place updates**: a write to LBA *k* allocates a new physical page, updates L2P[k], and marks the old physical page as stale.
- **Garbage collection (GC)**: when free physical pages run low, GC picks a victim block, copies live pages (those still referenced by L2P) to new locations, then erases the block.
- **Wear leveling**: ensures blocks are erased evenly by periodically moving cold data to well-worn blocks and placing new writes in lightly-worn blocks.
- **Write amplification factor (WAF)**: `WAF = NAND bytes written / host bytes written`. GC-induced writes are the main contributor. Steady-state WAF depends on write pattern and over-provisioning.

```python
from typing import Optional
import random

PAGES_PER_BLOCK = 64
NUM_BLOCKS = 256
PAGE_SIZE = 4096

class SimpleFTL:
    def __init__(self, num_lba: int = 4096,
                 pages_per_block: int = PAGES_PER_BLOCK,
                 num_blocks: int = NUM_BLOCKS):
        self.num_lba = num_lba
        self.ppb = pages_per_block
        self.num_blocks = num_blocks
        total_pages = num_blocks * pages_per_block

        self.l2p: Dict[int, Optional[int]] = {}  # LBA → PPA
        self.page_data: Dict[int, bytes] = {}     # PPA → data
        self.page_valid = [False] * total_pages   # is this page live?
        self.block_erase_count = [0] * num_blocks

        # Write pointer within current active block
        self._active_block = 0
        self._active_page_offset = 0
        self._free_pages = total_pages

        self.host_bytes_written = 0
        self.nand_bytes_written = 0

    def _ppa(self, block: int, offset: int) -> int:
        return block * self.ppb + offset

    def _block_of(self, ppa: int) -> int:
        return ppa // self.ppb

    def _alloc_page(self) -> int:
        """Find next free page via sequential log-structured allocation."""
        for _ in range(self.num_blocks):
            ppa = self._ppa(self._active_block, self._active_page_offset)
            if not self.page_valid[ppa]:
                self._active_page_offset += 1
                if self._active_page_offset >= self.ppb:
                    self._active_block = (self._active_block + 1) % self.num_blocks
                    self._active_page_offset = 0
                self._free_pages -= 1
                return ppa
            # Advance
            self._active_page_offset += 1
            if self._active_page_offset >= self.ppb:
                self._active_block = (self._active_block + 1) % self.num_blocks
                self._active_page_offset = 0
        raise RuntimeError("No free pages — run GC")

    def write(self, lba: int, data: bytes) -> None:
        assert len(data) == PAGE_SIZE
        self.host_bytes_written += PAGE_SIZE

        # Invalidate old mapping
        if lba in self.l2p and self.l2p[lba] is not None:
            old_ppa = self.l2p[lba]
            self.page_valid[old_ppa] = False

        if self._free_pages < self.num_blocks:  # threshold for GC
            self._gc()

        ppa = self._alloc_page()
        self.l2p[lba] = ppa
        self.page_data[ppa] = data
        self.page_valid[ppa] = True
        self.nand_bytes_written += PAGE_SIZE

    def read(self, lba: int) -> Optional[bytes]:
        ppa = self.l2p.get(lba)
        if ppa is None:
            return b'\xff' * PAGE_SIZE  # unwritten page reads as 0xFF in NAND
        return self.page_data.get(ppa, b'\xff' * PAGE_SIZE)

    def _gc(self) -> None:
        """Garbage collection: pick block with fewest live pages, copy live, erase."""
        # Find victim: block with lowest live page count (greedy GC)
        def live_count(b: int) -> int:
            return sum(1 for i in range(self.ppb) if self.page_valid[self._ppa(b, i)])

        victim = min(range(self.num_blocks), key=live_count)

        # Copy live pages to new locations
        for offset in range(self.ppb):
            ppa = self._ppa(victim, offset)
            if self.page_valid[ppa]:
                # Find which LBA maps here
                lba = next((k for k, v in self.l2p.items() if v == ppa), None)
                if lba is not None:
                    data = self.page_data[ppa]
                    self.page_valid[ppa] = False
                    new_ppa = self._alloc_page()
                    self.l2p[lba] = new_ppa
                    self.page_data[new_ppa] = data
                    self.page_valid[new_ppa] = True
                    self.nand_bytes_written += PAGE_SIZE  # GC write amplification

        # Erase block
        for offset in range(self.ppb):
            ppa = self._ppa(victim, offset)
            self.page_valid[ppa] = False
            if ppa in self.page_data:
                del self.page_data[ppa]
        self.block_erase_count[victim] += 1
        self._free_pages += self.ppb

    @property
    def waf(self) -> float:
        return self.nand_bytes_written / max(self.host_bytes_written, 1)

    def stats(self) -> None:
        live = sum(self.page_valid)
        total = self.num_blocks * self.ppb
        max_erase = max(self.block_erase_count)
        min_erase = min(self.block_erase_count)
        print(f"WAF:          {self.waf:.2f}x")
        print(f"Live pages:   {live}/{total}  ({100*live/total:.1f}% utilization)")
        print(f"Erase counts: min={min_erase}  max={max_erase}  "
              f"imbalance={max_erase - min_erase}")

def demo_ftl():
    ftl = SimpleFTL(num_lba=1024)
    data = b'A' * PAGE_SIZE

    # Sequential write — low WAF
    for lba in range(512):
        ftl.write(lba, data)
    print("=== After 512 sequential writes ===")
    ftl.stats()

    # Random overwrites — high WAF (GC triggered)
    for _ in range(1024):
        lba = random.randint(0, 511)
        ftl.write(lba, data)
    print("\n=== After 1024 random overwrites ===")
    ftl.stats()

demo_ftl()
```

---

## 7. Erasure Coding

RAID-5 XOR parity tolerates 1 failure. Reed-Solomon (RS) codes generalize this: given *k* data chunks and *m* parity chunks (total *n = k + m*), any *k* surviving chunks reconstruct the original data. This tolerates *m* simultaneous failures from any position.

### Galois Field Arithmetic

RS codes operate over **GF(2^8)** (256-element field). Operations:
- **Addition**: XOR (since characteristic is 2, -1 = 1, so subtraction = addition)
- **Multiplication**: using precomputed log/antilog tables to avoid polynomial long multiplication

The primitive polynomial `x^8 + x^4 + x^3 + x^2 + 1` (0x11D) generates the field.

### Systematic RS Encoding

Encode *k* data chunks into *n* chunks via multiplication by an *n×k* generator matrix (Cauchy matrix ensures any *k* rows are invertible). The first *k* rows of the generator matrix are the identity — systematic encoding means the first *k* output chunks are the original data unchanged.

```python
from typing import List, Tuple

# --- GF(2^8) arithmetic ---
GF_EXP = [0] * 512
GF_LOG = [0] * 256

def _init_gf() -> None:
    x = 1
    for i in range(255):
        GF_EXP[i] = x
        GF_LOG[x] = i
        x <<= 1
        if x & 0x100:
            x ^= 0x11d
    for i in range(255, 512):
        GF_EXP[i] = GF_EXP[i - 255]

_init_gf()

def gf_mul(a: int, b: int) -> int:
    if a == 0 or b == 0:
        return 0
    return GF_EXP[GF_LOG[a] + GF_LOG[b]]

def gf_div(a: int, b: int) -> int:
    if b == 0:
        raise ZeroDivisionError
    if a == 0:
        return 0
    return GF_EXP[(GF_LOG[a] - GF_LOG[b] + 255) % 255]

def gf_pow(x: int, power: int) -> int:
    return GF_EXP[(GF_LOG[x] * power) % 255] if x != 0 else 0

def gf_inv(x: int) -> int:
    return GF_EXP[255 - GF_LOG[x]]

# --- Matrix operations over GF(2^8) ---
Matrix = List[List[int]]

def mat_mul(A: Matrix, B: Matrix) -> Matrix:
    rows_A, cols_A = len(A), len(A[0])
    cols_B = len(B[0])
    C = [[0] * cols_B for _ in range(rows_A)]
    for i in range(rows_A):
        for j in range(cols_B):
            for l in range(cols_A):
                C[i][j] ^= gf_mul(A[i][l], B[l][j])
    return C

def mat_inv(M: Matrix) -> Matrix:
    """Gaussian elimination over GF(2^8)."""
    n = len(M)
    # Augment with identity
    aug = [M[i][:] + [1 if i == j else 0 for j in range(n)] for i in range(n)]
    for col in range(n):
        # Find pivot
        pivot = next((r for r in range(col, n) if aug[r][col] != 0), None)
        if pivot is None:
            raise ValueError("Matrix is singular over GF(2^8)")
        aug[col], aug[pivot] = aug[pivot], aug[col]
        inv_pivot = gf_inv(aug[col][col])
        aug[col] = [gf_mul(v, inv_pivot) for v in aug[col]]
        for row in range(n):
            if row != col and aug[row][col] != 0:
                factor = aug[row][col]
                aug[row] = [aug[row][i] ^ gf_mul(factor, aug[col][i]) for i in range(2 * n)]
    return [row[n:] for row in aug]

# --- Cauchy Reed-Solomon ---
class ReedSolomon:
    def __init__(self, k: int, n: int):
        assert n <= 256 and k < n
        self.k = k
        self.n = n
        self.enc_matrix = self._build_cauchy_matrix()

    def _build_cauchy_matrix(self) -> Matrix:
        """Build an n×k Cauchy matrix. Any k rows are guaranteed invertible."""
        # x_i = i for i in 0..n-1, y_j = n + j for j in 0..k-1
        # Cauchy[i][j] = 1 / (x_i XOR y_j) in GF(2^8)
        # For systematic encoding: first k rows = identity
        x = list(range(self.n))
        y = [self.n + j for j in range(self.k)]
        matrix = []
        for i in range(self.n):
            row = []
            for j in range(self.k):
                if i < self.k:
                    row.append(1 if i == j else 0)
                else:
                    row.append(gf_inv(x[i] ^ y[j]))
            matrix.append(row)
        return matrix

    def encode(self, data_chunks: List[bytes]) -> List[bytes]:
        assert len(data_chunks) == self.k
        chunk_len = len(data_chunks[0])
        output = [bytearray(chunk_len) for _ in range(self.n)]
        for byte_idx in range(chunk_len):
            data_vec = [data_chunks[j][byte_idx] for j in range(self.k)]
            for i in range(self.n):
                val = 0
                for j in range(self.k):
                    val ^= gf_mul(self.enc_matrix[i][j], data_vec[j])
                output[i][byte_idx] = val
        return [bytes(c) for c in output]

    def decode(self, chunks: List[Optional[bytes]], chunk_indices: List[int]) -> List[bytes]:
        """Recover k data chunks from any k available chunks."""
        assert len(chunks) == self.k and len(chunk_indices) == self.k
        chunk_len = len(chunks[0])

        # Extract the k rows of enc_matrix corresponding to available indices
        sub_matrix = [self.enc_matrix[i] for i in chunk_indices]
        inv_matrix = mat_inv(sub_matrix)

        output = [bytearray(chunk_len) for _ in range(self.k)]
        for byte_idx in range(chunk_len):
            recv_vec = [chunks[j][byte_idx] for j in range(self.k)]
            for i in range(self.k):
                val = 0
                for j in range(self.k):
                    val ^= gf_mul(inv_matrix[i][j], recv_vec[j])
                output[i][byte_idx] = val
        return [bytes(c) for c in output]

def demo_rs():
    print("=== Reed-Solomon RS(4, 6) demo ===")
    rs = ReedSolomon(k=4, n=6)  # 4 data, 6 total (2 parity) — tolerates 2 failures

    # Encode
    data = [bytes([i + 1] * 8) for i in range(4)]
    print(f"Original data chunks: {[list(d) for d in data]}")
    encoded = rs.encode(data)
    print(f"Encoded (6 chunks):   {[list(c) for c in encoded]}")

    # Corrupt chunks 0 and 3 (simulate 2 drive failures)
    available = [encoded[1], encoded[2], encoded[4], encoded[5]]
    indices   = [1, 2, 4, 5]
    recovered = rs.decode(available, indices)
    print(f"Recovered data:       {[list(r) for r in recovered]}")
    print(f"Recovery correct:     {recovered == data}")

demo_rs()
```

---

## 8. What Textbooks Don't Tell You

### fsync() Is Not Enough

`fsync(fd)` flushes the file's dirty pages and the journal commit for that file's metadata. But the **directory entry** pointing to the file is a separate metadata structure. A crash between `fsync(fd)` and `fsync(dirfd)` can leave the file's data and inode intact on disk but with no directory entry pointing to it — the file is orphaned (recoverable by `fsck`, but not automatically).

The safe pattern for atomic file replacement:

```python
import os, tempfile

def safe_write(path: str, data: bytes) -> None:
    """Write atomically: write to tmp, fsync, rename. fsync parent dir."""
    dirpath = os.path.dirname(os.path.abspath(path))
    fd, tmp_path = tempfile.mkstemp(dir=dirpath)
    try:
        os.write(fd, data)
        os.fsync(fd)       # flush file data + inode to disk
        os.close(fd)
        os.rename(tmp_path, path)  # atomic on POSIX (same filesystem)
        # fsync the directory to persist the rename
        dir_fd = os.open(dirpath, os.O_RDONLY)
        try:
            os.fsync(dir_fd)
        finally:
            os.close(dir_fd)
    except Exception:
        os.close(fd)
        os.unlink(tmp_path)
        raise
```

### The RAID-5 Write Hole

Updating one data block in a RAID-5 stripe requires four I/Os: read old data, read old parity, write new data, write new parity (read-modify-write). If power fails between writing new data and writing new parity, the stripe is inconsistent. On restart, RAID cannot know which blocks are new and which are old without a journal.

ZFS RAID-Z is immune because CoW means partial stripe writes never happen — the entire new stripe is written to new blocks.

### SSD Burst vs. Sustained Write Performance

Consumer SSDs have DRAM caches (256 MB–4 GB) and SLC write caches (a portion of TLC NAND operated in SLC mode). Burst writes go to these caches at 3–5 GB/s. Once the cache fills (sustained writes), performance falls to the TLC native write speed: 200–800 MB/s for sequential, much worse for random. Always benchmark with data sets larger than the SSD cache for production workload characterization.

### Inode Exhaustion

`df -h` shows 30% space used but the filesystem refuses new files? Run `df -i`. ext4 creates a fixed number of inodes at `mkfs` time (default ratio: 1 inode per 16 KB). A workload creating millions of 1-byte files exhausts inodes while leaving most disk blocks free.

Fix at mkfs time: `mkfs.ext4 -i 4096 /dev/sdX` (1 inode per 4 KB). Or use a filesystem without fixed inode tables: btrfs (dynamic inode allocation), xfs (also dynamic).

### ZFS ARC and the Linux Page Cache

On Linux, ZFS (OpenZFS) bypasses the kernel's VFS page cache for data blocks and manages its own cache (ARC) in kernel memory. But there is a subtle interaction: when accessed via mmap or certain VFS paths, data can appear in *both* the ARC and the page cache — double buffering that wastes RAM. For database workloads on ZFS-on-Linux, set `primarycache=metadata` on the dataset and use `O_DIRECT` from the application to prevent this.

---

## 9. Hard Exercises

### (a) Inode Exhaustion from Log Writes

A Python service writing 1 KB entries at 50,000/second to a single file should not exhaust inode count — that's one file, one inode. The symptom (filesystem reports 0% free, `df -h` shows 30% used) points to **deleted-but-open files**.

ext4 (and all Linux filesystems) allow unlinking a file while a process holds it open. The file's blocks are not freed until the last file descriptor is closed. If the logging service rotates logs by renaming the file and reopening, but a previous process still holds the old descriptor, the old file's 4 TB of data remains allocated. The kernel tracks this as "used blocks" even though `ls` cannot find the file.

Diagnosis: `lsof | grep deleted` or `ls -la /proc/<pid>/fd/` to find open file descriptors pointing to deleted inodes. Fix: explicitly close old log file descriptors before rotation, or use `logrotate` with `copytruncate` to truncate in-place.

### (b) Delayed Allocation Risk

```python
import os, sys, signal, time

# Write data without fsync, then demonstrate loss via delayed allocation
PATH = '/tmp/delayed_alloc_test'

def demonstrate_delayed_alloc():
    fd = os.open(PATH, os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o644)
    
    # Write 1 MB of data — buffered in page cache, NOT on disk yet
    payload = b'IMPORTANT DATA: This should persist\n' * 30_000
    n = os.write(fd, payload)
    print(f"os.write returned {n} bytes written — data is in page cache")
    print(f"File size (before fsync): {os.stat(PATH).st_size} bytes")
    
    # With delayed allocation, ext4 has not allocated blocks yet.
    # A crash here (or kill -9 before dirty page writeback) loses data.
    print("Sleeping 1s — in production this is where delayed alloc bites you...")
    time.sleep(1)
    
    # Safe version: call fsync before considering data durable
    os.fsync(fd)
    print("fsync() complete — data is now on disk")
    os.close(fd)

    # Verify
    with open(PATH, 'rb') as f:
        data = f.read()
    print(f"File contents confirmed: {len(data)} bytes")
    os.unlink(PATH)

demonstrate_delayed_alloc()
```

The risk: a process calls `write()` (returns success), then crashes before the 30-second dirty page writeback. ext4 with `delalloc` has not yet allocated physical blocks. The data is in the page cache only — and the page cache is in RAM. On crash, it is gone. The `fsync()` call forces allocation and writeback.

### (c) ZFS RAID-Z1 Capacity and Wear

With 4 × 4TB drives in RAID-Z1:
- Usable capacity: `(4 - 1) × 4TB = 12 TB` (1 drive is parity)
- ZFS also subtracts ~1–2% for internal metadata → approximately 11.5 TB usable

ZFS does **not** automatically migrate data from worn drives. RAID-Z has no wear leveling at the pool level — each individual drive's FTL handles its own wear leveling internally. ZFS does not know or control which physical blocks a drive's FTL places writes on.

Check wear status:
```bash
# NVMe drives: show Percentage Used (0=new, 100=worn out)
nvme smart-log /dev/nvme0 | grep percentage_used

# For all pool devices:
zpool status -v tank        # show error counts
smartctl -a /dev/sda        # SMART attributes including SSD wear
```

With one drive at 60% remaining wear and you have 3 years of writes, the healthy action is to **replace the worn drive proactively** before it fails:
```bash
zpool replace tank /dev/sdb /dev/sdd  # replace old with new drive
zpool scrub tank                       # verify all data after resilver
```

### (d) Erasure Coding for 1 PB with 40% Overhead and 2-Hour Reconstruction

Requirements:
- Tolerate 2 simultaneous node failures
- Storage overhead ≤ 40% → `n/k ≤ 1.4` → `m/k ≤ 0.4` → with `m = 2`: `k ≥ 5`
- Reconstruction time ≤ 2 hours per failed node

**RS(10, 12)** satisfies these:
- `k=10`, `m=2` → overhead = `2/10 = 20%` ✓ (well under 40%)
- Tolerates any 2 simultaneous failures ✓
- Effective storage: 1 PB / 1.2 = 833 TB of raw data across 10 data nodes

Reconstruction bandwidth calculation:
- Each data node stores 1 PB / 10 = 100 TB
- To reconstruct 1 failed node (100 TB of data), must read any 10 of 11 surviving nodes: `10 × 100 TB = 1,000 TB = 1 PB` of network reads
- Required in 2 hours: `1 PB / 7200 s = 142 GB/s` aggregate network bandwidth
- Per surviving node: `142 GB/s / 10 = 14.2 GB/s` each

In practice, use **minimum storage regenerating codes (MSRC)** (Dimakis et al. 2010) to reduce reconstruction bandwidth: instead of reading full chunks, each surviving node sends `chunk_size / (k)` of data. For RS(10,12) this reduces reconstruction bandwidth from 1 PB to ~182 TB — a 5.5× improvement.

### (e) Benchmark: fsync vs fdatasync vs no-sync

```python
import os, time, tempfile, statistics

def benchmark_sync(mode: str, iterations: int = 100, size: int = 4096) -> float:
    """Measure per-write latency for different sync modes."""
    data = b'x' * size
    fd, path = tempfile.mkstemp()
    try:
        latencies = []
        for _ in range(iterations):
            t = time.perf_counter()
            os.write(fd, data)
            if mode == 'fsync':
                os.fsync(fd)
            elif mode == 'fdatasync':
                os.fdatasync(fd)
            # 'none': no sync — data may be lost on crash
            latencies.append((time.perf_counter() - t) * 1e6)
        median_us = statistics.median(latencies)
        p99_us = sorted(latencies)[int(0.99 * iterations)]
        print(f"{mode:12s}  median={median_us:7.1f} µs  p99={p99_us:7.1f} µs  "
              f"throughput={size * iterations / sum(latencies) * 1e6 / 1e6:.1f} MB/s")
        return median_us
    finally:
        os.close(fd)
        os.unlink(path)

print("=== Sync mode benchmark (4KB writes) ===")
print(f"{'mode':12s}  {'median':>12s}  {'p99':>12s}  {'throughput':>15s}")
none_us   = benchmark_sync('none')
fdata_us  = benchmark_sync('fdatasync')
fsync_us  = benchmark_sync('fsync')

print()
print("When to use each:")
print(f"  none:      Max throughput. Data lost on crash. OK for caches, temp files.")
print(f"  fdatasync: Flushes data + minimal metadata (size). No atime/mtime flush.")
print(f"             Use for: log appends (size is the critical metadata).")
print(f"  fsync:     Flushes data + all metadata (mtime, permissions). Slower.")
print(f"             Use for: atomic file writes, database WAL commits.")
print()
print(f"  fdatasync vs fsync overhead: {(fsync_us - fdata_us)/fdata_us*100:.1f}% "
      f"({'fsync is slower by' if fsync_us > fdata_us else 'fsync is faster by'} "
      f"{abs(fsync_us - fdata_us):.1f} µs)")
```

**When each is appropriate:**

- **No sync**: Acceptable for scratch files, reconstructible caches, and data that can be regenerated. On crash, anything not written back is lost — including data `write()` returned success for.
- **`fdatasync()`**: Flushes file data and the file size to disk. Does not flush `mtime`, `atime`, or other non-critical metadata. Use when you care about data integrity but not metadata consistency (log appending: the size is critical, the mtime is not).
- **`fsync()`**: Flushes everything — data, all metadata, and the journal commit record. Required when correctness depends on all metadata being durable (database WAL commits, atomic rename sequences, anything where you query metadata post-crash).

On modern NVMe with power-loss protection (PLP) capacitors, `fdatasync()` latency is typically 50–200 µs. On SATA SSD without PLP (consumer drives), the drive may acknowledge the flush without actually reaching persistent storage — making "fsync" latency look fast (the drive lies) while providing no real durability guarantee. Check SMART attribute 233 (Media Wearout) and the drive's PLP specification.

---

## Summary: Key Invariants

| Property | ext4 | ZFS | NVMe SSD |
|----------|------|-----|----------|
| Write ordering | Journal | CoW + TXG | FTL (opaque) |
| Corruption detection | None (silent) | Per-block SHA-256 | ECC (single-bit) |
| Snapshot | LVM external | Built-in, O(1) | N/A |
| Max file size | 16 TB (48-bit extents) | 2^64 bytes | N/A |
| Inode limit | Fixed at mkfs | Dynamic | N/A |
| RAID rebuild reads all data | Yes | Yes | N/A |
| Tolerates write hole | No (journal) | Yes (CoW) | N/A |

The storage stack is a tower of abstractions each solving one failure mode of the layer below. ext4 solves MINIX fragmentation and uses a journal for consistency. ZFS eliminates the journal and write hole via CoW. NVMe eliminates the queue bottleneck. Erasure coding eliminates single-point-of-failure from RAID. Understanding each layer's failure model is the only way to build storage systems that are both fast and correct.

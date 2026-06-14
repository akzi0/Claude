# Lesson 02: Database Internals — B+ Trees, LSM Trees, MVCC, and WAL

> **Audience**: Engineers who have used PostgreSQL or MySQL in production but have never read the source code or a storage engine paper. This lesson operates at graduate seminar depth. By the end you will be able to reason about query latency, write amplification, vacuum tuning, and crash recovery from first principles.

---

## Table of Contents

1. [B+ Tree Internals](#1-b-tree-internals) — node structure, insertion with worked example, deletion, bulk loading, write amplification
2. [LSM Trees](#2-lsm-trees) — MemTable, SSTable format, compaction strategies, read/write amplification, write stalls
3. [Python Implementation: B+ Tree](#3-python-implementation-b-tree) — fully working insert, search, range_scan with tests
4. [Python Implementation: Bloom Filter + SkipList](#4-python-implementation-bloom-filter--skiplist) — LSM building blocks
5. [MVCC — Multi-Version Concurrency Control](#5-mvcc--multi-version-concurrency-control) — visibility rules, snapshot isolation, write skew, SSI, VACUUM, HOT, wraparound
6. [Write-Ahead Logging (WAL)](#6-write-ahead-logging-wal) — golden rule, record structure, ARIES algorithm, checkpointing, FPIs
7. [Buffer Pool Management](#7-buffer-pool-management) — LRU, clock algorithm, ARC, dirty page eviction
8. [Index Types Beyond B+ Tree](#8-index-types-beyond-b-tree) — Hash, GIN, GiST, BRIN
9. [What Textbooks Don't Tell You](#9-what-textbooks-dont-tell-you) — production realities
10. [Edge Cases and Failure Modes](#10-edge-cases-and-failure-modes) — torn writes, compaction debt, xid wraparound
11. [Production Monitoring Playbook](#11-production-monitoring-playbook) — SQL queries to diagnose common problems
12. [Hard Exercise: MVCC Under Load](#12-hard-exercise-mvcc-under-load)
13. [Summary and Decision Tree](#13-summary-and-decision-tree)

---

## 1. B+ Tree Internals

### 1.1 Why Trees at All?

A database index must support three operations efficiently: exact-key lookup, range scan, and ordered iteration. A hash map gives O(1) lookup but zero range support. A sorted array gives O(log n) lookup and O(1) range scan but O(n) inserts. A balanced BST (AVL, red-black) gives O(log n) for all three — but is useless on disk because each node is a separate pointer, and a lookup on a 1-billion-row table requires ~30 node visits, each potentially a separate random I/O.

The breakthrough insight behind B-trees (Bayer & McCreight, 1972): make each node large — large enough to fill one disk page (4 KB or 8 KB). Pack hundreds of keys into one node. Now a lookup on a 1-billion-row table requires only 3–4 disk reads.

### 1.2 Node Structure

**Internal nodes** store separator keys and child pointers. For a node of order `d` (maximum `2d` keys):

```
[ ptr_0 | key_1 | ptr_1 | key_2 | ptr_2 | ... | key_{2d} | ptr_{2d} ]
```

All keys in the subtree rooted at `ptr_i` satisfy: `key_i ≤ k < key_{i+1}`.

**Leaf nodes** store actual key-value pairs and a pointer to the next leaf:

```
[ (key_1, val_1) | (key_2, val_2) | ... | (key_{2d}, val_{2d}) | next_leaf_ptr ]
```

This sibling pointer is the critical difference from a plain B-tree. It enables range scans without returning to the root.

**Branching factor**: If keys are 8 bytes and pointers are 8 bytes, an internal node on a 4 KB page holds roughly `4096 / 16 ≈ 256` entries — a branching factor of ~256. A tree of height 3 can index `256³ ≈ 16 million` entries with at most 3 I/Os per lookup. Height 4 reaches 4 billion rows.

PostgreSQL uses 8 KB pages. InnoDB uses 16 KB by default. Tuning page size is the first lever for I/O reduction.

### 1.3 Insertion Algorithm

```
insert(key, value):
    leaf = find_leaf(key)          # traverse from root, O(log n) I/Os
    insert (key,value) into leaf
    if leaf.num_keys > ORDER - 1:
        split_leaf(leaf)

split_leaf(node):
    mid = len(node.keys) // 2
    new_leaf = BPlusNode(is_leaf=True)
    new_leaf.keys = node.keys[mid:]
    new_leaf.values = node.values[mid:]
    new_leaf.next_leaf = node.next_leaf
    node.keys = node.keys[:mid]
    node.values = node.values[:mid]
    node.next_leaf = new_leaf
    promote key = new_leaf.keys[0] up to parent
    if parent overflows: split_internal(parent)  # recursive
```

**Worked example**: Insert keys `[3, 7, 1, 8, 2, 9, 4, 6, 5]` into an order-2 B+ tree (max 2 keys per leaf before split, max 2 separator keys per internal node before split → max 3 children per internal node).

> **Reading the diagrams**: Internal nodes shown as `[key1, key2]`. Leaf nodes shown as `{k1,k2}`. Arrows `→` show leaf sibling pointers (the range-scan chain).

```
Step 1 — Insert 3:
  Single leaf: {3}

Step 2 — Insert 7:
  {3, 7}

Step 3 — Insert 1:
  Leaf {1,3,7} overflows (max=2). Split at mid=1:
    left={1,3},  right={7}.  Promote separator=7 to new root.

                [7]
               /   \
            {1,3}→{7}

Step 4 — Insert 8:
  8 ≥ 7 → right leaf. {7,8}.

                [7]
               /   \
            {1,3}→{7,8}

Step 5 — Insert 2:
  2 < 7 → left leaf {1,2,3} — overflow! Split: left={1,2}, right={3}. Promote 3 to parent.
  Parent [3,7] now has 2 keys, 3 children — OK (max=2 keys).

                [3, 7]
               /  |   \
           {1,2}→{3}→{7,8}

Step 6 — Insert 9:
  9 ≥ 7 → rightmost leaf {7,8,9} — overflow! Split: left={7,8}, right={9}. Promote 9.
  Parent [3,7,9] — overflow (max=2 keys)! Split internal node.
    mid key = 7 (index 1), promote 7 to new root.
    left-internal=[3], right-internal=[9]

                      [7]
                    /     \
                 [3]       [9]
                /   \     /   \
           {1,2}→{3}→{7,8}→{9}

Step 7 — Insert 4:
  4 < 7 → left subtree. 4 ≥ 3 → second child of [3] → leaf {3}. Now {3,4}.

                      [7]
                    /     \
                 [3]       [9]
                /   \     /   \
           {1,2}→{3,4}→{7,8}→{9}

Step 8 — Insert 6:
  6 < 7 → left subtree. 6 ≥ 3 → second child of [3] → leaf {3,4,6} — overflow!
  Split: left={3,4}, right={6}. Promote 6 to parent [3].
  Parent [3] becomes [3,6] — 2 keys, OK.

                      [7]
                    /     \
                [3,6]       [9]
               / |  \      /   \
          {1,2}→{3,4}→{6}→{7,8}→{9}

Step 9 — Insert 5:
  5 < 7 → left subtree. 5 ≥ 3, 5 < 6 → middle child of [3,6] → leaf {3,4,5} — overflow!
  Split: left={3,4}, right={5}. Promote 5 to parent [3,6].
  Parent [3,5,6] — overflow (3 keys > max 2)!
  Split internal [3,5,6]: mid=5, promote 5 to root.
    left-internal=[3], right-internal=[6]
  Root [7] becomes [5,7] — 2 keys, OK.

                        [5, 7]
                      /   |    \
                   [3]   [6]   [9]
                  /   \  / \   / \
             {1,2}→{3,4}→{5}→{6}→{7,8}→{9}
```

**Final tree state** (height=3):
- Root: `[5, 7]`
- Level 1 internals: `[3]`, `[6]`, `[9]`
- Leaves (linked list): `{1,2} → {3,4} → {5} → {6} → {7,8} → {9}`

A range scan for keys 3–7 descends once to leaf `{3,4}` then walks the sibling pointers: `{3,4}`, `{5}`, `{6}`, `{7,8}` — 4 leaf reads for 6 results, no tree re-traversal.

### 1.4 Deletion

Deletion is the inverse:

1. Find and remove the key from the leaf.
2. If the leaf is now **underfull** (fewer than `⌈ORDER/2⌉ - 1` keys):
   - **Borrow from sibling**: if adjacent sibling has a spare key, rotate it through the parent separator.
   - **Merge with sibling**: if no sibling can spare, merge the two leaves. Remove the separator key from the parent. If the parent is now underfull, apply borrow/merge recursively up the tree.
3. If the root has zero keys after a merge, the root is deleted and the tree shrinks in height.

The worst case is O(log n) merges propagating all the way to the root — the mirror of a cascading split during insertion.

### 1.5 Why B+ Trees, Not B-Trees?

In a B-tree, internal nodes store actual data (key-value pairs), not just separator keys. This wastes internal node space on payload bytes that could instead be used for more separator keys, reducing branching factor. More critically, **range scans** on a B-tree require returning to the tree for each key outside the current node — there are no sibling pointers at internal levels.

In a B+ tree: all data lives in leaves, internal nodes are pure routing. Leaves are a sorted linked list. A range scan finds the start leaf in O(log n) then follows sibling pointers in O(k) where k is the result size. This is why every production RDBMS uses B+ trees (with minor variations like PostgreSQL's use of a high-key in each page for concurrent access).

### 1.6 Bulk Loading

Inserting n keys one by one costs O(n log n). Bulk loading from sorted input costs O(n):

1. Sort the input (or assume it's already sorted).
2. Fill leaf nodes to a target occupancy (e.g., 70%) left to right.
3. As each leaf fills, emit the first key of the new leaf as a separator key.
4. Build internal nodes bottom-up from the separator keys.
5. Build the root last.

Bulk loading is 2–3× faster because:
- No split cascades: every write is sequential.
- Leaves are written in order → I/O is sequential.
- Target occupancy below 100% leaves room for future inserts without immediate splits.

PostgreSQL's `CREATE INDEX` uses bulk loading (sort + bottom-up build), not repeated inserts.

### 1.7 Write Amplification in B+ Trees

A single logical key insert can dirty multiple physical pages:

- The target leaf page.
- The parent internal page (split separator).
- The grandparent (if split cascades).
- The sibling leaf (sibling pointer update).

On a busy OLTP system writing 10,000 rows/sec with 1% leaf split rate, the actual page writes might be 3–5× the logical write count. PostgreSQL's `fillfactor` storage parameter (default 90 for tables, 90 for indexes) pre-allocates free space in each page so updates-in-place don't trigger splits, reducing write amplification.

**Comparison to LSM**: LSM trades read amplification for write amplification. B+ trees trade write amplification for read amplification. The wrong choice for your workload is expensive.

### 1.8 Concurrent B+ Tree Access: The B-Link Tree Protocol

A naive B+ tree with a single reader-writer lock scales to one CPU. PostgreSQL's index implementation uses the **B-link tree** (Lehman & Yao, 1981) to allow concurrent readers and writers with minimal locking:

**Three invariants** added to the standard B+ tree:
1. Every node has a **high-key** — the maximum key that belongs in this subtree.
2. Every node has a **right-sibling pointer** (not just leaves — internal nodes too).
3. A node being split keeps both halves accessible via the sibling pointer until the parent is updated.

**Why this helps**: Without B-link, splitting a leaf requires holding a write lock on the leaf AND its parent simultaneously (to atomically insert the separator key). Under concurrent writes, this causes lock escalation.

With B-link:
1. Writer acquires lock on the leaf being split.
2. Creates the new right sibling with the upper half of keys.
3. Sets `right_sibling` on the original leaf to point to the new sibling.
4. Releases the leaf lock.
5. Acquires the parent lock and inserts the separator key.

A reader that arrived between steps 3 and 5 finds the original leaf, sees `right_sibling` is set, follows it, and finds the correct key range — without needing a lock on the parent. This "link following" is the key insight.

PostgreSQL's actual implementation in `src/backend/access/nbtree/` uses:
- `LockBufferForCleanup` for exclusive access during splits.
- `ReadBuffer` + `LockBuffer(SHARE)` for concurrent reads.
- `BTPageOpaque` structure in each page stores `btpo_next` (right-sibling), `btpo_prev`, and `btpo_flags` (leaf/internal/root/deleted).

---

## 2. LSM Trees

### 2.1 Motivation

B+ trees require random writes: insert key 42 → find its leaf page anywhere on disk → write. On spinning disks, random writes are limited by seek time (~5ms per seek → ~200 IOPS). Sequential writes are limited by transfer speed (~100 MB/s → 25,000 4KB-page writes/sec). The gap is 100×.

Log-Structured Merge-Trees (O'Neil et al., 1996) convert random writes into sequential writes by batching all writes in memory and flushing as sequential files. The cost is paid during compaction (background merge sort).

### 2.2 Architecture

```
Write path:
  Client write → WAL (durable) → MemTable (in-memory, sorted)
                                      ↓ (when full, ~64MB)
                                  L0 SSTable (immutable file)
                                      ↓ (compaction)
                                  L1 SSTables
                                      ↓
                                  L2 SSTables
                                      ...
```

**MemTable**: A sorted in-memory data structure — typically a skip list (O(log n) insert/search) or red-black tree. RocksDB uses skip lists by default. When the MemTable exceeds a size threshold (e.g., 64 MB), it is frozen (becomes an immutable MemTable) and a new active MemTable is created. A background thread flushes the immutable MemTable to disk as an L0 SSTable.

**WAL**: Every write goes to the WAL before the MemTable. On crash, replay the WAL to reconstruct the MemTable. Without WAL, an in-memory MemTable is volatile.

### 2.3 SSTable Format

An SSTable (Sorted String Table) is an immutable, sorted file:

```
┌────────────────────────────────────┐
│  Data Blocks (sorted key-value)    │  ← actual data, compressed
│  ...                               │
├────────────────────────────────────┤
│  Index Block                       │  ← one entry per data block: (last_key, offset)
├────────────────────────────────────┤
│  Bloom Filter Block                │  ← probabilistic membership test
├────────────────────────────────────┤
│  Meta Block (compression, crc)     │
├────────────────────────────────────┤
│  Footer (index block offset, magic)│
└────────────────────────────────────┘
```

**Bloom filters** are critical for read performance. A point lookup for key `k` must check: MemTable → every L0 SSTable → L1 → L2 → ... Without bloom filters, each level requires a binary search in the index block then a disk read of a data block. For a missing key, this reads every level. With a bloom filter (1% false positive rate), 99% of missing-key lookups skip the disk read entirely. A 1% FPR bloom filter uses ~9.6 bits per key — tiny compared to the data.

### 2.4 Compaction: Leveled vs. Size-Tiered

**Leveled compaction** (RocksDB default, also available in Cassandra):

- L1, L2, ..., Ln each have a size limit: `size(L_{i+1}) = 10 × size(L_i)`.
- **Key invariant within each level (except L0)**: no two files share a key range overlap.
- When L_i exceeds its size limit, pick one file from L_i and merge-sort it into all overlapping files in L_{i+1}.
- L0 is special: files may overlap (they come directly from MemTable flushes). Reads must check all L0 files.
- Write amplification per level: data written to L_i may be rewritten into L_{i+1} up to 10 times (since L_{i+1} is 10× larger, one file from L_i touches at most 10 files in L_{i+1}).
- Total WA with 7 levels: approximately `10 × 6 = 60`. A byte written once may be physically rewritten 60 times.

**Size-tiered compaction** (Cassandra default):

- Group SSTables by similar size. When enough SSTables of similar size accumulate (e.g., 4), merge them into one larger SSTable.
- Lower write amplification (~10–30), but higher space amplification (SSTables of varying ages can overlap, so a point lookup must check many files).
- Better for write-heavy workloads. Worse for read performance and space usage.

**FIFO compaction** (RocksDB option):

- Never compact; just delete oldest files when size limit reached.
- Only appropriate for time-series data with TTL where old data is discarded.

### 2.5 Write and Space Amplification

**Write amplification (WA)**: Total bytes written to disk / bytes of logical data written.

- Leveled LSM: WA ≈ `10 × num_levels`. With 7 levels, WA ≈ 60.
- B+ tree: WA ≈ 3–10 under write workloads with splits and page dirtying.
- Paradox: LSM has higher WA than B+ tree for the same data. The advantage of LSM is that all writes are **sequential**, converting IOPS-bound writes (B+ tree random) to throughput-bound writes (LSM sequential). On SSDs where random write performance is much better, this advantage shrinks.

**Space amplification (SA)**: Disk bytes used / bytes of logical data.

- During compaction, old files and new merged file coexist: temporary SA = 2×.
- Size-tiered compaction can have SA > 2× because many SSTables contain overlapping key versions.
- Leveled compaction has SA ≈ 1.1× at stable state because L_n dominates and has no overlaps.

### 2.6 Read Path

```
Point lookup(key):
    check MemTable (O(log n))
    check immutable MemTable if exists
    for each L0 SSTable (newest first):
        check bloom filter → if positive: binary search index → read data block
    for L1, L2, ...:
        at most 1 file per level can contain key (no overlaps)
        check bloom filter → if positive: binary search index → read data block
    return NotFound
```

Read amplification = number of SSTables checked per query. With bloom filters, most reads terminate at the MemTable or L0. Worst case for a hot missing key: check all levels.

Range scan is worse than point lookup because bloom filters don't help (you must scan every level's files that overlap the range). B+ trees win decisively on range scans.

### 2.7 Compaction Debt and Write Stalls

If the application writes faster than the background compaction thread can process, L0 file count grows. More L0 files → more files to check per read (no bloom filter shortcut at L0 since files overlap) → reads degrade.

RocksDB write stall mechanism:
- `level0_slowdown_writes_trigger` (default 20): slow down writes when L0 has 20+ files.
- `level0_stop_writes_trigger` (default 36): stop writes entirely when L0 has 36+ files.

Tuning compaction threads (`max_background_compactions`, `max_background_jobs`) is critical for write-heavy production workloads.

### 2.8 LSM vs B+ Tree: Amplification Comparison

```
                     Write Amp    Read Amp (point)    Space Amp
────────────────────────────────────────────────────────────────
B+ Tree (HDD)           3–10x         1–2x             ~1.1x
B+ Tree (NVMe)          3–10x         1–2x             ~1.1x
LSM Leveled             10–60x        ~5–15x           ~1.1x
LSM Size-Tiered          5–30x        ~10–30x          ~2–3x
────────────────────────────────────────────────────────────────
```

**Read amplification**: Number of I/Os per point lookup in the worst case. B+ tree: 3–4 I/Os (tree height). LSM: MemTable (0 I/Os if hit) + up to 1 SSTable per level × (1 bloom check + 1 data block read if positive). With 7 levels: up to 7 data block reads + bloom filter checks.

**Key RocksDB options** for tuning amplification:

```python
# RocksDB options (pseudocode — actual API uses Options objects)
options = {
    # Bloom filter bits per key — higher bits → lower FPR → fewer disk reads
    "bloom_filter_bits_per_key": 10,       # 1% FPR; default is 10

    # MemTable size — larger → fewer L0 files per flush → less L0 compaction
    "write_buffer_size": 128 * 1024**2,    # 128 MB (default 64 MB)

    # Max number of MemTables being built (parallelism)
    "max_write_buffer_number": 4,

    # Level multiplier (default 10)
    "max_bytes_for_level_multiplier": 10,

    # Compression per level (less CPU on hot levels, more on cold)
    "compression_per_level": [
        "kNoCompression",    # L0: fast
        "kNoCompression",    # L1: fast
        "kLZ4Compression",   # L2+: good ratio/speed
        "kLZ4Compression",
        "kZstdCompression",  # deep levels: best ratio
        "kZstdCompression",
        "kZstdCompression",
    ],
}
```

---

## 3. Python Implementation: B+ Tree

```python
from dataclasses import dataclass, field
from typing import Optional, List, Any
import random

ORDER = 4  # max children per internal node; max keys = ORDER - 1 = 3

@dataclass
class BPlusNode:
    keys: List[Any] = field(default_factory=list)
    children: List['BPlusNode'] = field(default_factory=list)
    values: List[Any] = field(default_factory=list)  # only populated in leaf
    is_leaf: bool = True
    next_leaf: Optional['BPlusNode'] = None


class BPlusTree:
    def __init__(self):
        self.root = BPlusNode(is_leaf=True)

    # ------------------------------------------------------------------ search

    def search(self, key) -> Optional[Any]:
        leaf = self._find_leaf(self.root, key)
        for i, k in enumerate(leaf.keys):
            if k == key:
                return leaf.values[i]
        return None

    # ---------------------------------------------------------------- insert

    def insert(self, key, value):
        # Returns (promoted_key, new_right_node) if root split occurred, else None
        result = self._insert(self.root, key, value)
        if result is not None:
            promoted_key, new_right = result
            new_root = BPlusNode(is_leaf=False)
            new_root.keys = [promoted_key]
            new_root.children = [self.root, new_right]
            self.root = new_root

    def _insert(self, node: BPlusNode, key, value):
        """Returns (promoted_key, new_right_node) on overflow, else None."""
        if node.is_leaf:
            # Insert in sorted order
            idx = self._find_insert_pos(node.keys, key)
            node.keys.insert(idx, key)
            node.values.insert(idx, value)
            if len(node.keys) >= ORDER:
                return self._split_leaf(node)
            return None
        else:
            # Find child to descend into
            child_idx = self._find_child_idx(node.keys, key)
            child = node.children[child_idx]
            result = self._insert(child, key, value)
            if result is not None:
                promoted_key, new_right = result
                # Insert promoted key and new child into this internal node
                insert_pos = self._find_insert_pos(node.keys, promoted_key)
                node.keys.insert(insert_pos, promoted_key)
                node.children.insert(insert_pos + 1, new_right)
                if len(node.keys) >= ORDER:
                    return self._split_internal(node)
            return None

    # --------------------------------------------------------------- range scan

    def range_scan(self, start, end) -> List[Any]:
        """Returns all values where start <= key <= end, in sorted order."""
        results = []
        leaf = self._find_leaf(self.root, start)
        while leaf is not None:
            for i, k in enumerate(leaf.keys):
                if k > end:
                    return results
                if k >= start:
                    results.append((k, leaf.values[i]))
            leaf = leaf.next_leaf
        return results

    # ---------------------------------------------------------- internal helpers

    def _find_leaf(self, node: BPlusNode, key) -> BPlusNode:
        if node.is_leaf:
            return node
        child_idx = self._find_child_idx(node.keys, key)
        return self._find_leaf(node.children[child_idx], key)

    def _find_child_idx(self, keys, key) -> int:
        """Return index of child subtree that should contain key."""
        for i, k in enumerate(keys):
            if key < k:
                return i
        return len(keys)

    def _find_insert_pos(self, keys, key) -> int:
        for i, k in enumerate(keys):
            if key <= k:
                return i
        return len(keys)

    def _split_leaf(self, node: BPlusNode):
        """Split a full leaf. Returns (promoted_key, new_right_leaf)."""
        mid = len(node.keys) // 2
        new_leaf = BPlusNode(is_leaf=True)
        new_leaf.keys = node.keys[mid:]
        new_leaf.values = node.values[mid:]
        new_leaf.next_leaf = node.next_leaf

        node.keys = node.keys[:mid]
        node.values = node.values[:mid]
        node.next_leaf = new_leaf

        # Promote the first key of the new leaf
        return (new_leaf.keys[0], new_leaf)

    def _split_internal(self, node: BPlusNode):
        """Split a full internal node. Returns (promoted_key, new_right_node)."""
        mid = len(node.keys) // 2
        promoted_key = node.keys[mid]

        new_internal = BPlusNode(is_leaf=False)
        new_internal.keys = node.keys[mid + 1:]
        new_internal.children = node.children[mid + 1:]

        node.keys = node.keys[:mid]
        node.children = node.children[:mid + 1]

        return (promoted_key, new_internal)

    # ------------------------------------------------------------------- debug

    def _all_leaves(self):
        """Walk leaf chain; useful for verification."""
        leaf = self._leftmost_leaf(self.root)
        while leaf:
            yield from zip(leaf.keys, leaf.values)
            leaf = leaf.next_leaf

    def _leftmost_leaf(self, node):
        if node.is_leaf:
            return node
        return self._leftmost_leaf(node.children[0])


# ----------------------------------------------------------------- Tests

def test_bplus_tree():
    tree = BPlusTree()
    keys = random.sample(range(1000), 100)
    for k in keys:
        tree.insert(k, k * 10)

    # Verify every key is searchable
    for k in keys:
        assert tree.search(k) == k * 10, f"search({k}) failed"

    # Verify range_scan returns sorted results
    results = tree.range_scan(0, 999)
    result_keys = [r[0] for r in results]
    assert result_keys == sorted(result_keys), "range_scan not sorted"
    assert set(result_keys) == set(keys), "range_scan missing keys"

    # Spot-check a sub-range
    lo, hi = 200, 600
    sub = tree.range_scan(lo, hi)
    expected = sorted(k for k in keys if lo <= k <= hi)
    assert [r[0] for r in sub] == expected, f"range_scan({lo},{hi}) wrong"

    print("All B+ tree tests passed.")


if __name__ == "__main__":
    random.seed(42)
    test_bplus_tree()
```

**Implementation notes**:

- `ORDER = 4` means internal nodes hold at most 3 keys and 4 children. Leaf nodes hold at most 3 key-value pairs before splitting.
- The split functions maintain the leaf sibling chain — this is what makes `range_scan` O(k) after the initial O(log n) descent.
- `_split_internal` promotes the middle key up without keeping it in either child (unlike `_split_leaf` which keeps the promoted key in the right child). This is the B+ tree invariant: promoted internal keys are routing-only.
- For production use, add: node locking (B-link tree protocol for concurrent access), page serialization/deserialization (buffer pool manager), and WAL integration.

---

## 4. Python Implementation: Bloom Filter + SkipList

LSM trees are built from two fundamental data structures: a bloom filter (for probabilistic membership) and a skip list (for the in-memory MemTable). Understanding their internals demystifies LSM read and write paths.

### 4.1 Bloom Filter

A bloom filter answers "is key X definitely not in this set?" with zero false negatives and configurable false positive rate. It uses `k` independent hash functions and a bit array of size `m`.

**Space formula**: For n items and false positive probability p:
- `m = -n * ln(p) / (ln(2))²` bits
- `k = (m/n) * ln(2)` hash functions
- At p=1% and n=1,000,000: m ≈ 9.6 MB, k ≈ 7 hash functions

```python
import math
import hashlib
import struct

class BloomFilter:
    """
    Bloom filter with k hash functions derived from two base hashes
    (Kirsch-Mitzenmacher double hashing trick — only 2 hash computations
    regardless of k, saving CPU significantly).
    """
    def __init__(self, expected_items: int, false_positive_rate: float = 0.01):
        self.n = expected_items
        self.p = false_positive_rate
        # Optimal bit array size and hash count
        self.m = max(1, math.ceil(-expected_items * math.log(false_positive_rate)
                                  / (math.log(2) ** 2)))
        self.k = max(1, round((self.m / expected_items) * math.log(2)))
        self.bits = bytearray(math.ceil(self.m / 8))
        self._added = 0

    def _hash_pair(self, key: bytes):
        """Return two independent 64-bit hashes for key."""
        h = hashlib.blake2b(key, digest_size=16).digest()
        h1 = struct.unpack_from('<Q', h, 0)[0]
        h2 = struct.unpack_from('<Q', h, 8)[0]
        return h1, h2

    def _bit_positions(self, key: bytes):
        h1, h2 = self._hash_pair(key)
        for i in range(self.k):
            # gi(x) = h1(x) + i * h2(x)  mod m
            yield (h1 + i * h2) % self.m

    def add(self, key):
        if isinstance(key, str):
            key = key.encode()
        for pos in self._bit_positions(key):
            self.bits[pos >> 3] |= (1 << (pos & 7))
        self._added += 1

    def may_contain(self, key) -> bool:
        """Returns False if key is DEFINITELY absent. True means probably present."""
        if isinstance(key, str):
            key = key.encode()
        return all(
            self.bits[pos >> 3] & (1 << (pos & 7))
            for pos in self._bit_positions(key)
        )

    @property
    def actual_fpr(self) -> float:
        """Empirical false positive rate based on items added."""
        if self._added == 0:
            return 0.0
        n = self._added
        return (1 - math.exp(-self.k * n / self.m)) ** self.k

    def __repr__(self):
        return (f"BloomFilter(n={self.n}, p={self.p}, "
                f"m={self.m} bits, k={self.k} hashes, "
                f"actual_fpr={self.actual_fpr:.4f})")


def test_bloom_filter():
    bf = BloomFilter(expected_items=10_000, false_positive_rate=0.01)
    present = set(range(0, 10_000))
    for k in present:
        bf.add(str(k))

    # Zero false negatives
    for k in present:
        assert bf.may_contain(str(k)), f"False negative for {k}"

    # Measure false positive rate on absent keys
    absent = range(10_000, 20_000)
    fp = sum(1 for k in absent if bf.may_contain(str(k)))
    fpr = fp / len(list(absent))
    assert fpr < 0.03, f"FPR too high: {fpr:.4f}"
    print(f"BloomFilter test passed. FPR={fpr:.4f}, expected≤0.01, {bf}")
```

### 4.2 Skip List (LSM MemTable)

A skip list is a probabilistic data structure that provides O(log n) average case for insert, delete, and search — matching a balanced BST — but is much simpler to implement for concurrent access (a single CAS per level, vs. tree rotations requiring multi-node locking).

A skip list is multiple sorted linked lists layered on top of each other. Level 0 contains all elements. Level 1 contains ~50% of elements (each node is promoted to the next level with probability 0.5). Level k contains ~1/2^k of all elements.

**Search**: Start at the top level, walk right until the next node's key exceeds target, then drop down one level. Repeat until Level 0. Total path length: O(log n) expected.

```python
import random
from dataclasses import dataclass, field
from typing import Optional, List, Any

MAX_LEVEL = 16
PROBABILITY = 0.5


@dataclass
class SkipNode:
    key: Any
    value: Any
    forward: List[Optional['SkipNode']] = field(default_factory=list)

    def __post_init__(self):
        if not self.forward:
            self.forward = [None] * MAX_LEVEL


class SkipList:
    """
    Sorted skip list — the data structure used for LSM MemTables in RocksDB.
    O(log n) average insert/search. In production, forward pointers use
    atomic CAS for lock-free concurrent writes from multiple threads.
    """
    def __init__(self):
        self.header = SkipNode(key=None, value=None)
        self.level = 0
        self.size = 0

    def _random_level(self) -> int:
        lvl = 0
        while random.random() < PROBABILITY and lvl < MAX_LEVEL - 1:
            lvl += 1
        return lvl

    def insert(self, key, value):
        update = [None] * MAX_LEVEL
        current = self.header

        for i in range(self.level, -1, -1):
            while current.forward[i] and current.forward[i].key < key:
                current = current.forward[i]
            update[i] = current

        # Check for existing key (update value)
        candidate = current.forward[0]
        if candidate and candidate.key == key:
            candidate.value = value
            return

        new_level = self._random_level()
        if new_level > self.level:
            for i in range(self.level + 1, new_level + 1):
                update[i] = self.header
            self.level = new_level

        new_node = SkipNode(key=key, value=value)
        for i in range(new_level + 1):
            new_node.forward[i] = update[i].forward[i]
            update[i].forward[i] = new_node

        self.size += 1

    def search(self, key) -> Optional[Any]:
        current = self.header
        for i in range(self.level, -1, -1):
            while current.forward[i] and current.forward[i].key < key:
                current = current.forward[i]
        current = current.forward[0]
        if current and current.key == key:
            return current.value
        return None

    def range_scan(self, start, end) -> List[tuple]:
        results = []
        current = self.header
        # Find start position using express lanes
        for i in range(self.level, -1, -1):
            while current.forward[i] and current.forward[i].key < start:
                current = current.forward[i]
        current = current.forward[0]
        while current and current.key <= end:
            results.append((current.key, current.value))
            current = current.forward[0]
        return results

    def flush_sorted(self) -> List[tuple]:
        """Drain the MemTable to a sorted list (for SSTable creation)."""
        items = []
        current = self.header.forward[0]
        while current:
            items.append((current.key, current.value))
            current = current.forward[0]
        return items


def test_skiplist():
    sl = SkipList()
    keys = list(range(1000))
    random.shuffle(keys)
    for k in keys:
        sl.insert(k, k * 2)

    # Exact search
    for k in keys:
        assert sl.search(k) == k * 2, f"search({k}) failed"

    # Range scan returns sorted results
    results = sl.range_scan(100, 200)
    assert [r[0] for r in results] == list(range(100, 201)), "range_scan failed"

    # flush_sorted returns all items in order
    all_items = sl.flush_sorted()
    assert [x[0] for x in all_items] == list(range(1000)), "flush not sorted"
    print("SkipList tests passed.")


if __name__ == "__main__":
    random.seed(42)
    test_bloom_filter()
    test_skiplist()
```

**Why skip lists for MemTables?** RocksDB's `InlineSkipList` uses a wait-free CAS-based insert that allows multiple writer threads to insert concurrently without taking a mutex on the entire list — only atomic pointer swaps per node level. Red-black trees require rebalancing that touches multiple nodes, making lock-free concurrent writes significantly harder to implement correctly.

---

## 5. MVCC — Multi-Version Concurrency Control

### 5.1 The Core Idea

Classical locking (2PL, two-phase locking) forces readers to wait for writers and writers to wait for readers. Under typical OLTP workloads (many concurrent short reads + writes), this creates contention hotspots. MVCC eliminates reader-writer contention by a simple principle: **never overwrite a row; instead, create a new version and mark the old version as superseded**.

Readers see a consistent snapshot (the set of versions visible as of their transaction start time). Writers create new versions without touching the snapshot being read. Result: readers never block writers, writers never block readers. The tradeoff is storage overhead from multiple versions and the background work to clean them up.

**Isolation level anomaly matrix** — which anomalies each level prevents:

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Write Skew |
|----------------|-----------|--------------------|--------------|-----------:|
| Read Uncommitted | Possible | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible* | Possible |
| Snapshot Isolation | Prevented | Prevented | Prevented | **Possible** |
| Serializable (SSI) | Prevented | Prevented | Prevented | Prevented |

*PostgreSQL's Repeatable Read is implemented as Snapshot Isolation — phantoms are actually prevented, but write skew is still possible.

The critical takeaway: **Snapshot Isolation (PostgreSQL's default `REPEATABLE READ`) prevents phantoms but NOT write skew**. If your application has constraints that span multiple rows (e.g., "at most one primary contact per customer"), you need `SERIALIZABLE` or explicit locking.

### 5.2 PostgreSQL's MVCC Implementation

Every heap tuple (row version) in PostgreSQL carries two system columns:

| Column | Type | Meaning |
|--------|------|---------|
| `xmin` | `xid` (uint32) | Transaction ID that **created** this version |
| `xmax` | `xid` (uint32) | Transaction ID that **deleted/updated** this version (0 = live) |
| `cmin`/`cmax` | `cid` (uint32) | Command ID within transaction (for statement-level visibility) |
| `ctid` | `(page, slot)` | Physical location; points to newer version after UPDATE |

**Visibility rule**: A row version is visible to transaction T if:

1. `xmin` is committed AND `xmin` started before T's snapshot time AND `xmin` is not in T's list of in-progress transactions at snapshot time.
2. AND (`xmax` is null/zero, OR `xmax` aborted, OR `xmax` started after T's snapshot time, OR `xmax` is in T's list of in-progress transactions at snapshot time).

PostgreSQL materializes this as a **snapshot** at transaction start: `(xmin_horizon, xmax_horizon, active_xids[])`. Checking visibility is O(1) for the common case (xmin and xmax committed long before snapshot) and O(|active_xids|) for recent transactions.

**What happens on UPDATE**:

```sql
UPDATE employees SET salary = 90000 WHERE id = 42;
```

PostgreSQL does NOT modify the existing row. It:
1. Inserts a new heap tuple with `xmin = current_xid`, `salary = 90000`.
2. Sets `xmax = current_xid` on the old tuple.
3. If the new tuple is on the same page (HOT update — see §4.6), sets `ctid` on old tuple pointing to new.

The old version remains until VACUUM removes it.

### 5.3 Snapshot Isolation and Its Anomalies

Snapshot Isolation (SI) prevents the classic anomalies: dirty reads, non-repeatable reads, phantom reads. But it does NOT prevent **write skew**.

**Write skew scenario — the on-call doctor problem**:

```
Constraint: at least one doctor must be on call at all times.
Table: on_call(doctor_id, is_on_call)
Current state: Alice=true, Bob=true

T1 (Alice checking out): SELECT count(*) WHERE is_on_call = true → returns 2
T2 (Bob checking out):   SELECT count(*) WHERE is_on_call = true → returns 2
T1: UPDATE on_call SET is_on_call = false WHERE doctor_id = 'Alice'
T2: UPDATE on_call SET is_on_call = false WHERE doctor_id = 'Bob'
T1 commits. T2 commits.
Final state: both doctors off call. Constraint violated.
```

Neither transaction saw a conflict. Each read a count ≥ 1, so each decided it was safe to remove itself. This is write skew: two transactions read overlapping data and write disjoint data in a way that violates a global constraint.

Prevention options:
- `SELECT ... FOR UPDATE` (explicit locking, blocks under high contention).
- Use `SERIALIZABLE` isolation (triggers SSI — see §4.4).
- Restructure as a single row with a counter (and use `UPDATE ... WHERE count > 1`).

### 5.4 Serializable Snapshot Isolation (SSI)

PostgreSQL 9.1 introduced SSI (Ports & Grandi, 2012). SSI detects **dangerous structures** — specifically, **read-write anti-dependency cycles** — that indicate a potential non-serializable execution and aborts one transaction.

An anti-dependency (rw-antidep) edge T1 → T2 exists if T1 read something that T2 later wrote. A cycle of two rw-antidep edges (T1 → T2 → T1) indicates a potential write skew. SSI tracks these edges and aborts one participant when a dangerous cycle is detected.

The implementation uses `pg_stat_activity` predicate locks and an in-memory conflict graph. The overhead is low for read-heavy workloads (typically < 5% throughput reduction) but increases with write conflicts.

Enable with:
```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

### 5.5 VACUUM

Because old row versions are never overwritten, they accumulate as **dead tuples**. The heap file grows unboundedly without cleanup. `VACUUM` does:

1. **Scan heap pages**: identify dead tuples (xmax committed AND xmax < oldest active xid).
2. **Remove dead tuples**: zero out the tuple slot, update the page's free space map.
3. **Update visibility map**: mark pages where all tuples are known-visible to all transactions (index-only scans use this to skip heap fetches).
4. **Truncate trailing empty pages**: shrink the file if many trailing pages are empty.

`VACUUM FULL` is heavier: it rewrites the entire table to a new file, reclaiming all dead space. It holds an `AccessExclusiveLock` — table is unavailable for reads and writes during the rewrite. Avoid in production for large tables.

**`pg_repack`** is the production alternative to `VACUUM FULL`. It rebuilds the table online:
1. Creates a shadow table with the new, compact layout.
2. Applies delta changes (captured via a trigger) during the rebuild.
3. Swaps the names atomically at the end with an `ACCESS EXCLUSIVE` lock held for only milliseconds.

```sql
-- Install pg_repack extension, then:
pg_repack -t orders -d mydb --no-superuser-check
-- Table remains online throughout; brief lock only at swap
```

`autovacuum` runs automatically when:
```
dead_tuples > autovacuum_vacuum_threshold (default 50)
AND
dead_tuples / (live_tuples + dead_tuples) > autovacuum_vacuum_scale_factor (default 0.2)
```

For a 10M-row table, autovacuum triggers after ~2M dead tuples. Under 50K updates/sec, that's ~40 seconds of lag before vacuum begins.

**Partitioning as a vacuum strategy**: For time-series tables, partition by day or hour:

```sql
CREATE TABLE sensor_readings (
    recorded_at  TIMESTAMPTZ NOT NULL,
    sensor_id    INT NOT NULL,
    value        FLOAT NOT NULL
) PARTITION BY RANGE (recorded_at);

CREATE TABLE sensor_readings_2024_06
    PARTITION OF sensor_readings
    FOR VALUES FROM ('2024-06-01') TO ('2024-07-01');
```

Dropping an old partition (`DROP TABLE sensor_readings_2024_01`) is instantaneous and lock-free on the parent. No vacuum needed — the entire partition file is deleted at the OS level. This sidesteps the MVCC bloat problem entirely for append-mostly time-series workloads.

### 5.6 HOT (Heap Only Tuple) Updates

If an UPDATE modifies only columns that are NOT indexed, PostgreSQL can use the HOT optimization:

1. Insert the new tuple on the **same heap page** as the old tuple.
2. Set `ctid` in the old tuple to point to the new tuple.
3. Do NOT update any indexes — they still point to the old tuple's ctid.
4. When an index scan finds the old ctid, PostgreSQL follows the HOT chain to find the current version.

This avoids index writes for non-indexed-column updates, reducing I/O and index bloat. HOT is why `fillfactor < 100` on tables matters: it leaves room in each page for HOT updates. If the page is full, HOT fails and a new page is used with full index updates.

Check HOT effectiveness:
```sql
SELECT n_tup_hot_upd, n_tup_upd FROM pg_stat_user_tables WHERE relname = 'your_table';
-- n_tup_hot_upd / n_tup_upd should be high (> 80%) for heavily-updated tables
```

### 5.7 Transaction ID Wraparound

PostgreSQL uses 32-bit transaction IDs (`xid`). With 2^32 ≈ 4.3 billion possible xids, and a circular comparison model where any xid appears "in the past" or "in the future" based on modular arithmetic, the system can only distinguish transactions within a window of 2^31 ≈ 2.1 billion IDs.

If a table has a row with `xmin = 1000` and the current xid reaches `1000 + 2^31`, PostgreSQL's visibility logic will consider xmin=1000 to be "in the future" — the row becomes invisible. All data older than the horizon becomes inaccessible. This is **transaction ID wraparound**, and it is a genuine production emergency.

`VACUUM FREEZE` prevents this: it replaces real xmin/xmax values in tuples with `FrozenTransactionId` (a special value that is always considered "committed and in the past"). Once frozen, a tuple's visibility never depends on the xid comparison logic.

PostgreSQL tracks per-table `relfrozenxid` in `pg_class`. The distance from current xid to `relfrozenxid` is the **age** of the table. When age approaches 2^31, PostgreSQL forces autovacuum with freeze enabled.

**The Sentry incident (2015)**: Sentry suffered a 9-hour outage when their PostgreSQL database hit transaction ID wraparound. The database went into single-user emergency mode and required manual intervention to run `VACUUM FREEZE` on affected tables. Monitor with:

```sql
SELECT relname, age(relfrozenxid), 2^31 - age(relfrozenxid) AS remaining_xids
FROM pg_class WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC LIMIT 10;
```

Alert when `age(relfrozenxid)` exceeds 1.5 billion.

### 5.8 MySQL InnoDB MVCC: The Undo Log Approach

InnoDB implements MVCC differently from PostgreSQL, with important implications for garbage collection and read performance:

**PostgreSQL approach**: Old row versions live in the heap alongside live rows. They're invisible to most transactions but physically present until VACUUM removes them. The heap can become bloated but reads of the current version are always in-place.

**InnoDB approach**: Each row in the clustered index (InnoDB's B+ tree) stores only the **current version** plus a DB_TRX_ID (last modifier) and DB_ROLL_PTR (pointer to the undo log entry). Old versions live in a separate **undo log** structure, chained via roll pointers. To reconstruct the version visible to a snapshot, InnoDB walks the undo chain backward.

```
Clustered index page:
  row: (id=42, name="Alice", salary=90000, DB_TRX_ID=5001, DB_ROLL_PTR=→undo1)

Undo log segment (for TRX_ID=5001):
  undo1: (id=42, name="Alice", salary=80000, prev→undo2)

Undo log segment (for TRX_ID=4800):
  undo2: (id=42, name="Alice", salary=70000, prev→NULL)
```

A transaction that started before TRX_ID=5001 reads: current row → not visible (TRX_ID too new) → follow undo chain → undo1: TRX_ID=4800 visible → return salary=80000.

**Tradeoffs**:

| | PostgreSQL | InnoDB |
|-|-----------|--------|
| Old versions location | Heap (clustered with live rows) | Undo log (separate structure) |
| GC mechanism | VACUUM (explicit, tunable) | Purge thread (background, continuous) |
| Read of current version | Always one heap fetch | One clustered index fetch |
| Read of old version | Follow HOT chain (fast) | Follow undo chain (may be slow if many versions) |
| Bloat risk | Heap bloat (needs VACUUM) | Undo tablespace bloat (needs purge) |
| Long transactions | Block vacuum (snapshot horizon) | Block purge (history list length grows) |

InnoDB's `History List Length` (HLL) in `SHOW ENGINE INNODB STATUS` is the InnoDB equivalent of PostgreSQL's dead tuple count — high HLL means the purge thread is falling behind long-running transactions.

---

## 6. Write-Ahead Logging (WAL)

### 6.1 The Golden Rule

**WAL golden rule**: A WAL record describing a change must reach durable storage before the changed data page reaches durable storage.

Why? Consider the alternative — data page written first, then crash before WAL write:
- The data page shows a committed change.
- The WAL doesn't show it.
- Recovery has no record of the change.
- On restart, the data page might be inconsistent (half-written).
- We cannot undo or redo what we don't know about.

With WAL-first: if we crash after WAL write but before data page write, we can redo the change from WAL. If we crash before WAL write, the data page hasn't been written either (WAL first) — no corruption.

This property is enforced by `fsync()` or `fdatasync()` calls. PostgreSQL ensures WAL is fsynced at transaction commit before reporting success to the client. This is why `synchronous_commit = off` is dangerous: it breaks the golden rule to gain performance.

### 6.2 WAL Record Structure (PostgreSQL)

```
WAL Record:
  xl_tot_len      uint32   total record length
  xl_xid          xid      transaction id
  xl_prev         LSN      pointer to previous record (for backwards traversal)
  xl_info         uint8    info flags (includes rmgr-specific bits)
  xl_rmid         uint8    resource manager ID (heap, index, btree, sequence, ...)
  [2 bytes padding]
  xl_crc          uint32   CRC of record
  [record-type-specific data]
```

The **LSN** (Log Sequence Number) is a 64-bit byte offset in the WAL stream. It monotonically increases and uniquely identifies a position in the WAL.

Resource managers (`rmgr`) are pluggable handlers for specific record types:
- `XLOG`: checkpoints, full-page images.
- `Heap`: INSERT, UPDATE, DELETE of heap tuples.
- `Btree`: page splits, insertions in btree indexes.
- `Transaction`: COMMIT, ABORT, PREPARE.

### 6.3 The ARIES Algorithm

ARIES (Algorithms for Recovery and Isolation Exploiting Semantics, Mohan et al. 1992) is the theoretical foundation of crash recovery in most databases. It has three phases:

**Phase 1 — Analysis** (forward scan from last checkpoint LSN):
- Build a table of dirty pages (pages modified after last checkpoint but not flushed).
- Build a table of active transactions at crash time.
- Determine the redo point (earliest LSN of dirty page changes that haven't been flushed).

**Phase 2 — Redo** (forward scan from redo point):
- Replay every WAL record that affected a dirty page.
- Repeat changes exactly as recorded, even if the transaction later aborted.
- After redo, the database state is identical to what it was at crash time (including uncommitted changes).

**Phase 3 — Undo** (backward scan using prev LSN chain):
- For each active transaction at crash time (uncommitted), roll back changes using the prev LSN chain.
- Apply compensation log records (CLRs) to mark undo operations in WAL (so undo itself is restartable).

The key insight: redo first, then undo. This seems counterintuitive but is necessary because the dirty page table from the Analysis phase might have missed some pages (they might have been flushed to disk before crash). Redoing everything ensures the physical state is consistent before starting logical undo.

### 6.4 Checkpointing

A **checkpoint** is a point-in-time guarantee that all dirty pages modified before the checkpoint LSN have been flushed to disk. After a crash, WAL replay can start from the checkpoint LSN rather than from the beginning of WAL history.

Checkpoint procedure:
1. Record `BEGIN CHECKPOINT` in WAL.
2. Flush all dirty pages to disk (this is the slow part — can take minutes on a busy server).
3. Record `END CHECKPOINT` in WAL with checkpoint LSN, dirty page table, active xids.
4. Update `pg_control` (the control file) with the latest checkpoint LSN.

Checkpoint frequency is controlled by:
- `checkpoint_timeout` (default 5 min): force a checkpoint after this interval.
- `max_wal_size` (default 1 GB): force a checkpoint if WAL grows past this size.

**Tradeoff**: Frequent checkpoints reduce crash recovery time (less WAL to replay) but increase I/O because dirty pages are flushed more often. Infrequent checkpoints allow more WAL accumulation, increasing recovery time but reducing checkpoint I/O overhead.

### 6.5 Full-Page Images (FPI)

After a checkpoint, the **first write** to any data page must include a **full page image** (FPI) in the WAL record. Subsequent writes to the same page before the next checkpoint only record the delta.

Why FPIs? A 4 KB database page might span multiple filesystem blocks or disk sectors. A crash during a page write can leave a **partial page write** — half old data, half new data. There is no way to detect or repair this without a full copy of the pre-crash page state. By including the FPI in WAL after each checkpoint, recovery can restore the exact page state before applying the delta changes.

FPIs are expensive: they can constitute 30–50% of WAL volume on systems with many updates and frequent checkpoints. PostgreSQL offers WAL compression (`wal_compression = on`) to reduce this. With `wal_compression`, FPIs are LZ4/zstd compressed before writing — typically 3–5× size reduction for typical page data.

Enable checksums with `initdb --data-checksums` (or `pg_checksums` on an existing cluster) to detect corrupted pages at read time, independent of WAL.

### 6.6 Python: Minimal WAL + ARIES Recovery Simulation

This simulation illustrates the three ARIES phases (Analysis, Redo, Undo) on a tiny in-memory database:

```python
from dataclasses import dataclass, field
from typing import Optional, List, Dict
from enum import Enum

class OpType(Enum):
    BEGIN    = "BEGIN"
    WRITE    = "WRITE"   # logical write: (page_id, old_val, new_val)
    COMMIT   = "COMMIT"
    ABORT    = "ABORT"
    CKPT     = "CHECKPOINT"

@dataclass
class WALRecord:
    lsn: int
    xid: int
    op: OpType
    page_id: Optional[str] = None
    old_val: Optional[int] = None
    new_val: Optional[int] = None
    prev_lsn: int = 0   # previous LSN for this transaction (undo chain)

class MiniWAL:
    """Tiny WAL + ARIES recovery demo. Database = dict of page_id → value."""

    def __init__(self):
        self.log: List[WALRecord] = []
        self.next_lsn = 1
        self.last_ckpt_lsn = 0
        # Simulated database (in memory)
        self.pages: Dict[str, int] = {}
        # Per-xid last LSN (for prev_lsn chain)
        self._xid_last: Dict[int, int] = {}

    def _emit(self, xid: int, op: OpType, **kwargs) -> WALRecord:
        prev = self._xid_last.get(xid, 0)
        rec = WALRecord(lsn=self.next_lsn, xid=xid, op=op, prev_lsn=prev, **kwargs)
        self.log.append(rec)
        self._xid_last[xid] = self.next_lsn
        self.next_lsn += 1
        return rec

    def begin(self, xid: int):       self._emit(xid, OpType.BEGIN)
    def commit(self, xid: int):      self._emit(xid, OpType.COMMIT)
    def abort(self, xid: int):       self._emit(xid, OpType.ABORT)

    def write(self, xid: int, page_id: str, new_val: int):
        old_val = self.pages.get(page_id, 0)
        self._emit(xid, OpType.WRITE, page_id=page_id, old_val=old_val, new_val=new_val)
        self.pages[page_id] = new_val  # apply to buffer pool

    def checkpoint(self):
        self.last_ckpt_lsn = self.next_lsn - 1
        self._emit(0, OpType.CKPT)
        print(f"  [CKPT] at LSN {self.last_ckpt_lsn}")

    def simulate_crash(self):
        """Crash: lose in-memory state (pages reset to post-checkpoint snapshot)."""
        print("\n  *** CRASH ***")
        # Simulate: pages flushed up to checkpoint, then lost
        self.pages = {}  # reset; WAL survives on disk

    def recover(self):
        print("\n=== ARIES RECOVERY ===")

        # Phase 1: Analysis — find committed/aborted/active xids since last ckpt
        committed, aborted, active = set(), set(), set()
        print("\n-- Phase 1: Analysis --")
        for rec in self.log:
            if rec.lsn <= self.last_ckpt_lsn:
                continue
            if rec.op == OpType.BEGIN:    active.add(rec.xid)
            elif rec.op == OpType.COMMIT: committed.add(rec.xid); active.discard(rec.xid)
            elif rec.op == OpType.ABORT:  aborted.add(rec.xid);   active.discard(rec.xid)
        print(f"  Committed: {committed}, Aborted: {aborted}, Active at crash: {active}")

        # Phase 2: Redo — replay all WRITE records from redo point
        print("\n-- Phase 2: Redo --")
        redo_from = self.last_ckpt_lsn + 1
        for rec in self.log:
            if rec.lsn < redo_from or rec.op != OpType.WRITE:
                continue
            print(f"  Redo LSN={rec.lsn} xid={rec.xid}: {rec.page_id} → {rec.new_val}")
            self.pages[rec.page_id] = rec.new_val

        # Phase 3: Undo — rollback active (uncommitted) transactions
        print("\n-- Phase 3: Undo --")
        to_undo = active | (aborted - committed)
        for rec in reversed(self.log):
            if rec.xid in to_undo and rec.op == OpType.WRITE:
                print(f"  Undo LSN={rec.lsn} xid={rec.xid}: {rec.page_id} ← {rec.old_val}")
                self.pages[rec.page_id] = rec.old_val

        print(f"\n  Recovered pages: {self.pages}")


def demo_aries():
    wal = MiniWAL()
    wal.write(xid=1, page_id="A", new_val=10)
    wal.write(xid=2, page_id="B", new_val=20)
    wal.checkpoint()                               # checkpoint after T1 and T2 writes
    wal.write(xid=1, page_id="A", new_val=30)     # T1 writes again after ckpt
    wal.commit(xid=1)                              # T1 commits
    wal.write(xid=2, page_id="C", new_val=50)     # T2 writes C but never commits
    # T2 is active at crash time

    print("Pre-crash pages:", wal.pages)
    wal.simulate_crash()
    wal.recover()
    # Expected: A=30 (T1 committed), B=0 (T2 undone), C=0 (T2 undone)
    assert wal.pages.get("A") == 30, f"A should be 30, got {wal.pages.get('A')}"
    assert wal.pages.get("B", 0) == 0, f"B should be 0, got {wal.pages.get('B')}"
    assert wal.pages.get("C", 0) == 0, f"C should be 0, got {wal.pages.get('C')}"
    print("\nARIES recovery demo: all assertions passed.")

if __name__ == "__main__":
    demo_aries()
```

Run this to see the three phases play out with concrete LSN values. The key observation: T2's writes to B and C are redone in Phase 2 (the log says they happened), then undone in Phase 3 (T2 was not committed at crash time).

---

## 7. Buffer Pool Management

Every database sits between the on-disk storage engine and the query executor. The **buffer pool** (PostgreSQL calls it `shared_buffers`) is the in-memory cache of disk pages. Understanding how it works explains why cold-start queries are 100× slower than warm ones, and why `shared_buffers` tuning matters more than almost anything else.

### 7.1 The Buffer Pool as a Page Table

The buffer pool is a fixed-size array of page slots (each 8 KB in PostgreSQL). Each slot can hold one disk page identified by `(tablespace, relation, fork, block_number)`. A hash table maps page IDs to their slot index for O(1) lookup.

```
Buffer Pool (e.g., 8 GB = 1,048,576 slots of 8 KB):
┌──────┬─────────────────┬──────────┬────────┬──────────┐
│ Slot │    Page ID      │  Pin Ct  │  Dirty │  Usage   │
├──────┼─────────────────┼──────────┼────────┼──────────┤
│  0   │ rel=orders, b=0 │    2     │  Yes   │  clock=1 │
│  1   │ rel=index, b=7  │    0     │  No    │  clock=0 │
│  ...                                                   │
└──────────────────────────────────────────────────────-─┘
```

**Pin count**: While a backend is reading or modifying a page, it increments the pin count. An eviction algorithm never evicts a pinned page (it's in active use). Pin counts reach zero when no backend holds a reference.

**Dirty flag**: Set when a page is modified in memory. The page must be written to disk before its slot can be reused. A dirty page is written by a background writer (`bgwriter`) or at checkpoint.

### 7.2 Eviction Algorithms

When the buffer pool is full and a new page must be loaded, an eviction policy selects which existing page to evict (write to disk if dirty, then overwrite).

**LRU (Least Recently Used)**: Evict the page least recently accessed. Optimal for workloads with temporal locality. Catastrophic for sequential scans: scanning a 100 GB table fills the buffer pool with pages never seen again, evicting hot OLTP pages (the "buffer pool pollution" or "scan thrashing" problem).

**Clock Algorithm (used by PostgreSQL)**: A circular buffer of slots with one reference bit each. A "clock hand" sweeps around:
- If reference bit = 1: set to 0 (give a second chance) and advance.
- If reference bit = 0: evict this slot.

When a page is accessed, set its reference bit to 1. This approximates LRU without the overhead of maintaining an exact access list.

PostgreSQL's `clock_sweep_cost` parameter controls how aggressively the clock hand advances. For sequential scans, PostgreSQL uses a small ring buffer (256 KB by default) isolated from the main pool — preventing scan thrashing.

```python
class ClockReplacer:
    """Simplified Clock (Second-Chance) eviction algorithm."""
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.frames = [None] * capacity      # page ids
        self.ref_bits = [False] * capacity
        self.hand = 0
        self.size = 0

    def access(self, frame_idx: int):
        self.ref_bits[frame_idx] = True

    def evict(self) -> int:
        """Return frame index to evict. Raises if all frames pinned."""
        sweeps = 0
        while sweeps < 2 * self.capacity:
            if not self.ref_bits[self.hand]:
                evicted = self.hand
                self.hand = (self.hand + 1) % self.capacity
                return evicted
            self.ref_bits[self.hand] = False
            self.hand = (self.hand + 1) % self.capacity
            sweeps += 1
        raise RuntimeError("No evictable frame (all pinned)")
```

**ARC (Adaptive Replacement Cache)**: Used by ZFS and some database systems. Maintains two LRU lists: T1 (recently used once) and T2 (recently used twice). Ghost lists (T1_ghost, T2_ghost) track recently evicted entries. Self-tuning: if ghost hits in T1 dominate, increase T1 size; if T2 ghost hits dominate, increase T2 size. ARC adapts to both recency and frequency workloads automatically, outperforming plain LRU on mixed workloads.

### 7.3 Double Buffering and O_DIRECT

A subtle issue: the OS also maintains a page cache. Without special configuration, database page writes go: database buffer pool → OS page cache → disk. This means pages are cached twice (double buffering), wasting memory.

PostgreSQL uses `O_DIRECT` (bypassing the OS page cache) only for WAL segments when `wal_sync_method = open_datasync`. The main data files use the OS page cache. On Linux, `posix_fadvise(POSIX_FADV_DONTNEED)` after a sequential scan hints the OS to evict those pages.

MySQL InnoDB uses `O_DIRECT` for its buffer pool by default (`innodb_flush_method = O_DIRECT`), eliminating double buffering.

### 7.4 Tuning `shared_buffers`

PostgreSQL's conventional wisdom: set `shared_buffers` to 25% of RAM, rely on OS page cache for the rest. Modern systems with large RAM (256 GB+) may benefit from higher ratios (40–50%), but returns diminish past that because PostgreSQL's own buffer management overhead grows.

Diagnostic: check hit ratio — a healthy OLTP system should see > 99% buffer hit rate:

```sql
SELECT
    sum(heap_blks_hit)   AS heap_hits,
    sum(heap_blks_read)  AS heap_reads,
    round(100.0 * sum(heap_blks_hit) /
          nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) AS hit_pct
FROM pg_statio_user_tables;
```

---

## 8. Index Types Beyond B+ Tree

PostgreSQL supports six index types. Choosing the wrong one for a query is a common performance trap.

### 8.1 Hash Index

```sql
CREATE INDEX idx_hash ON users USING HASH (email);
```

A hash index uses an extensible hash table: hash buckets, each a chain of heap tuple pointers. Lookup is O(1). But hash indexes:
- Support only `=` comparisons (no ranges, no ordering).
- Pre-PostgreSQL 10: not WAL-logged → not crash-safe. Now WAL-logged but still rarely superior to B-tree for equality-only columns (B-tree with a single equality condition is also very fast — one btree page descent = O(log n) which is typically 3–4 disk reads).

**When to use**: almost never. Only consider if you have equality-only lookups on very long keys (UUIDs, SHA256 hashes) and benchmarks prove the benefit.

### 8.2 GIN (Generalized Inverted Index)

```sql
CREATE INDEX idx_gin ON articles USING GIN (to_tsvector('english', body));
CREATE INDEX idx_gin_tags ON products USING GIN (tags);  -- tags is text[]
```

GIN is an inverted index: for each distinct value in the indexed collection (e.g., each lexeme in a text column, each element in an array), GIN stores a posting list of heap tuple pointers where that value appears.

| Operation | Performance |
|-----------|-------------|
| `@>` (contains) | O(log n) per element |
| `&&` (overlaps) | O(log n) per element |
| Full text `@@` | O(log n + result_size) |
| Updates | Slow (posting lists must be updated); use `fastupdate` |

GIN `fastupdate`: pending insertions are accumulated in an unsorted list and merged in batch, amortizing the update cost. But a query touching a large pending list pays the merge cost — set `gin_pending_list_limit` appropriately.

### 8.3 GiST (Generalized Search Tree)

GiST is a framework for building tree-structured indexes on arbitrary data types. The implementation provides the tree structure; operators provide predicates (the `consistent` and `union` methods).

Built-in GiST indexes in PostgreSQL:
- **Geometric types**: `<->` (distance), `&&` (overlap), `@>` (contains), `<@` (within) on `point`, `circle`, `polygon`, `box`.
- **Range types**: `int4range`, `tstzrange`, etc. — containment, overlap, adjacent.
- **PostGIS**: spatial indexes on `geometry` columns.

```sql
-- Find all events overlapping a time range
CREATE INDEX idx_gist ON events USING GiST (event_period);
SELECT * FROM events WHERE event_period && '[2024-01-01, 2024-03-31]'::tstzrange;
```

GiST pages store bounding boxes (or other predicates). A search prunes branches whose bounding box cannot satisfy the predicate. Worst case is O(n) (all boxes overlap), best case O(log n).

### 8.4 BRIN (Block Range Index)

```sql
CREATE INDEX idx_brin ON sensor_readings USING BRIN (recorded_at) WITH (pages_per_range = 128);
```

BRIN is the opposite of B+ tree: extremely small index, coarse-grained, only useful when the column value is highly correlated with physical storage order (i.e., values increase as the table grows, like timestamps or auto-increment IDs).

BRIN stores one summary record per `pages_per_range` heap pages: `(min_value, max_value)` for the range. A query `WHERE recorded_at > '2024-06-01'` checks each range summary and reads only ranges where `max_value >= query_value`.

BRIN index size for a 100M-row time-series table with `pages_per_range=128`: roughly `100M / 128 / (8KB/24bytes) ≈ 2,300 pages ≈ 18 MB`. A B-tree index on the same column: ~2.1 GB. BRIN is 100× smaller.

**Limitation**: BRIN is useless if the column is not correlated with physical order. After many updates and HOT updates, physical order degrades — run `CLUSTER` to re-establish it.

### 8.5 Choosing the Right Index

| Workload Pattern | Index Type |
|-----------------|------------|
| Equality, range, ordering on scalar | B-tree (default) |
| Equality only on long keys | Hash (benchmark first) |
| Full-text search, array contains | GIN |
| Geometric/spatial, range type overlap | GiST |
| Monotone column (timestamps, serial IDs) on huge table | BRIN |
| Partial: `WHERE active = true` | Partial B-tree |
| Expression: `lower(email)` | Expression B-tree |

---

## 9. What Textbooks Don't Tell You

### 9.1 B+ Tree Page Splits Are the Silent Killer

Textbooks describe splits as O(log n) events. In production, under high write throughput to a hotspot range (e.g., an auto-increment primary key always inserting at the right edge), **every insert splits the rightmost leaf**. This creates a "right-edge lock chain" — the rightmost leaf, its parent, and potentially grandparent are all under exclusive modification simultaneously.

PostgreSQL uses a **B-link tree** (Lehman-Yao protocol): each internal node stores a "high key" — the maximum key that should be in this subtree — and a right-sibling pointer. This allows concurrent readers to "follow the link" if a concurrent split moved their target to the right sibling, without requiring the writer to hold a lock on the parent during the split. This reduces the locking window significantly but requires careful implementation.

Mitigation: use `fillfactor = 70` on write-heavy indexes to pre-allocate space. For UUID primary keys (which insert randomly), splits are distributed across the tree — no hotspot but higher page count.

### 9.2 LSM is Not a Universal Win

The hype around LSM (driven by Cassandra/RocksDB/LevelDB adoption) has led to a misconception that LSM is simply "better than B-tree". It is not:

| Property | B+ Tree | LSM |
|----------|---------|-----|
| Write throughput (SSD) | Moderate | High |
| Write throughput (HDD) | Low | Very High |
| Read latency (point) | Low | Moderate (bloom filter dependent) |
| Range scan | Excellent | Poor (must merge multiple levels) |
| Space amplification | ~1.1x | ~1.1x–3x |
| Write amplification | ~3–10x | ~10–60x |
| Operational complexity | Low | High (compaction tuning) |

Use B+ tree for: relational OLTP with balanced read/write, complex queries with range scans, foreign keys and joins.
Use LSM for: write-heavy time-series, event logs, key-value stores, workloads where write throughput dominates and reads are mostly recent data (hot in MemTable or L0).

**The third option: columnar storage**. Both B+ tree and LSM are row-oriented — they store all columns of a row together. For OLAP queries that aggregate only a few columns across millions of rows (`SELECT avg(revenue), sum(units) FROM sales WHERE year=2024`), reading entire rows wastes I/O.

Columnar storage (Parquet, Apache Arrow, DuckDB, ClickHouse) stores all values for each column contiguously. A query reading 3 of 50 columns reads 6% of the data. With compression (dictionary encoding + run-length encoding on sorted columns), columnar storage is typically 5–20× smaller than row storage for analytical data.

**Do not use PostgreSQL for analytics on 10B+ rows.** Use a dedicated OLAP engine. The storage engine mismatch is fundamental, not a tuning problem.

### 9.3 MVCC Bloat is a Production Emergency

A common PostgreSQL anti-pattern: a table that receives constant updates (e.g., a status field on a high-traffic order table). Each update creates a dead tuple. If autovacuum can't keep up (blocked by a long-running transaction, or configured too conservatively), the table bloats:

- Heap file grows → table scans get slower.
- Index scans must traverse dead tuples to find live ones (heap fetch for each index entry, even dead ones, until `visibility_cutoff` optimization in PG 13+).
- Query planner sees stale statistics (autovacuum also runs ANALYZE) → bad plans.

Diagnosis:
```sql
SELECT relname,
       pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
       n_dead_tup,
       n_live_tup,
       round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;
```

For tables with > 20% dead tuples under active write workload, investigate whether autovacuum is being blocked by long transactions:

```sql
SELECT pid, now() - xact_start AS age, query FROM pg_stat_activity
WHERE xact_start IS NOT NULL ORDER BY age DESC LIMIT 5;
```

### 9.4 Sequential vs. Random I/O: The Fundamental Cost Model

Understanding when random I/O matters:

- **HDD**: Sequential read/write: ~100–200 MB/s. Random 4KB read: ~0.5 ms seek + transfer = ~1 ms → 1000 IOPS. The 200× gap between sequential and random I/O is why LSM was invented.
- **SSD (SATA)**: Sequential: ~500 MB/s. Random 4KB read: ~0.1 ms → 10,000 IOPS. Gap: ~10×.
- **NVMe SSD**: Sequential: ~3500 MB/s. Random 4KB read: ~0.02 ms → 50,000 IOPS. Gap: ~5×.

On NVMe, B+ tree random writes are nearly as fast as LSM sequential writes. The traditional LSM advantage is dramatically reduced. PostgreSQL on NVMe with high `random_page_cost` tuned to 1.1 (instead of default 4.0) can outperform LSM systems for mixed workloads.

### 9.5 MVCC and Foreign Key Lock Contention

PostgreSQL acquires a `RowShareLock` on the referenced row when checking a foreign key constraint during INSERT/UPDATE. Under concurrent inserts referencing the same parent row (e.g., many orders for the same customer), these lock checks serialize:

```sql
-- Thread 1: INSERT INTO orders(customer_id, ...) VALUES (42, ...)
-- Thread 2: INSERT INTO orders(customer_id, ...) VALUES (42, ...)
-- Both must check: does customer 42 exist? They both lock row 42 in customers.
```

This is usually fine (shared locks don't conflict with each other, only with exclusive locks), but if a concurrent DELETE or UPDATE on the parent row is attempted, it conflicts with all the shared locks, causing a pile-up.

Mitigation: defer FK checks (`DEFERRABLE INITIALLY DEFERRED`), batch inserts for the same parent, or redesign to avoid high-contention parent rows.

---

## 10. Edge Cases and Failure Modes

### 10.1 Half-Written Pages (Torn Writes)

Modern storage devices have sectors of 512 bytes or 4096 bytes. A database page of 8 KB spans multiple sectors. A power failure mid-write leaves a **torn page**: some sectors contain new data, others contain old data.

PostgreSQL's defense:
1. **Full page images in WAL** (described in §5.5): on recovery, the exact page state from before the first post-checkpoint write is available, so torn pages can be replaced.
2. **Page checksums** (`pg_checksums`): on every page read, verify checksum. Detect torn/corrupted pages immediately rather than silently propagating corruption.
3. **Double-write buffer** (MySQL InnoDB): before writing a page to its final location, write it to a sequential "doublewrite buffer" on disk. On recovery, if the target location has a torn page, copy from the doublewrite buffer. PostgreSQL avoids this overhead by relying on FPIs in WAL.

### 10.2 LSM Compaction Under Peak Traffic

Compaction is CPU and I/O intensive — it reads multiple SSTables, merge-sorts them, and writes new SSTables, then deletes old ones. During peak traffic, compaction competes with foreground read/write operations for:
- CPU cycles for decompression, comparison, re-compression.
- Disk I/O bandwidth for reading input files and writing output files.
- Memory for read buffers and merge heap.

Production tuning strategies:
- Limit compaction I/O: `rate_limiter_bytes_per_sec` in RocksDB limits compaction bandwidth.
- Separate compaction to background threads: `max_background_compactions` should match available parallelism without starving foreground I/O.
- Use I/O scheduling (Linux `ionice`, cgroups) to prioritize foreground.
- Monitor L0 file count as the leading indicator: rising L0 count → compaction debt accumulating → imminent read degradation.
- Set conservative write stall thresholds and alert on them before they fire.

### 10.3 Transaction ID Exhaustion in Production

The Sentry outage (2015) is the canonical case study. Sequence of events:
1. autovacuum was not running (or too slow) on a high-write table.
2. Table age (`age(relfrozenxid)`) approached 2^31.
3. PostgreSQL detected imminent wraparound and triggered an emergency `autovacuum FREEZE` on the table.
4. The emergency FREEZE required a full sequential scan of the table — very slow on a large table.
5. During FREEZE, the table was effectively unusable (FREEZE holds a ShareUpdateExclusiveLock, which blocks DDL but not DML — however, the I/O from FREEZE degraded system performance severely).
6. Total downtime: 9 hours.

Prevention checklist:
```sql
-- Run weekly:
SELECT relname, age(relfrozenxid),
       2147483647 - age(relfrozenxid) AS xids_remaining
FROM pg_class
WHERE relkind = 'r' AND age(relfrozenxid) > 500000000
ORDER BY age(relfrozenxid) DESC;

-- Alert if xids_remaining < 500,000,000 (500M remaining)
-- Manually trigger VACUUM FREEZE:
VACUUM FREEZE VERBOSE your_table;
```

Autovacuum settings to prevent this on high-write tables:
```sql
ALTER TABLE high_write_table SET (
    autovacuum_vacuum_cost_delay = 2,       -- ms; default 20; lower = faster vacuum
    autovacuum_vacuum_cost_limit = 800,     -- default 200; higher = more I/O per round
    autovacuum_freeze_max_age = 100000000   -- freeze aggressively at 100M (default 200M)
);
```

---

## 11. Production Monitoring Playbook

These SQL queries are the first things to run when investigating PostgreSQL performance issues. Each targets a specific failure mode described in this lesson.

### 11.1 Table Bloat and Vacuum Health

```sql
-- Dead tuple ratio per table (MVCC bloat detector)
SELECT
    schemaname,
    relname,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS total_size,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

Alert if `dead_pct > 20%` on any table with `n_live_tup > 100000`.

### 11.2 Transaction ID Wraparound Risk

```sql
-- Tables closest to wraparound (most dangerous first)
SELECT
    schemaname,
    relname,
    age(relfrozenxid)                            AS xid_age,
    2147483647 - age(relfrozenxid)               AS xids_remaining,
    pg_size_pretty(pg_total_relation_size(oid))  AS size
FROM pg_class
WHERE relkind = 'r'
  AND age(relfrozenxid) > 100000000
ORDER BY age(relfrozenxid) DESC
LIMIT 10;
```

**Alert thresholds**:
- `xids_remaining < 1,000,000,000` → Warning: accelerate autovacuum freeze.
- `xids_remaining < 500,000,000` → Critical: manually run `VACUUM FREEZE VERBOSE table_name`.
- `xids_remaining < 10,000,000` → Emergency: PostgreSQL will refuse writes to prevent corruption.

### 11.3 Long-Running Transactions Blocking Vacuum

```sql
-- Transactions holding snapshots that block vacuum
SELECT
    pid,
    usename,
    application_name,
    now() - xact_start              AS xact_age,
    now() - query_start             AS query_age,
    state,
    left(query, 100)                AS query_snippet
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND now() - xact_start > interval '5 minutes'
ORDER BY xact_start ASC;
```

Any transaction older than your SLA for vacuum lag is a problem. The oldest xact_start determines how far back vacuum's horizon is stuck.

### 11.4 Buffer Pool Hit Rate

```sql
-- Overall buffer hit ratio
SELECT
    sum(blks_hit)   AS hits,
    sum(blks_read)  AS reads,
    round(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit) + sum(blks_read), 0), 2) AS hit_pct
FROM pg_stat_database
WHERE datname = current_database();

-- Per-table hit ratio (find cold tables or under-cached indexes)
SELECT
    relname,
    heap_blks_hit,
    heap_blks_read,
    round(100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) AS heap_hit_pct,
    idx_blks_hit,
    idx_blks_read,
    round(100.0 * idx_blks_hit / NULLIF(idx_blks_hit + idx_blks_read, 0), 2)  AS idx_hit_pct
FROM pg_statio_user_tables
ORDER BY heap_blks_read + idx_blks_read DESC
LIMIT 20;
```

### 11.5 HOT Update Effectiveness

```sql
-- HOT ratio per table (high is good — means fewer index writes)
SELECT
    relname,
    n_tup_upd,
    n_tup_hot_upd,
    round(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 1) AS hot_pct
FROM pg_stat_user_tables
WHERE n_tup_upd > 1000
ORDER BY n_tup_upd DESC
LIMIT 20;
```

If `hot_pct < 50%` on a heavily updated table:
1. Check if `fillfactor` is set to < 100 (`\d+ table_name` or `pg_class.reloptions`).
2. Check if the updates are modifying indexed columns (HOT cannot apply if any updated column is indexed).

### 11.6 Index Bloat and Unused Indexes

```sql
-- Indexes never used since last stats reset (candidates for removal)
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND pg_relation_size(indexrelid) > 1024 * 1024  -- > 1 MB
ORDER BY pg_relation_size(indexrelid) DESC;
```

Unused indexes waste write I/O (every INSERT/UPDATE/DELETE must update every index). An index never scanned is pure overhead.

### 11.7 Lock Contention

```sql
-- Active lock waits (who is blocked by whom)
SELECT
    blocked.pid                     AS blocked_pid,
    blocked.query                   AS blocked_query,
    blocking.pid                    AS blocking_pid,
    blocking.query                  AS blocking_query,
    now() - blocked.query_start     AS wait_duration
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0
ORDER BY wait_duration DESC;
```

---

## 12. Hard Exercise: MVCC Under Load

**Setup**: A PostgreSQL table `orders` has 10 million rows. A long-running analytics query `T1` (e.g., `SELECT sum(amount) FROM orders WHERE status = 'completed'`) started 2 minutes ago and is still running. Since `T1` started, OLTP transactions have updated 500,000 rows (adding/modifying orders).

**Questions and Answers**:

### (a) How many row versions exist in the heap?

At minimum, **10,500,000** row versions:
- 10,000,000 original live versions (some may have been updated, but their originals remain as dead tuples).
- 500,000 new versions created by the OLTP updates.

In practice, the updated rows' old versions are dead (xmax is set by committed OLTP transactions), so:
- 9,500,000 live + unmodified.
- 500,000 dead (old versions of updated rows).
- 500,000 live (new versions of updated rows).
- Total physical tuples: **10,500,000**.

### (b) Are the dead rows from updates before T1 visible to T1?

The 500,000 updates all occurred **after** T1 started. T1 holds a snapshot taken at T1's start time. The OLTP transactions that performed updates have xids **greater** than T1's xmin horizon.

Under snapshot isolation: a row version is visible to T1 if `xmax` is null OR `xmax` is not yet committed as of T1's snapshot OR `xmax`'s transaction started after T1's snapshot.

The OLTP updates started after T1 → T1 sees the **old versions** of those 500,000 rows (the pre-update state). The new versions are NOT visible to T1 (their xmin > T1's xmax horizon). The old versions are still visible to T1 and will remain until T1 commits/aborts.

Answer: T1 sees the old (now "dead to other transactions") versions of the 500,000 updated rows. These appear live to T1 due to MVCC snapshot isolation.

### (c) Can autovacuum clean up the dead rows while T1 is running?

**No.** autovacuum cannot remove a dead tuple if any active transaction's snapshot can see it. The `oldest active snapshot xmin` (visible in `pg_stat_activity` and `pg_replication_slots`) is the hard lower bound for vacuum.

PostgreSQL tracks the oldest xmin across all active transactions as `relfrozenxid_horizon`. autovacuum will skip tuples with `xmax` >= this horizon. Since T1 started before the 500K updates, those "dead" tuples (old versions of updated rows) are still visible to T1 and cannot be vacuumed.

Once T1 completes, the horizon advances and autovacuum can reclaim those 500K dead tuples.

### (d) What setting controls how long autovacuum waits?

`old_snapshot_threshold` (PostgreSQL 9.6+, default `-1` = disabled): if enabled, PostgreSQL may allow vacuum to remove dead tuples even if they're older than the threshold, potentially causing errors for transactions with very old snapshots ("snapshot too old" error). This is an emergency escape valve.

For less aggressive control: `lock_timeout` and `statement_timeout` on the analytics query itself. Also relevant: `idle_in_transaction_session_timeout` prevents idle transactions from holding snapshots open indefinitely.

The primary mechanism is not a wait — it's that autovacuum simply skips what it can't clean. There is no built-in "wait then force" mechanism for regular autovacuum (only for the emergency wraparound case). The practical advice is to avoid long-running OLTP-blocking analytics queries; use read replicas for analytics instead.

### (e) Schema and autovacuum configuration for 50,000 updates/second

At 50,000 updates/sec, the table accumulates ~3 million dead tuples per minute. Standard autovacuum settings will fail to keep up.

**Schema design**:

```sql
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    status      VARCHAR(20) NOT NULL,
    amount      NUMERIC(12,2) NOT NULL,
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Separate hot columns from cold to enable HOT updates
    -- Only index columns that are actually queried by range
    CONSTRAINT orders_status_check CHECK (status IN ('pending','processing','completed','cancelled'))
) WITH (
    fillfactor = 70  -- leave 30% free space for HOT updates
);

-- Partial index on active orders only (most lookups are for non-completed orders)
CREATE INDEX idx_orders_active ON orders (customer_id, updated_at)
    WHERE status IN ('pending', 'processing')
    WITH (fillfactor = 80);

-- Status index with low cardinality — consider if really needed vs. seqscan
CREATE INDEX idx_orders_status ON orders (status, id)
    WITH (fillfactor = 80);
```

**Autovacuum configuration** (table-level, aggressive):

```sql
ALTER TABLE orders SET (
    autovacuum_enabled = on,
    autovacuum_vacuum_threshold = 1000,        -- trigger after 1000 dead tuples (not default 50)
    autovacuum_vacuum_scale_factor = 0.001,    -- 0.1% of table (not default 20%)
    autovacuum_vacuum_cost_delay = 1,          -- 1ms delay between rounds (not 20ms)
    autovacuum_vacuum_cost_limit = 2000,       -- 2000 cost units/round (not 200)
    autovacuum_analyze_threshold = 500,
    autovacuum_analyze_scale_factor = 0.001,
    autovacuum_freeze_max_age = 50000000       -- freeze at 50M xids (not 200M)
);
```

**System-level tuning** (`postgresql.conf`):

```ini
autovacuum_max_workers = 8          # more parallel workers (default 3)
autovacuum_naptime = 5s             # check more frequently (default 1min)
autovacuum_vacuum_cost_delay = 2ms  # faster global vacuum
maintenance_work_mem = 1GB          # more memory for vacuum sort
```

**Operational approach**:
- Route analytics queries to a read replica to avoid long-running transactions blocking vacuum on the primary.
- Monitor `pg_stat_user_tables.n_dead_tup` per minute; alert if > 5M dead tuples.
- Consider partitioning by `updated_at` (daily or hourly): vacuum operates per-partition, parallelizing cleanup across smaller tables and allowing old complete partitions to be dropped entirely (instant, lock-free).

---

## 13. Summary and Decision Tree

### 13.1 Core Concepts at a Glance

| Concept | Core Insight | Common Failure |
|---------|-------------|----------------|
| B+ tree | Disk-optimized search tree. O(log n) I/Os. All data in leaves, linked for range scans. | Hot-edge splits under monotone insert workloads. `fillfactor` is the knob. |
| LSM tree | Convert random writes to sequential. MemTable → SSTables → compaction. | L0 file count spike under write bursts → read degradation. Monitor L0 count. |
| MVCC | Never overwrite — create versions. Readers see snapshots. Writers never block readers. | Dead tuple accumulation under high UPDATE rates without sufficient vacuum. |
| WAL | Log before you write. Enables redo on crash. ARIES: Analyze → Redo → Undo. | `synchronous_commit = off` breaks the golden rule. FPIs dominate WAL volume. |
| Vacuum | MVCC dead-row cleanup. Horizon = oldest active snapshot. Cannot clean beyond it. | Long-running analytics query blocks vacuum → bloat cascade. Use read replicas. |
| xid wraparound | 32-bit xids exhausted at 2^31. Freeze writes FrozenXid to break the cycle. | Misconfigured autovacuum freeze thresholds. Sentry-class 9-hour outage. |
| Buffer pool | In-memory cache of disk pages. Clock eviction. Pin counts prevent eviction. | Scan thrashing: sequential scan evicts hot OLTP pages. Ring buffer mitigates. |
| Index types | B-tree is default. GIN for arrays/FTS. GiST for geometric. BRIN for sorted tables. | Wrong index type → full scans. Unused indexes → write overhead. |

### 13.2 Storage Engine Decision Tree

```
Is your workload write-heavy (> 10K writes/sec per node)?
├── YES → LSM tree (RocksDB, Cassandra, ScyllaDB)
│         ├── Time-series / append-only? → FIFO compaction, partition by time
│         └── Mixed read-write?          → Leveled compaction, tune bloom filter FPR
└── NO  → B+ tree (PostgreSQL, MySQL InnoDB)
          ├── Need ACID transactions?   → PostgreSQL with MVCC + SSI
          ├── Read-heavy OLAP?         → Consider columnar storage (Parquet/DuckDB)
          └── Mixed OLTP?              → PostgreSQL, tune shared_buffers + autovacuum

Choosing an index:
├── Scalar equality + range + ORDER BY → B-tree (default)
├── Full-text search, array membership  → GIN
├── Geometric distance / overlap        → GiST
├── Monotone column (ts, serial ID)     → BRIN (100× smaller than B-tree)
└── Equality-only on hash values        → Hash (rarely beats B-tree in practice)

MVCC tuning:
├── High UPDATE rate                    → fillfactor=70, aggressive autovacuum settings
├── Long analytics queries              → Dedicated read replica, avoid primary OLTP lock
├── Table age > 1.5B xids              → VACUUM FREEZE immediately, check autovacuum config
└── HOT ratio < 50%                    → Check if indexed columns are being updated
```

### 13.3 The Feedback Loop You Must Internalize

```
HIGH UPDATE RATE
    │
    ▼
DEAD TUPLES ACCUMULATE (MVCC — old row versions)
    │
    ▼
LONG ANALYTICS QUERY HOLDS SNAPSHOT
    │ (blocks autovacuum horizon from advancing)
    ▼
VACUUM CANNOT CLEAN DEAD TUPLES
    │
    ▼
TABLE BLOAT GROWS → HEAP PAGES INCREASE → MORE I/O PER SCAN
    │
    ▼
QUERY PLANNER STATS STALE → BAD PLANS → LONGER QUERIES
    │
    ▼
LONGER QUERIES HOLD SNAPSHOTS LONGER
    │
    └──────────────────────────────► (back to top: cycle accelerates)
```

Breaking the cycle requires one of:
1. Route analytics to a read replica (removes snapshot from primary).
2. Reduce autovacuum `vacuum_cost_delay` and increase `vacuum_cost_limit` so vacuum catches up faster.
3. Impose `statement_timeout` or `idle_in_transaction_session_timeout` to prevent runaway transactions.
4. Partition the table so vacuum operates on smaller, isolated units.

### 13.4 Interaction Map: What Touches What

```
Client Write ──► WAL ──► Buffer Pool ──► Data Pages ──► Disk
                              │
                    Dirty Page Tracking
                              │
                         Checkpoint
                              │
                    WAL Retention Trimmed

MVCC INSERT/UPDATE/DELETE:
    Row versioned in Heap (xmin/xmax)
        │
        ├──► Index entries written (or HOT: skipped)
        └──► Dead tuples accumulate
                    │
               autovacuum
                    │
               Scan + reclaim (blocked by old snapshots)
                    │
               Freeze if age > threshold
```

---

### Further Reading

| Paper / Book | Why Read It |
|-------------|-------------|
| Hellerstein et al., "Architecture of a Database System" (2007) | End-to-end database internals; free PDF online |
| O'Neil et al., "The Log-Structured Merge-Tree" (1996) | Original LSM paper; surprisingly readable |
| Mohan et al., "ARIES: A Transaction Recovery Method" (1992) | Foundation of crash recovery in every RDBMS |
| Graefe, "A Survey of B-tree Logging and Recovery Techniques" (2012) | Deep dive into B-tree concurrency and WAL |
| Ports & Grandi, "Serializable Snapshot Isolation in PostgreSQL" (2012) | SSI theory and PostgreSQL implementation |
| "Designing Data-Intensive Applications" — Kleppmann (2017) | Practitioner-level coverage of storage engines |

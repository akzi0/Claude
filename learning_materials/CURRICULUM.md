# CURRICULUM.md — Advanced Systems CS: Dependency Graph, Learning Paths, and Cross-Reference Guide

> **Quality standard**: Every section name cited below is drawn verbatim from the lesson files in this repository. No invented references.

---

## 1. Concept Dependency Graph

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONCEPT DEPENDENCY GRAPH (read bottom-up)                │
└─────────────────────────────────────────────────────────────────────────────┘

  L07: Cryptography Internals
  §1 Elliptic Curves · §4 ECDSA/EdDSA · §5 Schnorr/ZKP · §6 TLS 1.3
       │ requires: field arithmetic (math), hash functions
       │ benefits from: L03 §4 C++11 Memory Model (constant-time critical)
       │ benefits from: L06 §7 SIMD (vectorized bignum multiply)
       ▼
  L01: Distributed Consensus
  §2 Paxos · §3 Raft · §5 Byzantine Fault Tolerance · §6 What Textbooks Don't Tell You
       │ requires: basic networking, state machines
       │ benefits from: L02 §6 WAL (log replication IS a WAL), L02 §5 MVCC (read linearizability)
       │ benefits from: L03 §6 Lock-Free (compare-and-swap used in Raft leader election libs)
       ▼
  L02: Database Internals
  §1 B+ Tree · §2 LSM Trees · §5 MVCC · §6 WAL · §7 Buffer Pool
       │ requires: file I/O concepts, basic data structures (B-trees, skip lists)
       │ requires: L03 §1 The Memory Illusion (buffer pool coherence), L03 §3 Memory Barriers (WAL fsync)
       │ benefits from: L04 §5 Demand Paging (mmap vs O_DIRECT)
       │ benefits from: L06 §5 MESI Protocol (cache-friendly B+ Tree node size = cache line)
       ▼
  L03: Memory Models and Lock-Free Programming
  §1 The Memory Illusion · §2 CPU Memory Models · §4 C++11 Memory Model · §6 Lock-Free Data Structures
       │ requires: L06 §2 Tomasulo's Algorithm (why reordering happens at hardware level)
       │ requires: L06 §5 MESI Protocol (invalidation queues → store buffers)
       │ requires: L04 §2 x86-64 Page Tables (virtual addresses in CAS arguments)
       │ benefits from: L05 §4 Key Optimization Passes (compiler reordering = compiler barrier need)
       ▼
  L06: CPU Microarchitecture
  §2 Tomasulo's Algorithm · §3 Branch Prediction · §5 MESI Protocol · §11 Memory Ordering
       │ requires: digital logic basics (pipeline stages, registers)
       │ requires: L04 §2 x86-64 Page Tables, L04 §3 TLB (§10 TLB/Address Translation in L06)
       │ benefits from: L05 §5 Register Allocation (PRF in L06 is hardware register renaming)
       ▼
  L04: OS and Virtual Memory
  §2 x86-64 Page Tables · §3 TLB · §4 Huge Pages · §6 CoW and fork() · §9 What Textbooks Don't Tell You
       │ requires: basic OS concepts (processes, address spaces)
       │ requires: L06 §2 Tomasulo's (understand why TLB shootdown uses IPIs)
       │ benefits from: L05 §8 What Textbooks Don't Tell You (JIT and W^X pages)
       ▼
  L05: Compiler Internals
  §2 Building SSA Form · §4 Key Optimization Passes · §5 Register Allocation · §13 Instruction Selection
       │ requires: basic parsing (ASTs), graph theory (dominators)
       │ requires: L06 §7 SIMD (instruction selection targets SIMD units)
       │ requires: L04 §9 What Textbooks Don't Tell You (W^X for JIT code generation)
       └──────────────────────────────────────────────────────────────────────
```

### Dependency Narrative

**L06 (CPU Microarchitecture)** and **L04 (OS / Virtual Memory)** are the bedrock. L06's §2 "Out-of-Order Execution: Tomasulo's Algorithm" explains *why* memory reordering exists at all, making L03's §1 "The Memory Illusion" and §2 "CPU Memory Models" fully comprehensible rather than a list of rules to memorize. L04's §2 "x86-64 Four-Level Page Tables" and §3 "Translation Lookaside Buffer" are prerequisites for L06's §10 "TLB/Address Translation" and for understanding the virtual address operands in any lock-free CAS loop.

**L05 (Compiler Internals)** sits above L06 because §5 "Register Allocation via Graph Coloring" (Chaitin's algorithm) is the *software* dual of L06's hardware register renaming (RAT/PRF in Tomasulo). §13 "Instruction Selection" targets the exact execution units described in L06 §7 "SIMD and Instruction-Level Parallelism." §8 "What Textbooks Don't Tell You" covers JIT W^X page flips, which requires L04 §9.

**L03 (Memory Models)** depends on L06's MESI protocol explanation (§5 "Cache Coherence: MESI Protocol") and L04's virtual memory. Once those are solid, L03's §4 "C++11 Memory Model" — the six memory orders — becomes a direct mapping to hardware behaviors already understood. Lock-free structures in §6 then have a rigorous foundation.

**L02 (Database Internals)** draws on L03 for buffer pool coherence and WAL fsync semantics, and on L04 for the mmap vs O_DIRECT trade-off discussed in §7 "Buffer Pool Management." The B+ Tree node size (§1 "B+ Tree Internals") is driven by cache-line sizing from L06 §5.

**L01 (Distributed Consensus)** is the highest-level lesson. Log replication in §3 "Raft" is structurally identical to WAL from L02 §6 "Write-Ahead Logging." The CAS operations used in Raft client libraries draw on L03 §6 "Lock-Free Data Structures."

**L07 (Cryptography)** is largely self-contained (requires number theory) but benefits from L03 §4 for constant-time comparison semantics and from L06 §7 for SIMD-accelerated finite field arithmetic.

---

## 2. Learning Paths

### Path A — Systems / Backend / Infrastructure Engineer

**Goal**: Build, operate, and debug high-performance distributed systems.

| Step | Lesson | Why This Order |
|------|--------|---------------|
| 1 | L06: CPU Microarchitecture | Hardware reality first. §2 "Out-of-Order Execution: Tomasulo's Algorithm" and §5 "Cache Coherence: MESI Protocol" make everything downstream non-mysterious. |
| 2 | L04: OS and Virtual Memory | §2 "x86-64 Four-Level Page Tables," §3 "TLB," §6 "Copy-on-Write and fork()" — the OS-level substrate every backend process lives inside. |
| 3 | L03: Memory Models | §1 "The Memory Illusion" + §4 "C++11 Memory Model" + §6 "Lock-Free Data Structures." Now you can write concurrent code that is actually correct. |
| 4 | L02: Database Internals | §1 "B+ Tree Internals," §2 "LSM Trees," §5 "MVCC," §6 "WAL" — the storage engine that underlies every database you operate. |
| 5 | L01: Distributed Consensus | §2 "Paxos from First Principles," §3 "Raft," §6 "What Textbooks Don't Tell You" (Pre-vote, leader stickiness, read linearizability). The capstone for distributed systems work. |
| 6 | L05: Compiler Internals | §4 "Key Optimization Passes," §8 "What Textbooks Don't Tell You" (PGO, alias analysis, inlining). Optional but explains why your profiler shows what it shows. |
| 7 | L07: Cryptography | §6 "TLS 1.3 Handshake," §4 "ECDSA and EdDSA," §5 "Schnorr Protocol." Necessary for any system that crosses a trust boundary. |

**Rationale**: Hardware → OS → concurrency model → storage → consensus mirrors the actual call stack of a distributed system. Each lesson's "What Textbooks Don't Tell You" section should be read as a final pass after the main content.

---

### Path B — Security / Systems Security Researcher

**Goal**: Understand attack surfaces in hardware, OS, compilers, and cryptographic protocols.

| Step | Lesson | Key Security Content |
|------|--------|---------------------|
| 1 | L06: CPU Microarchitecture | §3 "Branch Prediction" (Spectre v1 — PHT poisoning, BTB aliasing). §12 "SMT/Hyper-Threading" (covert channels). |
| 2 | L04: OS and Virtual Memory | §9 "What Textbooks Don't Tell You" — ASLR entropy limits, W^X enforcement, JIT bypass patterns, KPTI and Meltdown mitigations via PCID in §3 "TLB." |
| 3 | L07: Cryptography | §1 "Elliptic Curves from First Principles" (ECDLP hardness, Pollard's rho, Shor's algorithm). §4 "ECDSA and EdDSA" (Sony PS3 nonce reuse catastrophe). §5 "Schnorr Protocol" (MuSig2 rogue-key attacks). §6 "TLS 1.3 Handshake" (0-RTT replay risk). |
| 4 | L03: Memory Models | §6 "Lock-Free Data Structures" (ABA problem, hazard pointer bypass). §8 "What Textbooks Don't Tell You" (NUMA side channels, seq_cst cost). |
| 5 | L05: Compiler Internals | §8 "What Textbooks Don't Tell You" (alias analysis limits, optimizer cliff — compiler eliding security-critical zeroing). §4 §4.2 DCE (dead code elimination removing crypto wipes). |
| 6 | L01: Distributed Consensus | §5 "Byzantine Fault Tolerance" — PBFT, HotStuff, Tendermint. §6 "What Textbooks Don't Tell You" (read linearizability, Pre-vote). |
| 7 | L02: Database Internals | §5 "MVCC" (write skew, SSI rw-antidependency cycles as correctness attacks). §5.7 "Transaction ID Wraparound" (Sentry outage). |

**Rationale**: Start with hardware attack surfaces (Spectre/Meltdown) where misunderstanding the microarchitecture is the root cause of vulnerability. Move to OS mitigations, then cryptographic protocol design, then software-level pitfalls. BFT and MVCC anomalies close the loop at the distributed/database level.

---

### Path C — Compiler / Programming Language Theory

**Goal**: Build compilers, interpreters, runtimes, or work on language design.

| Step | Lesson | Why This Order |
|------|--------|---------------|
| 1 | L05: Compiler Internals | The core. §2 "Building SSA Form" (Cytron 1991 IDF algorithm), §4 "Key Optimization Passes" (SCCP lattice, LICM), §5 "Register Allocation via Graph Coloring" (Chaitin), §11 "Control Dependence and the Program Dependence Graph," §13 "Instruction Selection" (BURG). |
| 2 | L06: CPU Microarchitecture | §2 "Out-of-Order Execution: Tomasulo's Algorithm" — hardware register renaming is the runtime dual of L05's §5 Chaitin allocation. §7 "SIMD and Instruction-Level Parallelism" — the target for §13 Instruction Selection. §3 "Branch Prediction" — what profile-guided optimization (PGO, L05 §8) is optimizing for. |
| 3 | L04: OS and Virtual Memory | §9 "What Textbooks Don't Tell You" — W^X and JIT code generation. §6 "CoW and fork()" for GC-based runtimes. §4 "Huge Pages" for large heap management. |
| 4 | L03: Memory Models | §4 "C++11 Memory Model" — GC write barriers and safepoints require precise memory ordering. §6 "Lock-Free Data Structures" — concurrent GC mark queues use these primitives. §7b "RCU: Read-Copy-Update" — used in JVM JIT invalidation. |
| 5 | L02: Database Internals | §2 "LSM Trees" — persistent data structure literature overlaps heavily. §5 "MVCC" — functional language persistent data structures share the structural sharing insight. |
| 6 | L07: Cryptography | §7 "SHA-256 Internals: The Compression Function" — compiler optimization of cryptographic code (constant-time constraints from §4 interact with §5 §4.2 DCE). |
| 7 | L01: Distributed Consensus | §3 "Raft" — distributed compilation, build systems, and replicated type-checkers. |

**Rationale**: L05 first because it is the direct object of study. L06 immediately after because the ISA and execution model are the target; you cannot write good instruction selection without understanding the machine you're targeting. L04 for runtime concerns. L03 for concurrent runtime correctness.

---

### Path D — Sprint (Time-Constrained: Best 3 Lessons)

**For**: Engineers with 2–3 weeks who need maximal density of transferable insight.

**Recommended trio**: **L03 → L02 → L06**

| Rank | Lesson | Standalone Value |
|------|--------|-----------------|
| 1 | L03: Memory Models and Lock-Free | §1 "The Memory Illusion" permanently changes how you reason about concurrent code. §4 "C++11 Memory Model" is the most practically applicable single chapter in the series. Bugs from ignoring this cost weeks of debugging. |
| 2 | L02: Database Internals | §5 "MVCC," §6 "WAL," §2 "LSM Trees." Every production system stores data. This lesson explains how *every major storage engine* actually works, replacing cargo-cult usage with first-principles understanding. |
| 3 | L06: CPU Microarchitecture | §2 "Tomasulo's Algorithm," §3 "Branch Prediction," §5 "MESI Protocol." Makes profiler output interpretable. Branch misprediction, false sharing, and ROB stalls become *nameable phenomena* rather than mystery slowdowns. |

**What you sacrifice**: L01 (consensus) requires L02 to be maximally useful anyway. L05 (compilers) requires L06. L04 (OS) is implicit background in L03 and L06. L07 (crypto) is a prerequisite domain of its own.

**Reading order within each lesson**: Main sections first, "What Textbooks Don't Tell You" last, Python examples as needed for concreteness.

---

## 3. Lesson Summary Cards

---

### Lesson 01 — Distributed Consensus

**Core Insight**: Consensus is impossible with even one faulty process in a fully asynchronous system (FLP), so every real protocol smuggles in a partial synchrony assumption — understanding *which* assumption reveals *when* a system will be unavailable.

**Hard Prerequisites**:
- State machine replication concept (deterministic application of log entries)
- Basic graph theory (quorum intersection proofs require reasoning about set overlaps)
- Network fundamentals: packet loss, message delay, partial failure vs. crash

**Key Mental Models**:
1. **Quorum Intersection** (§2 "Paxos from First Principles"): Any two quorums share at least one node → no two leaders can form conflicting majorities simultaneously.
2. **Log as Linearization** (§3 "Raft" — Log Structure): The replicated log is the system's ground truth; committed entries define the happens-before for all client operations.
3. **Leader Completeness** (§3 "Raft" — Leader Completeness): A Raft leader always has all committed entries. The election mechanism is a distributed proof that the winner holds the complete history.
4. **Byzantine Generals Framing** (§5 "Byzantine Fault Tolerance"): BFT requires 3f+1 nodes for f traitors because f traitors can frame f honest nodes, requiring an f+1 honest majority to outvote the coalition.

**The Insight Textbooks Miss** (from §6 "What Textbooks Don't Tell You"):
- **Pre-vote Extension**: Prevents disruptive elections when a partitioned node rejoins — without it, a recovered follower with a higher term forces a leader step-down even if the leader is healthy.
- **Leader Stickiness**: A candidate should not displace a leader it can hear from; this prevents spurious term increments from degrading throughput.
- **Read Linearizability costs**: Follower reads require either lease-based timestamps, ReadIndex RPC round-trips, or log reads — each has different tail latency and availability trade-offs not discussed in the original Raft paper.

**Real Mastery Time**: 3–4 weeks. Includes implementing Single-Node Raft, reproducing the Figure 8 scenario (§3 "Raft" — Figure 8 Danger), and understanding why a No-Op entry must be committed on leader election (§3 — Single-Node Raft and No-Op Entry).

**Cross-Lesson Connections**:
- **L02 §6 "Write-Ahead Logging"**: Raft's replicated log and WAL share the same invariant — the log record must be durable before the operation is visible.
- **L03 §6 "Lock-Free Data Structures"**: CAS primitives underpin Raft client retry loops and atomic state transitions in implementations.
- **L02 §5 "MVCC"**: Multi-version concurrency control at the database layer implements read linearizability (one of three approaches in §6 "What Textbooks Don't Tell You").

---

### Lesson 02 — Database Internals

**Core Insight**: Every database performance trade-off is a point on the write-amplification / read-amplification / space-amplification triangle — B+ Trees optimize for reads at the cost of write amplification; LSM Trees invert this, trading read amplification for write throughput.

**Hard Prerequisites**:
- Balanced tree data structures (B-trees)
- File I/O syscalls (read/write/fsync, O_DIRECT)
- Basic transaction concepts (ACID)

**Key Mental Models**:
1. **WAL Golden Rule** (§6 "Write-Ahead Logging" — The Golden Rule): Log records describing a change must reach durable storage *before* the data pages they describe. Violating this means a crash leaves orphaned dirty pages with no redo path.
2. **MVCC Visibility** (§5 "MVCC"): Rows have (xmin, xmax) transaction ID bounds. A snapshot sees a row if and only if xmin is committed before the snapshot and xmax is either absent or committed after. No locks needed for reads.
3. **Compaction as Garbage Collection** (§2 "LSM Trees" — Leveled vs. Size-Tiered Compaction): SSTable compaction is not optional maintenance — without it, read amplification grows as O(log N) SSTables accumulate, and space amplification approaches 2× (size-tiered) or 1.1× (leveled).
4. **ARIES Three Phases** (§6 "Write-Ahead Logging" — ARIES): Analysis finds dirty pages and active transactions at crash; Redo replays all logged operations forward; Undo rolls back uncommitted transactions. The algorithm is correct regardless of when the crash occurred.

**The Insight Textbooks Miss** (from §9 "What Textbooks Don't Tell You"):
- **Transaction ID Wraparound** (§5.7): PostgreSQL uses 32-bit transaction IDs in a modular ring. If VACUUM doesn't advance the horizon, rows become invisible when the counter wraps — the Sentry outage was this failure mode at scale.
- **HOT Updates**: When a row update touches only non-indexed columns, PostgreSQL chains the new version to the old heap tuple without touching index pages. The index entry remains valid, reducing write amplification significantly for workloads with narrow updates.
- **InnoDB vs. PostgreSQL MVCC** (§5.8): InnoDB stores old versions in a separate undo log; PostgreSQL stores all versions in the heap. InnoDB's approach means point-in-time reads walk the undo chain; PostgreSQL's means VACUUM must reclaim dead tuples from the heap.

**Real Mastery Time**: 4–5 weeks. Build a toy LSM (MemTable + SSTable merge), implement ARIES on paper for a crash scenario, and trace a PostgreSQL MVCC visibility decision in the source.

**Cross-Lesson Connections**:
- **L03 §3 "Memory Barriers"**: The fsync in WAL (§6) is a storage-level memory barrier — without it, the OS write buffer is not durable.
- **L04 §5 "Demand Paging"**: The buffer pool (§7 "Buffer Pool Management") can use mmap to delegate page eviction to the OS, but this conflicts with WAL's durability guarantees (O_DIRECT bypasses page cache entirely).
- **L06 §5 "Cache Coherence: MESI Protocol"**: B+ Tree node size (§1) is tuned to cache line boundaries; false sharing on B-Link Tree split/merge operations is a real performance concern.
- **L01 §3 "Raft"**: WAL and Raft log replication are structurally identical; the difference is that WAL replicates locally while Raft replicates across nodes.

---

### Lesson 03 — Memory Models and Lock-Free Programming

**Core Insight**: CPUs and compilers both reorder memory operations for performance; the C++11 memory model is a formal contract between the programmer and the hardware that specifies exactly which reorderings are permitted and which are not — violating this contract produces data races that are undefined behavior, not merely unexpected behavior.

**Hard Prerequisites**:
- L06 §5 "Cache Coherence: MESI Protocol" (invalidation queues cause the non-atomicity that makes memory models necessary)
- L06 §2 "Out-of-Order Execution: Tomasulo's Algorithm" (store buffers are the hardware mechanism behind write reordering)
- Basic concurrent programming (locks, threads)

**Key Mental Models**:
1. **Store Buffer / Invalidation Queue Split** (§1 "The Memory Illusion"): Stores go to a local store buffer before becoming visible to the cache coherence fabric; loads read from an invalidation queue that may not yet have applied all incoming invalidations. These two independent queues break the naive model of "shared memory = one array."
2. **Litmus Test as Contract** (§2 "CPU Memory Models"): The MP, SB, LB, and IRIW litmus tests are the canonical four experiments distinguishing x86 TSO from ARM/POWER. If you know which outcomes each architecture permits, you know the memory model.
3. **Happens-Before as the Correctness Criterion** (§4 "C++11 Memory Model"): A data race is two conflicting accesses with no happens-before edge between them. The six memory orders are mechanisms for creating happens-before edges.
4. **EBR Grace Period** (§6 "Lock-Free Data Structures" — Epoch-Based Reclamation): A retired node can be freed only when all threads have passed through a quiescent state observed *after* the retirement. The epoch counter is a distributed snapshot of "nobody is still reading the old generation."

**The Insight Textbooks Miss** (from §8 "What Textbooks Don't Tell You"):
- Python's GIL is *not* a memory model — it prevents concurrent bytecode execution but does not prevent CPU reordering within a Python thread or guarantee any ordering across threads accessing C extension shared state.
- `seq_cst` is not free: on x86, every seq_cst store becomes an `XCHG` (implicit LOCK prefix, ~30ns vs 4ns for a plain store). On ARM/POWER, it costs DMB barriers on both sides.
- NUMA effects (§8) make lock-free code's latency non-uniform: a CAS on a cacheline owned by a remote NUMA node costs 3–5× a local CAS, invalidating worst-case assumptions derived from single-socket benchmarks.

**Real Mastery Time**: 3–4 weeks. Requires writing correct acquire/release code, implementing the Treiber Stack and ABA fix with tagged pointers, and reasoning through the SB litmus test outcome on both x86 and ARM without hardware.

**Cross-Lesson Connections**:
- **L06 §5 "Cache Coherence: MESI Protocol"** and **L06 §11 "Memory Ordering, Store Buffers, and the x86 TSO Model"**: These two sections *are* the hardware explanation for why §1 "The Memory Illusion" happens.
- **L02 §6 "Write-Ahead Logging"**: The fsync durability contract is a storage-level happens-before edge; L03's framework explains why fsync + mfence must both be present for correct durable writes.
- **L05 §4 "Key Optimization Passes"**: Compiler optimizations (CSE in §4.3, DCE in §4.2, LICM in §4.5) all produce reorderings that a `volatile` keyword cannot prevent — only proper C++11 atomics create compiler barriers.
- **L07 §4 "Digital Signatures: ECDSA and EdDSA"**: Constant-time comparison requires `memory_order_seq_cst` loads to prevent speculative reads from creating timing oracles.

---

### Lesson 04 — OS and Virtual Memory

**Core Insight**: Virtual memory is not just address translation — it is the mechanism by which the OS implements isolation, overcommit, copy-on-write, NUMA topology, and security policies (ASLR, W^X) all through a single hardware-software interface: the page table walk.

**Hard Prerequisites**:
- Binary representation (hex addresses, bit fields)
- Basic OS concepts (processes, syscalls, file descriptors)
- L06 §2 "Out-of-Order Execution: Tomasulo's Algorithm" helps for understanding why TLB shootdown requires IPIs

**Key Mental Models**:
1. **Four-Level Walk Cost** (§2 "x86-64 Four-Level Page Tables"): A page table walk on a cold TLB costs 4 memory accesses (PGD → PUD → PMD → PTE) plus the final access — ~500ns on DDR4. TLB hits collapse this to ~1ns. The factor-of-500 gap explains why TLB occupancy determines performance more than cache hit rate for pointer-chasing workloads.
2. **CoW Fork as Deferred Copy** (§6 "Copy-on-Write and fork()"): After fork(), parent and child share all pages marked read-only. The first write to any page triggers a fault, allocates a fresh page, copies content, and marks it writable. Redis BGSAVE exploits this: the child takes a clean snapshot without blocking the parent.
3. **Huge Pages Collapse the Walk** (§4 "Huge Pages"): A 2MB PMD-level mapping replaces 512 4KB PTEs with one entry. TLB pressure drops 512×. Transparent Huge Pages (THP/khugepaged) deliver this automatically but cause latency spikes during compaction. MAP_HUGETLB grants explicit control.
4. **PCID Avoids Full TLB Flush** (§3 "Translation Lookaside Buffer"): PCID (bits 11:0 of CR3) tags TLB entries with an address-space identifier, allowing context switches to retain TLB entries from the outgoing process. KPTI (Meltdown mitigation) doubles the context switch cost without PCID.

**The Insight Textbooks Miss** (from §9 "What Textbooks Don't Tell You"):
- `struct page` is 64 bytes (one cache line) in Linux. With 4GB RAM and 4KB pages, that's 1M page structs = 64MB just for metadata. Huge pages reduce this proportionally.
- **Linux 6.1 Maple Tree**: The `vm_area_struct` list (VMA) was a red-black tree from 2.6 until 6.1, when it was replaced with a Maple Tree — a B-tree variant optimized for range operations and RCU-safe concurrent access.
- **Buddy Allocator orders 0–10**: The kernel buddy system allocates contiguous physical pages in powers-of-2 up to 4MB (order-10). Internal fragmentation is bounded; external fragmentation is the primary challenge for large contiguous allocations (needed by DMA and huge pages).
- **SLUB Allocator**: kmalloc uses SLUB, a per-CPU slab allocator that maintains per-CPU caches of fixed-size objects. Allocation is typically lock-free at the fast path; contention only occurs on the slow path when a new slab must be fetched from the buddy allocator.

**Real Mastery Time**: 3 weeks. Walk the page table of a running process with `/proc/$pid/maps` and `pagemap`, observe CoW page count changes during `fork()` under memory pressure, and instrument TLB shootdown IPIs with `perf`.

**Cross-Lesson Connections**:
- **L06 §10 "TLB/Address Translation"**: L06's §10 covers TLB structure and PCID from the microarchitecture perspective; L04 §3 covers the same from the OS perspective — read both together.
- **L03 §6 "Lock-Free Data Structures"**: Virtual address arithmetic in CAS loops requires understanding alignment guarantees from PTEs; the ABA tagged-pointer fix (§6, "ABA Problem") depends on virtual address upper bits being available.
- **L05 §8 "What Textbooks Don't Tell You"**: JIT compilers must make pages executable (W^X violation), requiring `mprotect()` with the exact page-table permission flags from L04 §2.
- **L02 §7 "Buffer Pool Management"**: The decision between `mmap` and `O_DIRECT` in database buffer pools is directly an L04 §5 "Demand Paging" trade-off.

---

### Lesson 05 — Compiler Internals

**Core Insight**: SSA form is the single most consequential data structure in modern compilers — by ensuring each variable has exactly one static definition, it makes most optimization problems tractable (constant propagation becomes a lattice fixpoint, liveness becomes a simple backward dataflow, interference becomes a graph coloring problem).

**Hard Prerequisites**:
- Graph theory: dominators, DFS, graph coloring (NP-complete for k≥3)
- Data structures: trees (for dominator computation), union-find (for SSA destruction)
- L06 §7 "SIMD and Instruction-Level Parallelism" (instruction selection targets SIMD units)

**Key Mental Models**:
1. **Dominance Frontier as Phi Placement Oracle** (§2 "Building SSA Form"): A phi function is needed at a join point J for variable v if and only if J is in the dominance frontier of a definition of v. The Cytron 1991 IDF algorithm computes this for all variables in O(|edges| × depth) time.
2. **SCCP Lattice** (§4 "Key Optimization Passes" §4.4): Each variable's value lives in the lattice ⊤ (not yet evaluated) → constant c → ⊥ (non-constant). Sparse conditional constant propagation propagates only when values change, visits conditional branches only when their operands leave ⊤, and achieves O(edges) complexity vs. naive O(nodes²).
3. **Interference Graph Coloring** (§5 "Register Allocation via Graph Coloring"): Two variables interfere if they are both live at any program point. k-coloring the interference graph with k = #registers gives a legal register assignment without spills; nodes that cannot be colored are spilled to the stack. Chaitin proved this is NP-complete in general.
4. **Instruction Selection as Tree Tiling** (§13 "Instruction Selection"): The BURG algorithm tiles an expression tree with target-ISA patterns (each pattern covers a subtree and has a cost). Dynamic programming finds the minimum-cost tiling in O(n) time. x86's complex addressing modes (`[rax + rbx*4 + 32]`) create large-gain patterns.

**The Insight Textbooks Miss** (from §8 "What Textbooks Don't Tell You"):
- **SSA Destruction**: Converting out of SSA requires handling lost-copy and swap problems. The naive "replace phi with copies" approach introduces incorrect programs when swap patterns arise; the correct fix uses a sequentialization algorithm before copy placement.
- **Alias Analysis Limits** (§8): Both Andersen's (flow-insensitive, set-based) and Steensgaard's (near-linear, union-find) alias analyses are *conservative* — they may report aliases that do not exist, blocking LICM and CSE on pointer-heavy code. This is why C's `restrict` keyword exists.
- **Inlining as Meta-Optimization** (§8): Inlining does not directly reduce code; it *enables* subsequent passes. A call site prevents constant propagation across the call boundary; inlining exposes the callee's body to the caller's constants, allowing SCCP to eliminate branches and DCE to remove dead parameters.
- **Optimizer Cliff** (§8): PGO (Profile-Guided Optimization) can dramatically change which branches are predicted taken and which loops are unrolled. A function that is fast under PGO may be 3× slower without it, creating invisible performance cliffs when benchmark profiles diverge from production.

**Real Mastery Time**: 5–6 weeks. Implement SSA construction (Cytron IDF algorithm from §2), implement SCCP (§4.4 lattice), implement graph-coloring register allocation (Chaitin §5.3), and compare LLVM IR before/after `-O2` on a non-trivial function (§7 "LLVM IR").

**Cross-Lesson Connections**:
- **L06 §2 "Out-of-Order Execution: Tomasulo's Algorithm"**: The RAT (Register Alias Table) + PRF (Physical Register File) in Tomasulo is hardware register renaming — the runtime dual of Chaitin's compile-time register allocation (§5 "Register Allocation"). Both solve the false dependence problem; one in software, one in hardware.
- **L04 §9 "What Textbooks Don't Tell You"**: W^X page protection (L04) is the OS constraint that JIT compilers must work around — `mprotect(PROT_EXEC)` requires understanding both the OS page table (L04 §2) and the compiler's code generation pipeline.
- **L03 §4 "C++11 Memory Model"**: The compiler optimizer is free to reorder loads/stores in the absence of synchronization. `std::atomic` with `memory_order_acquire/release` is how the programmer inserts compiler barriers that block the reorderings in §4 "Key Optimization Passes."
- **L06 §7 "SIMD and Instruction-Level Parallelism"**: §13 "Instruction Selection" targets SIMD execution units; auto-vectorization (SLP vectorization in LLVM) is an optimization pass that detects parallel scalar operations and replaces them with SIMD instructions.

---

### Lesson 06 — CPU Microarchitecture

**Core Insight**: Modern CPUs are speculative, out-of-order, superscalar machines that commit results in-order while executing them out-of-order — the Reorder Buffer (ROB) is the mechanism that reconciles speculative execution with the programmer's sequential semantics, and its size determines how far ahead the CPU can look to find independent instructions.

**Hard Prerequisites**:
- Digital logic (pipeline stages, registers, multiplexers)
- Assembly language basics (registers, load/store, arithmetic)
- L04 §2 "x86-64 Four-Level Page Tables" and §3 "TLB" (for §10 "TLB/Address Translation")

**Key Mental Models**:
1. **ROB as Latency-Hiding Buffer** (§8 "What Textbooks Don't Tell You"): Maximum parallelism = ROB_size / average_latency. With a 352-entry ROB (Ice Lake) and an average load latency of 4 cycles, the CPU can overlap 88 independent loads. If a single high-latency operation (200-cycle LLC miss) stalls retirement, it consumes 200/4 = 50 ROB entries, leaving only 302 for other work.
2. **TAGE Predictor** (§3 "Branch Prediction"): TAGE uses multiple tagged tables indexed by geometric-length history (4, 8, 16, 32, 64 bits). Longer history tables are consulted first; a hit in a longer-history table overrides shorter-history predictions. This allows TAGE to predict patterns requiring thousands of branches of context.
3. **False Sharing** (§5 "Cache Coherence: MESI Protocol"): Two variables in the same 64-byte cache line, written by different threads, force the line to bounce between Modified states across cores even though neither thread is accessing the other's variable. The fix is padding to cache-line alignment.
4. **Store-to-Load Forwarding** (§8 "What Textbooks Don't Tell You"): A store that has not yet reached L1 cache can be forwarded directly to a load that reads the same address from the store buffer — bypassing the cache entirely. This forwarding has a fixed ~5-cycle latency; if the addresses only partially overlap, forwarding fails and latency jumps to ~13 cycles.

**The Insight Textbooks Miss** (from §8 "What Textbooks Don't Tell You"):
- **Power Gating and Frequency Boost**: Cores not executing workloads are power-gated. Turbo Boost increases the active core's frequency by reclaiming that thermal budget. Sequential single-threaded workloads often run *faster* with fewer threads because the active core runs at higher frequency.
- **Spectre v1** (§3): PHT (Pattern History Table) poisoning by an attacker causes the victim's speculative execution to read out-of-bounds memory. The mis-speculated load leaves a side channel in the cache (L1 timing difference) that leaks one bit per gadget invocation. RETPOLINE mitigates BTB-based Spectre v2 but not PHT-based Spectre v1.
- **SMT Covert Channel** (§12 "SMT/Hyper-Threading"): Hyperthreads share execution ports and the ROB. A thread can measure execution port pressure to infer what the sibling is computing — a microarchitectural covert channel that persists even with OS-level process isolation.

**Real Mastery Time**: 4 weeks. Use `perf stat` to measure IPC, branch misprediction rate, LLC misses, and TLB shootdowns on a real workload; correlate with ROB-size calculations from §8.

**Cross-Lesson Connections**:
- **L03 §1 "The Memory Illusion"** and **L03 §2 "CPU Memory Models"**: The store buffers and invalidation queues described in L03 §1 are the *same* store buffers described in L06 §11 "Memory Ordering, Store Buffers, and the x86 TSO Model." Read both for the complete picture.
- **L05 §5 "Register Allocation via Graph Coloring"**: L06's RAT + PRF is hardware register renaming; L05's Chaitin is software register allocation. Both eliminate false dependencies (WAR/WAW hazards). Read L06 §2 immediately after L05 §5.
- **L04 §3 "Translation Lookaside Buffer"** / **L04 §4 "Huge Pages"**: L06 §10 "TLB/Address Translation" re-examines TLBs from the pipeline perspective; L04 provides the OS and software perspective. Together they explain KPTI overhead.
- **L07 §7 "SHA-256 Internals: The Compression Function"**: SHA-256's 64 rounds are a perfect target for §7 "SIMD and Instruction-Level Parallelism" — Intel's SHA-NI extensions implement the compression function in dedicated execution units.

---

### Lesson 07 — Cryptography Internals

**Core Insight**: Elliptic-curve cryptography derives its security from the asymmetry between scalar multiplication (easy: O(log n) point doublings) and the discrete logarithm problem (believed hard: best classical algorithm is O(√p) group operations) — and the entire modern cryptographic stack (ECDH, ECDSA, EdDSA, TLS 1.3) is built on this single asymmetry.

**Hard Prerequisites**:
- Modular arithmetic (Fermat's little theorem for modular inverse)
- Group theory basics (cyclic groups, generators, order)
- Probability (birthday paradox for Pollard's rho)

**Key Mental Models**:
1. **Point at Infinity as Identity** (§1 "Elliptic Curves from First Principles"): The group law requires an identity element. The point at infinity O is this element: P + O = P for all P. Every negation P + (-P) = O. This makes the set of curve points a proper abelian group.
2. **Nonce Reuse Is Fatal** (§4 "Digital Signatures: ECDSA and EdDSA" — Sony PS3): If ECDSA signing uses the same nonce k for two messages, the private key is recoverable via linear algebra in two operations. Sony used k=1 for all PlayStation 3 game signatures. RFC 6979 deterministic k (HMAC-DRBG of the private key and message hash) eliminates this attack class entirely.
3. **Fiat-Shamir as Random Oracle Replacement** (§5 "Zero-Knowledge Proofs: Schnorr Protocol"): An interactive ZKP (prover sends commitment R, verifier sends challenge c, prover responds with s) can be made non-interactive by replacing the verifier's random challenge with H(R ‖ message). Security relies on the hash being a random oracle — modeled but not provably true for SHA-256.
4. **HKDF as Two-Phase Key Derivation** (§3 "ECDH Key Exchange"): Extract phase (HMAC-SHA-256 with salt) compresses the shared secret into uniform randomness; Expand phase (iterative HMAC with counter and context) stretches it into keying material of arbitrary length. TLS 1.3's key schedule (§6 "TLS 1.3 Handshake") applies HKDF three times (Handshake Secret, Master Secret, Traffic Secret).

**The Insight Textbooks Miss**:
- **Signature Malleability** (§4): In ECDSA, if (r, s) is a valid signature, so is (r, -s mod n). Bitcoin transactions were malleable until BIP 66 mandated DER encoding with low-s normalization. EdDSA (§4 — EdDSA) is non-malleable by construction.
- **MuSig2 Rogue-Key Attack** (§5 "Zero-Knowledge Proofs: Schnorr Protocol" — MuSig2): In naive Schnorr multi-signature, a malicious signer can choose their public key as K_attacker = K_target × K_1^{-1}, making the aggregate key equal to their own. MuSig2 prevents this via key commitment: all signers must commit to their public keys before seeing others'.
- **0-RTT Replay** (§6 "TLS 1.3 Handshake"): TLS 1.3's 0-RTT mode sends application data in the first flight using a pre-shared key. The server cannot distinguish a replayed 0-RTT record from a fresh one without application-level sequencing. Only idempotent operations (GET) are safe in 0-RTT.
- **EdDSA Montgomery Ladder** (§4): The Montgomery ladder computes scalar multiplication in constant time — the same number of point operations regardless of the bit pattern of the scalar. This eliminates timing side channels that afflict naive double-and-add when scalar bits are tested and the "double only" path is faster than "double-and-add."

**Real Mastery Time**: 4–5 weeks. Implement secp256k1 point addition from scratch, implement ECDSA with RFC 6979 nonce generation, verify the PS3 key recovery derivation algebraically, and trace one complete TLS 1.3 handshake in Wireshark against a test server.

**Cross-Lesson Connections**:
- **L03 §4 "C++11 Memory Model"**: Constant-time comparison (`crypto_verify_32`) requires `memory_order_seq_cst` to prevent speculative loads from leaking via cache timing — directly applies L03's ordering guarantees.
- **L06 §7 "SIMD and Instruction-Level Parallelism"**: SHA-256 (§7 "SHA-256 Internals") and AES-GCM (mentioned in §6 TLS cipher suite breakdown) exploit SIMD — Intel SHA-NI and AES-NI are dedicated execution units for the respective compression functions.
- **L05 §4 "Key Optimization Passes"** — §4.2 DCE: Compilers may eliminate "dead" stores of zeroed key material if the result is never used. Defensive code must use `memset_s` (or platform equivalent) to prevent DCE from removing security-critical zeroing.
- **L01 §5 "Byzantine Fault Tolerance"**: HotStuff and Tendermint (§5 in L01) use threshold signatures (Schnorr/BLS) for aggregate BFT certificates — directly built on the Schnorr linearity property from L07 §5.

---

## 4. Concept Cross-Reference Table

| Concept | Lessons | Specific Sections | Why It Recurs |
|---------|---------|-------------------|---------------|
| Compare-and-Swap (CAS) | L03, L01 | L03 §6 "Lock-Free Data Structures"; L01 §3 "Raft" (leader election CAS) | CAS is the universal primitive for building both lock-free data structures (Treiber Stack, MS Queue) and distributed atomic state transitions. It recurs because it is the minimal synchronization instruction available on all commodity hardware. |
| Memory Barriers / Fences | L03, L06, L02, L07 | L03 §3 "Memory Barriers"; L06 §11 "Memory Ordering, Store Buffers, and the x86 TSO Model"; L02 §6 "Write-Ahead Logging"; L07 §4 "ECDSA and EdDSA" (constant-time) | Any system that communicates between threads, processes, or nodes must order stores and loads. The barrier is the mechanism; it recurs in lock-free code, WAL durability, and constant-time crypto. |
| Write-Ahead Logging (WAL) | L02, L01 | L02 §6 "Write-Ahead Logging (WAL)"; L01 §3 "Raft" (replicated log) | WAL is the universal durability primitive: the log record must be durable before the effect it describes. In databases (L02) this is local; in Raft (L01) it is replicated. Same invariant, different scope. |
| MESI Cache Coherence | L06, L03 | L06 §5 "Cache Coherence: MESI Protocol"; L03 §1 "The Memory Illusion" | MESI is the hardware mechanism that implements shared memory. Store buffers (L03 §1) exist because MESI state transitions have latency; false sharing (L06 §5) is a consequence of cache-line granularity in the MESI protocol. |
| Epoch-Based Reclamation (EBR) | L03 | L03 §6 "Lock-Free Data Structures" (Epoch-Based Reclamation) | Safe memory reclamation in lock-free data structures without GC. Epoch counters track quiescent states; retired nodes are freed only after all threads have observed the epoch advance. Related to MVCC snapshot isolation in L02. |
| Snapshot Isolation / MVCC | L02, L01 | L02 §5 "MVCC"; L01 §6 "What Textbooks Don't Tell You" (read linearizability via multi-version reads) | Both database and distributed systems need reads that see a consistent state without blocking concurrent writers. MVCC achieves this by maintaining multiple row versions (L02) or reading from a committed log index (L01). |
| Dominance / Dominators | L05 | L05 §2 "Building SSA Form" (Dominators, Dominance Frontier, IDF) | Dominator trees are the structural backbone of SSA construction (phi placement), loop detection (LICM), and control-dependence graphs (PDG). Every flow-sensitive optimization uses dominators. |
| Register Renaming | L05, L06 | L05 §5 "Register Allocation via Graph Coloring"; L06 §2 "Out-of-Order Execution: Tomasulo's Algorithm" (RAT/PRF) | False dependencies (WAR/WAW hazards) block both compile-time optimization and runtime scheduling. Chaitin (L05) removes them statically; Tomasulo's RAT (L06) removes them dynamically. Both allocate from a pool of physical names. |
| TLB and Address Translation | L04, L06 | L04 §3 "Translation Lookaside Buffer"; L06 §10 "TLB/Address Translation" | Every memory access begins with address translation. TLB performance determines the practical cost of pointer-chasing workloads. PCID (L04 §3) and KPTI (L06 §10) are the OS/hardware split for Meltdown mitigation. |
| Huge Pages | L04, L06 | L04 §4 "Huge Pages"; L06 §10 "TLB/Address Translation" | Huge pages reduce TLB pressure by covering 512× more address space per TLB entry. This affects database buffer pools, JVM heaps, and any workload with large working sets. THP (L04) automates this; MAP_HUGETLB gives explicit control. |
| Spectre / Side Channels | L06, L07, L04 | L06 §3 "Branch Prediction" (Spectre v1); L07 §4 "ECDSA and EdDSA" (timing side channels); L04 §9 "What Textbooks Don't Tell You" (ASLR, W^X) | Speculative execution and timing differences create information channels that bypass software isolation. PHT poisoning (L06) leaks cache state; timing-variable scalar multiplication (L07) leaks key bits; ASLR (L04) raises the cost of exploit development. |
| Scalar Multiplication | L07 | L07 §1 "Elliptic Curves from First Principles"; §4 "ECDSA and EdDSA" (Montgomery ladder) | EC scalar multiplication (double-and-add) is the operation at the heart of all ECC. Its variable-time naive form is exploitable (L07 §4); the Montgomery ladder is constant-time. Quantum hardness (Shor's algorithm, L07 §1) will break it post-quantum. |
| Phi Functions (SSA) | L05 | L05 §2 "Building SSA Form" | Phi functions are the merge operator in SSA form: they select among incoming values based on control flow. Placing them correctly (IDF algorithm) is the hard problem in SSA construction. All downstream optimizations (SCCP, liveness, interference) depend on correct phi placement. |
| Bloom Filter | L02 | L02 §4 (Bloom Filter + SkipList) | LSM Trees use Bloom filters to avoid reading SSTables that cannot contain a key. A false positive causes a wasted disk read; false negatives are impossible. Tuning the false positive rate vs. memory is an engineering decision embedded in every LSM-based storage engine. |
| Copy-on-Write (CoW) | L04, L02 | L04 §6 "Copy-on-Write (CoW) and fork()"; L02 §5 "MVCC" (row-level CoW via xmin/xmax) | CoW defers the cost of making a copy until a write occurs. OS fork() (L04) applies it at page granularity; PostgreSQL MVCC (L02) applies it at row granularity. Both trade read-amplification for write isolation. |
| False Sharing | L06, L03 | L06 §5 "Cache Coherence: MESI Protocol"; L03 §8 "What Textbooks Don't Tell You" (NUMA effects) | False sharing causes cache-line bouncing between Modified states with no logical sharing of data. It is invisible to the programmer (variables appear independent) but catastrophic to performance. Padding to 64-byte alignment is the canonical fix. |
| Dataflow Analysis | L05 | L05 §4 "Key Optimization Passes" (§4.5 LICM — natural loops); §5 §5.1 "Liveness" (backward dataflow) | Dataflow frameworks (forward for reaching definitions/constant propagation, backward for liveness/use-def) underpin most scalar optimizations. The worklist algorithm is the canonical iterative fixpoint solver. |
| Nonce / Randomness | L07, L01 | L07 §4 "ECDSA and EdDSA" (RFC 6979, PS3 nonce reuse); §5 "Schnorr Protocol" (Fiat-Shamir random oracle) | Randomness failure is catastrophic in cryptography: deterministic nonces (RFC 6979) and commitment schemes (MuSig2) exist precisely because entropy failures have repeatedly broken production systems. The Fiat-Shamir transform (L07 §5) is sound only under the random oracle model. |
| Log-Structured Storage | L02, L01 | L02 §2 "LSM Trees"; L01 §3 "Raft" (append-only replicated log) | Sequential writes are 10–100× faster than random writes on both HDD and SSD. LSM Trees (L02) and Raft's log (L01) both exploit this by making all writes append-only and deferring reorganization (compaction / snapshotting). |
| Quorum / Majority | L01 | L01 §2 "Paxos from First Principles" (Quorum Intersection); §3 "Raft" (Commitment Rule §5.4.1) | Any two quorums in a majority-quorum system share at least one node. This shared node is the bridge that prevents split-brain. Flexible Paxos (L01 §2) generalizes: read quorum × write quorum > N is the invariant, allowing asymmetric quorum sizes. |
| Compaction / Garbage Collection | L02, L03 | L02 §2 "LSM Trees" (Leveled vs. Size-Tiered Compaction); L03 §6 "Lock-Free Data Structures" (EBR as GC) | Deferred cleanup is a pattern: LSM compaction reclaims obsolete SSTables; EBR reclaims retired lock-free nodes; PostgreSQL VACUUM reclaims dead tuple versions. All require tracking "who might still be reading" before freeing. |
| ARIES Recovery | L02 | L02 §6 "Write-Ahead Logging (WAL)" (ARIES Analysis/Redo/Undo phases) | ARIES is the theoretical foundation for crash recovery in all major relational databases (PostgreSQL, MySQL InnoDB, SQL Server). Analysis → Redo → Undo is the only correct ordering; applying them out of order produces inconsistency. |
| Branch Prediction | L06, L05 | L06 §3 "Branch Prediction" (TAGE, BTB, Spectre v1); L05 §8 "What Textbooks Don't Tell You" (PGO) | Branch misprediction flushes the pipeline (15–20 cycle penalty on modern CPUs). Predictors (2-bit, Two-Level Adaptive, TAGE) are the hardware countermeasure; PGO (L05 §8) is the compiler-time countermeasure. Spectre v1 (L06 §3) weaponizes the predictor. |
| Happens-Before | L03, L01 | L03 §4 "C++11 Memory Model"; L01 §3 "Raft" (log commitment creates happens-before on operations) | Happens-before is the fundamental partial order for reasoning about concurrent correctness. In C++11 (L03), it is defined by synchronization operations. In Raft (L01), log commitment defines a total order on client operations that is a happens-before extension. |
| Inlining and Interprocedural Optimization | L05 | L05 §8 "What Textbooks Don't Tell You" (inlining as meta-optimization) | Inlining is not itself an optimization; it is an enabler. It exposes callee bodies to the caller's constants (enabling SCCP), eliminates call overhead (enabling register allocation across the call site), and enables devirtualization. The danger is code size explosion reducing I-cache effectiveness. |
| Zero-Knowledge Proofs | L07, L01 | L07 §5 "Zero-Knowledge Proofs: Schnorr Protocol"; L01 §5 "Byzantine Fault Tolerance" (threshold signatures in HotStuff) | ZKPs prove knowledge without revealing the secret. Schnorr (L07) is the simplest example; its linearity enables MuSig2 aggregation and BFT threshold signatures (L01). The Fiat-Shamir transform makes interactive ZKPs non-interactive. |
| SIMD / Vectorization | L06, L07, L05 | L06 §7 "SIMD and Instruction-Level Parallelism"; L07 §7 "SHA-256 Internals" (SHA-NI); L05 §13 "Instruction Selection" | SIMD parallelizes the same operation over a vector of data elements. At the hardware level (L06), AVX-512 provides 16-wide float32 execution. At the compiler level (L05), SLP auto-vectorization detects eligible patterns. Cryptographic primitives (L07) exploit dedicated SIMD extensions (SHA-NI, AES-NI). |
| Spill Code / Register Pressure | L05, L06 | L05 §5 "Register Allocation via Graph Coloring" (§5.4 Spill Code); L06 §2 "Tomasulo's Algorithm" (280-entry PRF) | When more live variables exist than physical registers (L05) or more in-flight instructions exist than ROB/RS entries (L06), values must be spilled to memory. Spill code in L05 generates extra loads/stores; ROB overflow in L06 stalls the front-end. Both are forms of resource pressure. |
| Lattice / Fixpoint Iteration | L05 | L05 §4 "Key Optimization Passes" §4.4 "SCCP" (TOP/CONST/BOT lattice) | Lattice-based fixpoint iteration is the theoretical basis for all dataflow analyses (liveness, reaching definitions, constant propagation, alias analysis). SCCP (L05 §4.4) is the canonical example: the lattice provides a bounded descent that guarantees termination and precision. |

---

## 5. What Comes Next

### Lesson 01 — Distributed Consensus

**Papers**:
1. Lamport, L. (1998). "The Part-Time Parliament." *ACM Transactions on Computer Systems*, 16(2), 133–169. The original Paxos paper; deceptively simple, endlessly subtle. Read after completing L01 §2 "Paxos from First Principles."
2. Ongaro, D. & Ousterhout, J. (2014). "In Search of an Understandable Consensus Algorithm (Extended Version)." *USENIX ATC 2014*. The Raft paper; Figure 8 and the cluster membership section are mandatory.
3. Abraham, I. & Malkhi, D. (2017). "The Blockchain Consensus Layer and BFT." *BEATCS*. Connects L01 §5 "Byzantine Fault Tolerance" to modern BFT protocols (HotStuff, Tendermint) and blockchain consensus.

**OSS Projects**:
- **etcd** (`github.com/etcd-io/etcd`): Production Raft implementation in Go. Study `raft/raft.go` for the state machine and `raft/log.go` for the replicated log — directly maps to L01 §3 "Raft" Log Structure and Commitment Rule.
- **TiKV** (`github.com/tikv/tikv`): Distributed key-value store with Multi-Raft (one Raft group per region). Illustrates Cluster Membership Changes from L01 §3 at scale.

**What This Series Does Not Cover**:
- **Viewstamped Replication** (Liskov & Cowling 2012) — predates Paxos in some respects
- **CRDTs** (Conflict-free Replicated Data Types) — eventual consistency without consensus
- **Geo-distributed consensus** (Spanner TrueTime, clock uncertainty as a physical quorum)
- **Reconfiguration under live traffic** beyond the joint-consensus outline in L01 §3

---

### Lesson 02 — Database Internals

**Papers**:
1. Mohan, C. et al. (1992). "ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging." *ACM TODS*, 17(1), 94–162. The definitive reference for L02 §6 "Write-Ahead Logging (WAL)" — Analysis/Redo/Undo phases.
2. O'Neil, P. et al. (1996). "The Log-Structured Merge-Tree (LSM-Tree)." *Acta Informatica*, 33(4), 351–385. The original LSM paper; compare to L02 §2 "LSM Trees" for the historical context.
3. Neumann, T. & Weikum, G. (2010). "The RUM Conjecture." (See also: Idreos, S. et al. (2016). "Designing Access Methods: The RUM Conjecture." *EDBT 2016*.) Formalizes the read/update/memory amplification trade-off that underlies all of L02.

**OSS Projects**:
- **RocksDB** (`github.com/facebook/rocksdb`): The canonical production LSM implementation. Study `db/compaction/` for Leveled vs. Size-Tiered compaction from L02 §2, and `db/memtable/` for SkipList MemTable.
- **PostgreSQL** source (`github.com/postgres/postgres`): `src/backend/storage/smgr/md.c` (buffer manager), `src/backend/access/heap/heapam.c` (MVCC visibility), `src/backend/access/transam/xlog.c` (WAL). All directly illuminate L02 §5, §6, §7.

**What This Series Does Not Cover**:
- **Column stores** (Vectorwise, MonetDB, DuckDB) and vectorized query execution
- **Query optimization** (Volcano/Cascades planner, cardinality estimation, join ordering)
- **Distributed transactions** (2PC, 2PL, MVTO across shards)
- **NewSQL** architectures (Spanner F1, CockroachDB) that combine L01 consensus with L02 storage

---

### Lesson 03 — Memory Models and Lock-Free Programming

**Papers**:
1. Sewell, P. et al. (2010). "x86-TSO: A Rigorous and Usable Programmer's Model for x86 Multiprocessors." *CACM*, 53(7), 89–97. The formal model behind L03 §2 "CPU Memory Models" — x86 TSO is defined here precisely.
2. Herlihy, M. (1991). "Wait-Free Synchronization." *ACM TOPLAS*, 13(1), 124–149. The theoretical foundation for lock-free and wait-free data structures in L03 §6 "Lock-Free Data Structures."
3. McKenney, P.E. & Slingwine, J.D. (1998). "Read-Copy Update: Using Execution History to Solve Concurrency Problems." *PDCS 1998*. The original RCU paper; directly extends L03 §7b "RCU: Read-Copy-Update."

**OSS Projects**:
- **Linux kernel RCU** (`kernel/rcu/`): Production RCU implementation. `kernel/rcu/tree.c` is the full Tree RCU (L03 §7b). `Documentation/RCU/` explains quiescent states and grace periods.
- **Folly** (`github.com/facebook/folly`): Facebook's C++ library. `folly/concurrency/` has production-quality lock-free queues and hazard pointer implementations (L03 §6 "Hazard Pointers").

**What This Series Does Not Cover**:
- **Transactional Memory** (HTM via TSX/RTM, STM) — optimistic concurrency at the hardware level
- **Formal verification** of lock-free algorithms (TLA+, Iris/Coq proofs)
- **GPU memory models** (CUDA/PTX weak consistency) — significantly weaker than ARM/POWER
- **Persistent memory** (Intel Optane) — requires reasoning about cache-line persistence, not just ordering

---

### Lesson 04 — OS and Virtual Memory

**Papers**:
1. Bhatt, S. & Larus, J. (1992). "Measuring the Effects of Data and Demand-Driven Compilation." *(For TLB impact)*. More directly: Banga, G. & Druschel, P. (1999). "Measuring the Capacity of a Web Server Under Realistic Loads." *USENIX OSDI* — demonstrates working-set / TLB interaction in production workloads.
2. Gorman, M. (2004). *Understanding the Linux Virtual Memory Manager*. Prentice Hall. The definitive reference for L04 §9 "What Textbooks Don't Tell You" — Buddy allocator, SLUB, vm_area_struct, struct page.
3. Lipp, M. et al. (2018). "Meltdown: Reading Kernel Memory from User Space." *USENIX Security 2018*. The attack that required KPTI (L04 §3 "TLB" — PCID) and L06 §10.

**OSS Projects**:
- **Linux kernel mm/** (`github.com/torvalds/linux/tree/master/mm`): `mm/memory.c` (page fault handler, CoW — L04 §6), `mm/mmap.c` (VMA management, Maple Tree in 6.1+ — L04 §9), `mm/hugetlb.c` (huge pages — L04 §4).
- **jemalloc** (`github.com/jemalloc/jemalloc`): Production userspace allocator. Contrast with SLUB (L04 §9) — jemalloc operates in userspace using mmap; SLUB operates in kernel using the buddy allocator.

**What This Series Does Not Cover**:
- **eBPF** as an OS extension mechanism — interacts deeply with L04 page tables and L06 JIT
- **io_uring** — asynchronous I/O that bypasses traditional page fault paths
- **Persistent memory** PMDK and DAX (direct access to non-volatile byte-addressable storage)
- **Hypervisor / VM memory management** (EPT/NPT — nested page tables, balloon drivers)

---

### Lesson 05 — Compiler Internals

**Papers**:
1. Cytron, R. et al. (1991). "Efficiently Computing Static Single Assignment Form and the Control Dependence Graph." *ACM TOPLAS*, 13(4), 451–490. The paper that defines the IDF algorithm in L05 §2 "Building SSA Form." Every SSA-based compiler implements this.
2. Chaitin, G. et al. (1981). "Register Allocation via Coloring." *Computer Languages*, 6(1), 47–57. The foundational paper for L05 §5 "Register Allocation via Graph Coloring." NP-completeness proof + Chaitin's algorithm.
3. Cooper, K., Harvey, T. & Kennedy, K. (2001). "A Simple, Fast Dominance Algorithm." *TR-06*, Rice University. The O(n²) algorithm used in L05 §3 Python SSA Construction (SSABuilder class). Simple to implement correctly, suitable for most compiler frontends.

**OSS Projects**:
- **LLVM** (`github.com/llvm/llvm-project`): The reference implementation for L05 §7 "LLVM IR." Study `llvm/lib/Transforms/Scalar/SCCP.cpp` (SCCP — §4.4), `llvm/lib/CodeGen/RegAllocGreedy.cpp` (register allocation — §5.3), `llvm/lib/Analysis/DominanceFrontier.cpp` (IDF — §2).
- **GraalVM/Graal** (`github.com/oracle/graal`): JIT compiler for JVM. Illustrates how L05's static optimizations interact with L04's W^X page management and L03's GC write barriers in a production JIT.

**What This Series Does Not Cover**:
- **Garbage collector design** (tri-color marking, generational GC, concurrent GC write barriers)
- **Type systems and type inference** (Hindley-Milner, System F, dependent types)
- **Verified compilers** (CompCert — a formally verified C compiler in Coq)
- **Link-time optimization (LTO)** and whole-program analysis beyond what alias analysis covers

---

### Lesson 06 — CPU Microarchitecture

**Papers**:
1. Tomasulo, R.M. (1967). "An Efficient Algorithm for Exploiting Multiple Arithmetic Units." *IBM Journal of R&D*, 11(1), 25–33. The original paper defining the algorithm in L06 §2. Only 9 pages; every page is essential.
2. Seznec, A. & Michaud, P. (2006). "A Case for (Partially) TAgged GEometric History Length Branch Prediction." *Journal of Instruction-Level Parallelism*, 8, 1–23. Defines TAGE (L06 §3 "Branch Prediction"). The current state-of-the-art predictor design.
3. Kocher, P. et al. (2019). "Spectre Attacks: Exploiting Speculative Execution." *IEEE S&P 2019*. Defines PHT-based Spectre v1 (L06 §3) and BTB-based Spectre v2. Mandatory reading after L06 §3 and L07 §4.

**OSS Projects**:
- **gem5** (`github.com/gem5/gem5`): Cycle-accurate CPU simulator. Can model OOO pipelines (L06 §2 Tomasulo), TAGE branch predictors (L06 §3), and MESI coherence (L06 §5) with configurable parameters. Invaluable for microarchitecture research.
- **Intel VTune / Linux `perf`** (built-in): `perf stat -e instructions,cycles,branch-misses,cache-misses,dtlb-loads` on a real workload connects L06's theory to measurable hardware events. Not a code repo but an essential tool.

**What This Series Does Not Cover**:
- **GPU microarchitecture** (warp scheduling, SIMT, memory coalescing)
- **Memory controller / DRAM timing** (tRCD, tCL, tRP, row buffer hits)
- **Network-on-chip (NoC)** for many-core systems (mesh topologies, routing algorithms)
- **RISC-V implementation** details — the series focuses on x86 and ARM

---

### Lesson 07 — Cryptography Internals

**Papers**:
1. Bernstein, D.J. & Lange, T. (2007). "Faster Addition and Doubling on Elliptic Curves." *ASIACRYPT 2007*. Defines extended twisted Edwards coordinates (L07 §4 — EdDSA). The math behind Ed25519's complete addition formulas.
2. Rescorla, E. (2018). RFC 8446 — "The Transport Layer Security (TLS) Protocol Version 1.3." *IETF*. The normative specification for L07 §6 "TLS 1.3 Handshake." Every cipher suite, key schedule, and 0-RTT detail is here.
3. Nick, J., Ruffing, T. & Seurin, Y. (2021). "MuSig2: Simple Two-Round Schnorr Multi-Signatures." *CRYPTO 2021*. The formal security proof for MuSig2 (L07 §5 "Schnorr Protocol" — MuSig2). Proves security in the algebraic group model under the AOMDL assumption.

**OSS Projects**:
- **libsecp256k1** (`github.com/bitcoin-core/secp256k1`): Bitcoin's elliptic curve library. `src/group_impl.h` implements point addition (L07 §1) with constant-time Montgomery ladder (L07 §4 — EdDSA Montgomery Ladder). `src/schnorr_impl.h` implements BIP-340 Schnorr (L07 §5).
- **BoringSSL** (`github.com/google/boringssl`): Google's TLS library (used in Chrome). `ssl/tls13_*.cc` implements the TLS 1.3 key schedule (L07 §6) and 0-RTT logic. `crypto/fipsmodule/ec/` implements ECDSA/EdDSA (L07 §4).

**What This Series Does Not Cover**:
- **Post-quantum cryptography** (CRYSTALS-Kyber/Dilithium, NTRU, lattice-based constructions) — Shor's algorithm (L07 §1) breaks ECC; this series only names the threat
- **Pairing-based cryptography** (BLS signatures, zk-SNARKs, KZG polynomial commitments)
- **Symmetric cryptography internals** (AES SPN structure, AES-GCM, ChaCha20-Poly1305)
- **Formal verification of cryptographic protocols** (ProVerif, CryptoVerif, EasyCrypt)
- **Hardware Security Modules (HSMs)** and key management infrastructure

---

### What the 7-Lesson Series as a Whole Does NOT Cover

**Missing Domains**:
1. **Networking stack internals**: TCP congestion control (CUBIC, BBR), kernel bypass (DPDK, io_uring), NIC offload, RDMA. The series stops at the host CPU.
2. **Machine learning systems**: GPU kernels, distributed training (AllReduce, NCCL), mixed-precision arithmetic, model parallelism. Only L06 §7 touches SIMD, which is the scalar tip of the ML iceberg.
3. **Operating system design beyond virtual memory**: Scheduler algorithms (CFS, EEVDF), interrupt handling, device drivers, file system internals (ext4 journaling, ZFS CoW), container namespaces and cgroups.
4. **Programming language theory**: Type systems, lambda calculus, operational semantics, denotational semantics, proofs of type safety. L05 covers the optimization backend but not the frontend type system.
5. **Formal methods**: TLA+ for distributed systems, separation logic for memory safety proofs, model checking (SPIN), SMT solvers (Z3). The series reasons informally about correctness.
6. **Persistent memory and storage hardware**: NVMe protocol, SSD FTL (Flash Translation Layer), wear leveling, Optane/PMem byte-addressable persistence. L02 assumes block storage.
7. **Security engineering beyond primitives**: Secure enclaves (SGX/TrustZone), supply chain security, binary hardening (CFI, shadow stacks), fuzzing and symbolic execution.

**Missing Depth**:
- All 7 lessons use Python for illustrations. Production systems are C/C++/Rust/Go/Java. The gap between illustrative Python and production-grade concurrent code is significant.
- The series is single-machine or single-datacenter in orientation. Geo-distributed systems (Spanner, Calvin) require synthesizing L01 + L02 + networking at a level not covered here.
- No lesson covers measurement methodology: performance benchmarking, statistical significance, confounding factors from NUMA/Turbo Boost/THP that make micro-benchmarks misleading.

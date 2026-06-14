# Graduate CS Self-Study Curriculum: Systems & Cryptography

## 1. Overview

This curriculum covers seven modules of graduate-level computer science, targeting engineers who want to understand how the machine actually works — from distributed consensus protocols down to individual CPU clock cycles, and sideways into the mathematics that secures every TLS connection. The modules are not shallow surveys. Each one contains working Python implementations of the core data structures and algorithms it teaches: you will implement Raft from scratch, build a B+ tree with split logic, write SHA-256 without a crypto library, and construct SSA form including phi-node insertion. A self-studying engineer who works through all seven modules will have internalized the concepts that most engineers only encounter as black boxes.

The modules span three broad domains: **distributed systems** (Lessons 01, 02), **systems programming** (Lessons 03, 04, 05, 06), and **applied cryptography** (Lesson 07). These domains are more entangled than they appear. The memory model you learn in Lesson 03 is the same TSO model explained by the CPU microarchitecture in Lesson 06. The TLB shootdown mechanism in Lesson 04 requires memory barriers explained in Lesson 03. The register file that Lesson 05's register allocator targets is described in Lesson 06's out-of-order execution pipeline. MVCC in Lesson 02 and Raft in Lesson 01 are two halves of the same database durability story. The dependency map below makes every such connection explicit.

Estimated total study time: **200–280 hours** (28–40 hours per module including exercises). The hardest conceptual barriers are SSA construction (Lesson 05), the TAGE branch predictor (Lesson 06), and the correctness proof for ECDSA (Lesson 07).

---

## 2. Concept Dependency Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONCEPT DEPENDENCY MAP                                    │
│                    (arrows = "required to fully understand")                │
└─────────────────────────────────────────────────────────────────────────────┘

[01 Distributed Consensus]          [02 Database Internals]
   Paxos / Raft / PBFT                B+Tree / LSM / MVCC / WAL
   FLP / CAP / Two Generals           ARIES / Buffer Pool / XID
          │                                    │
          └───── replication protocol ────────►│
                                               │
          ◄──── crash recovery complement ─────┘
          │
          │   [07 Cryptography]
          │      ECDH / ECDSA / TLS 1.3
          │      Schnorr / MuSig2 / SHA-256
          └───── TLS 1.3 secures distributed ──►[07]
                                                 │
                                    [07] largely standalone
                                    (number theory prerequisites only)

[03 Memory Models & Lock-Free]      [06 CPU Microarchitecture]
   MESI protocol (software view)       MESI protocol (hardware view)
   TSO / ARM / POWER models            Store buffers / OOO execution
   C++11 memory_order                  Tomasulo / ROB / RS / RAT
   Treiber Stack / MS-Queue            TAGE branch predictor
   Hazard pointers / EBR               SIMD / AVX / cache hierarchy
          │                                    │
          │◄─── why reordering happens ─────────┤
          │     (store buffers + OOO = TSO)     │
          │                                     │
          │──── MFENCE in TLB shootdowns ──►[04]│
          │                                     │
          └──── false sharing (both cover) ─────┘
                                                 │
                                                 │ pipeline hazards → instruction
                                                 │ scheduling; PRF is the register
                                                 │ file that register allocator targets
                                                 ▼
[05 Compiler Internals]             [04 OS Virtual Memory]
   Lexer / Parser / AST / IR           x86-64 page tables (PGD→PTE)
   SSA form + phi nodes                TLB / PCID / ASID
   Dominators / Dominator tree         Demand paging / CoW / fork()
   SCCP / GVN / LICM / DCE            Huge pages (2MB / 1GB / THP)
   Graph coloring register alloc       NUMA / mmap / ASLR / W^X
   LLVM IR / CPython / PyPy JIT        struct mm_struct / vm_area_struct
          │                                    │
          │◄──── PRF size limits allocation ───┘
          │      (352 ROB entries cap MLP)
          │
          └──── JIT code gen pattern ──►[04]
                (W^X: mmap+mprotect)

Cross-cutting connections:
  [03] NUMA topology ─────────────────► [04] NUMA allocation (mbind/madvise)
  [04] huge pages (TLB miss reduction) ─► [06] DRAM bandwidth optimization
  [06] instruction latencies ─────────► [05] instruction scheduling pass
  [05] alias analysis barrier ─────────► [06] why vectorization fails
  [02] B+Tree buffer pool ─────────────► [04] mmap + O_DIRECT patterns
  [01] Raft log ───────────────────────► [02] WAL as single-node Raft analog
  [07] SHA-256 internals ──────────────► [01] cryptographic hash in consensus
  [03] RCU mechanism ──────────────────► [04] Linux kernel read-mostly structures
  [06] SMT/Hyper-Threading ────────────► [03] false sharing across logical cores
```

---

## 3. Recommended Learning Orders

### Path A — Systems-First: `06 → 03 → 04 → 05 → 01 → 02 → 07`

Start with the hardware. Lesson 06 (CPU Microarchitecture) gives you the physical reality: pipeline stages, out-of-order execution, the store buffer, MESI state machines, and the TLB as a hardware structure. With that foundation, Lesson 03 (Memory Models) transforms from "mysterious rules" into "obvious consequences of the store buffer." The TSO model stops being an arbitrary constraint and becomes the inevitable result of allowing stores to drain to L1 asynchronously. Next, Lesson 04 (OS Virtual Memory) builds upward from the TLB hardware you understand into the OS page table walker, CoW semantics, and NUMA policies — the hardware-OS interface becomes navigable. Lesson 05 (Compiler Internals) then shows how the compiler reasons about this hardware: instruction scheduling avoids the pipeline hazards from Lesson 06, register allocation targets the physical register file described there, and JIT code generation uses the W^X mmap pattern from Lesson 04.

With systems mastered, Lesson 01 (Distributed Consensus) adds the network dimension: what happens when you have multiple copies of memory across machines instead of cache lines across sockets? Lesson 02 (Database Internals) extends this with the storage layer: B+ trees, LSM, MVCC, and WAL are the persistence machinery that sits beneath a Raft cluster. Lesson 07 (Cryptography) is the capstone: the mathematics that makes distributed systems trustworthy.

**Best for:** engineers with a hardware/OS/C background. Concept gaps feel small at each step.

### Path B — Application-First: `01 → 02 → 07 → 03 → 06 → 04 → 05`

Start at the application layer. Lesson 01 (Distributed Consensus) tackles the hardest conceptual challenge early — consensus under asynchrony — when your mind is fresh. Lesson 02 (Database Internals) extends this into the storage engine: the Raft log you built in Lesson 01 is a simplified WAL; MVCC shows why crash recovery is non-trivial. Lesson 07 (Cryptography) rounds out the application-layer picture: TLS 1.3 secures your distributed system, ECDSA signs your transactions, SHA-256 builds your Merkle trees.

Then descend. Lesson 03 (Memory Models) motivates why you cannot implement Raft's commit protocol without memory barriers. Lesson 06 (CPU Microarchitecture) explains why those barriers cost what they cost. Lesson 04 (OS Virtual Memory) shows how the OS translates your malloc() calls into the NUMA topology that Lesson 03 warned you about. Lesson 05 (Compiler Internals) closes the loop: the compiler that compiles your lock-free data structures performs the alias analysis that determines which memory ordering transformations it can legally make.

**Best for:** engineers with a distributed-systems or web-services background. Motivation stays high because you see *why* the low-level details matter before drowning in them.

---

## 4. Per-Lesson Summary Cards

### Lesson 01 — Distributed Consensus

| Field | Detail |
|---|---|
| **Core question** | How do N processes agree on a single value when any of them can crash or be slow, and messages can be delayed? |
| **What you'll know after** | Why FLP makes consensus impossible under pure asynchrony; how Paxos sidesteps FLP with randomization/timing; how Raft adds leader election and log replication; what Byzantine faults require (PBFT/HotStuff); the exact failure scenario in Raft Figure 8 |
| **Key Python implementations** | `AcceptorState` + `Proposer` (Paxos); `RaftNode` full cluster with `simulate_partition_and_recovery()`; `PBFTReplica` + `simulate_pbft()` |
| **Connects to** | Lesson 02 (Raft is the replication layer beneath a database); Lesson 07 (TLS secures the consensus wire protocol) |
| **Prerequisite concepts** | Basic networking (TCP, message passing); none of the other 6 lessons are required |
| **Estimated time** | 35–45 hours |
| **Hardest part** | The Raft Figure 8 scenario: understanding *why* a leader cannot commit entries from previous terms by counting replicas, and why the no-op trick fixes it |

---

### Lesson 02 — Database Internals

| Field | Detail |
|---|---|
| **Core question** | How does a database store, index, and recover data such that crashes leave it consistent and concurrent readers never block writers? |
| **What you'll know after** | B+ tree split/merge mechanics; why LSM trees win on write throughput but lose on read amplification; PostgreSQL's xmin/xmax visibility rules; ARIES three-phase recovery; the Sentry XID wraparound incident |
| **Key Python implementations** | `BPlusTree` (insert, split\_leaf, split\_internal, range\_scan); `BloomFilter` (Kirsch-Mitzenmacher double hashing); `SkipList`; `MiniWAL` + `demo_aries()` |
| **Connects to** | Lesson 01 (WAL is single-node crash recovery; Raft adds the distributed dimension); Lesson 04 (buffer pool uses mmap and O\_DIRECT; CoW explains fork + BGSAVE) |
| **Prerequisite concepts** | Basic data structures (trees, linked lists); Lesson 01 helpful but not required |
| **Estimated time** | 30–40 hours |
| **Hardest part** | Write skew in Snapshot Isolation and how SSI detects it via anti-dependency cycles without actually serializing transactions |

---

### Lesson 03 — Memory Models & Lock-Free Programming

| Field | Detail |
|---|---|
| **Core question** | What ordering guarantees does the hardware actually provide, and how do you write correct concurrent code without locks? |
| **What you'll know after** | MESI state machine; why x86 TSO allows only store-load reordering; why ARM/POWER are weaker; all four C++11 memory orders and when each is sufficient; ABA problem and three solutions; RCU semantics |
| **Key Python implementations** | SB litmus test (multiprocessing); `WaitFreeCounter`; `DistributedCounter`; `TreiberStack`; `MSQueue` (Michael-Scott); `SeqLock`; `RCUConfig` |
| **Connects to** | Lesson 06 (hardware explanation of why TSO exists: store buffers drain asynchronously); Lesson 04 (MFENCE required in TLB shootdown IPIs; NUMA topology shared) |
| **Prerequisite concepts** | Python threading basics; no hardware background required (but Lesson 06 first makes it click faster) |
| **Estimated time** | 35–45 hours |
| **Hardest part** | The IRIW litmus test: four threads, two stores, two loads — why ARM allows an outcome that x86 forbids, and what this implies about coherence vs consistency |

---

### Lesson 04 — OS Virtual Memory

| Field | Detail |
|---|---|
| **Core question** | How does the OS give every process the illusion of a private 48-bit address space, and what is the performance cost? |
| **What you'll know after** | x86-64 four-level page table walk (CR3→PGD→PUD→PMD→PTE→PA); every PTE bit (NX, PFN, dirty, accessed, U/S, R/W, Present); TLB shootdown via IPI; CoW semantics for fork(); huge page promotion via THP; Linux struct mm\_struct and vm\_area\_struct (maple tree in 6.1+) |
| **Key Python implementations** | `print_address_space_layout()`; `demonstrate_cow_physical_sharing()`; `benchmark_mmap_munmap()`; `allocate_huge_page_mmap()`; `execute_jit_shellcode()` (W^X pattern); `read_buddy_allocator_stats()` |
| **Connects to** | Lesson 03 (TLB shootdowns require memory barriers; NUMA allocation policies); Lesson 06 (TLB covered from hardware side; huge pages optimize DRAM bandwidth); Lesson 05 (JIT code generation uses mmap+mprotect W^X pattern) |
| **Prerequisite concepts** | C basics (pointers, virtual vs physical address); basic OS concepts (processes, system calls) |
| **Estimated time** | 30–40 hours |
| **Hardest part** | TLB shootdown scalability: why invalidating a single page on a 512-core machine can stall every core for microseconds, and why PCID/ASID only partially helps |

---

### Lesson 05 — Compiler Internals

| Field | Detail |
|---|---|
| **Core question** | How does a compiler transform source text into optimized machine code, and where does it get stuck? |
| **What you'll know after** | The full pipeline (Lexer→Parser→AST→IR→SSA→Optimization→Instruction Selection→Register Allocation→Emission); how to construct SSA form with dominance frontiers and phi nodes; SCCP lattice (⊤/const/⊥); why alias analysis is the fundamental optimization barrier; graph coloring register allocation with spill |
| **Key Python implementations** | `SSABuilder` (compute\_dominators, compute\_dominance\_frontiers, insert\_phi\_functions, rename\_variables); `sccp()`; `licm()`; `build_interference_graph()`; `color_graph()` (Chaitin's algorithm) |
| **Connects to** | Lesson 06 (instruction scheduling targets pipeline latencies; PRF size constrains register allocation); Lesson 04 (JIT code generation W^X pattern) |
| **Prerequisite concepts** | Basic graph algorithms (DFS, topological sort); understanding of assembly helpful; Lesson 06 recommended first for Path A |
| **Estimated time** | 40–55 hours (the largest module) |
| **Hardest part** | SSA construction: computing iterated dominance frontiers and then correctly renaming variables via DFS over the dominator tree, especially understanding why the phi-node insertion must precede renaming |

---

### Lesson 06 — CPU Microarchitecture

| Field | Detail |
|---|---|
| **Core question** | What happens inside a modern superscalar out-of-order CPU between fetching an instruction and retiring it? |
| **What you'll know after** | All stages of the Intel Ice Lake (Sunny Cove) pipeline; Tomasulo's algorithm with renaming (RAT, PRF, RS, ROB, CDB); how TAGE branch prediction works with geometric history lengths; why Spectre v1 is a fundamental consequence of speculative execution; why the 352-entry ROB limits memory-level parallelism to ~352 outstanding cache misses |
| **Key Python implementations** | `benchmark_access_patterns()`; `benchmark_false_sharing()`; `benchmark_branch_prediction()` (sorted vs unsorted); `benchmark_simd()` (NumPy vs pure Python); `benchmark_matmul_variants()` (tiling + packing) |
| **Connects to** | Lesson 03 (hardware reason for TSO: store buffers; MESI from hardware side; false sharing); Lesson 04 (TLB from hardware side; huge pages optimize DRAM bandwidth); Lesson 05 (pipeline hazards → instruction scheduling; PRF constrains register allocation) |
| **Prerequisite concepts** | Basic assembly reading; basic cache/memory hierarchy concepts |
| **Estimated time** | 35–45 hours |
| **Hardest part** | The TAGE predictor: geometric history lengths, tagged entries, the "useful" bit, provider vs alternate prediction — it is a multi-level tournament predictor with 5–6 interacting mechanisms |

---

### Lesson 07 — Cryptography Internals

| Field | Detail |
|---|---|
| **Core question** | How do elliptic curves provide security equivalent to 3072-bit RSA at 256 bits, and how is this assembled into real protocols? |
| **What you'll know after** | Group law on elliptic curves (point addition, doubling, scalar multiplication); why ECDLP is hard (no index calculus); Montgomery ladder timing-safe scalar mul; ECDH/HKDF; ECDSA correctness and the Sony PS3 catastrophe (fixed k → private key recovery); Ed25519 complete addition formulas; Schnorr ZKP and Fiat-Shamir; MuSig2 two-round nonce protocol; TLS 1.3 full handshake; SHA-256 from scratch |
| **Key Python implementations** | `EllipticCurve` (add, double, scalar\_mul, modinv); `ECDSA` (sign with RFC 6979 deterministic nonce, verify); `SchnorrProof` (prove, verify, simulate); `musig2_key_aggregate()`; `tls13_key_schedule()`; `sha256_scratch()` |
| **Connects to** | Lesson 01 (TLS 1.3 secures distributed protocol wire; SHA-256 in consensus Merkle trees); Lesson 02 (SHA-256 used in content-addressable storage) |
| **Prerequisite concepts** | Modular arithmetic (Fermat's Little Theorem); basic group theory helpful but taught inline |
| **Estimated time** | 30–40 hours |
| **Hardest part** | The ECDSA correctness proof (why verification recovers the signer's public key) and understanding the Sony PS3 attack: a single reused nonce k across two signatures leaks the private key via simple algebra |

---

## 5. Concept Cross-Reference Table

| Concept | Primary Lesson | Secondary Lesson(s) | Notes |
|---|---|---|---|
| MESI cache coherence protocol | 03 (software view) | 06 (hardware state machine) | Lesson 06 shows actual M/E/S/I transitions in silicon |
| False sharing | 03 (padding fix) | 06 (MESI ping-pong benchmark) | Lesson 06 shows it as 5–8x slowdown in benchmarks |
| x86 TSO memory model | 03 (rules) | 06 (cause: store buffer + OOO) | Lesson 06 explains *why* the rule exists |
| NUMA topology | 03 (distributed counters) | 04 (mbind/madvise policies) | Lesson 04 covers kernel allocation; Lesson 03 covers access patterns |
| TLB (Translation Lookaside Buffer) | 04 (OS management) | 06 (hardware structure, PCID) | Lesson 06 covers iTLB/dTLB/STLB latencies |
| Huge pages (2MB / 1GB) | 04 (THP, MAP_HUGETLB) | 06 (DRAM bandwidth effect) | Same mechanism, different optimization goal |
| Memory barriers / fences | 03 (MFENCE, acquire/release) | 04 (TLB shootdown IPIs) | 04 shows OS kernel requiring barriers |
| W^X / JIT code generation | 04 (mprotect pattern) | 05 (JIT compilation) | Same mmap+mprotect pattern in both |
| Register file (physical) | 06 (RAT, PRF ~280 regs) | 05 (register allocator target) | Lesson 05 allocator must not exceed PRF size |
| Instruction scheduling | 05 (post-allocation) | 06 (hazard avoidance) | Lesson 06 defines the latencies Lesson 05 schedules around |
| Alias analysis | 05 (optimization barrier) | 06 (vectorization failures) | Aliasing prevents SIMD and reordering in both |
| Paxos | 01 (single-decree, Multi-Paxos) | — | Foundation for Raft conceptually |
| Raft | 01 (full protocol) | 02 (replication layer) | Lesson 02's WAL is what Raft replicates |
| WAL / ARIES recovery | 02 (full ARIES) | 01 (Raft as distributed WAL) | ARIES = single-node crash recovery |
| MVCC | 02 (PostgreSQL xmin/xmax) | — | Snapshot Isolation + SSI covered in depth |
| B+ tree | 02 (insert/split/delete) | — | Foundation of all page-oriented indexes |
| LSM tree | 02 (MemTable + SSTable) | — | Foundation of RocksDB / Cassandra / BigTable |
| SSA form | 05 (construction algorithm) | — | Used in LLVM IR; enables most optimizations |
| Graph coloring register allocation | 05 (Chaitin's algorithm) | — | NP-complete; spill heuristics matter |
| SCCP (Sparse Conditional Constant Propagation) | 05 (lattice values) | — | Best constant propagation algorithm |
| Elliptic curve group law | 07 (point add/double) | — | Foundation for all ECC protocols |
| ECDSA | 07 (sign/verify/attack) | — | Sony PS3 catastrophe from k reuse |
| TLS 1.3 handshake | 07 (full key schedule) | 01 (secures distributed comms) | HKDF key derivation tree |
| SHA-256 | 07 (from scratch) | 01 (Merkle trees) | Merkle-Damgård + Davies-Meyer |
| Spectre v1 | 06 (attack mechanism) | — | Consequence of speculative execution + cache timing |
| CoW (Copy-on-Write) | 04 (fork + page faults) | — | Redis BGSAVE problem from CoW amplification |
| Lock-free data structures | 03 (Treiber, MS-Queue) | — | ABA problem + hazard pointers + EBR |
| RCU (Read-Copy-Update) | 03 (RCUConfig) | 04 (Linux kernel read-mostly) | Lesson 04 uses RCU for vm_area_struct |
| PBFT / Byzantine fault tolerance | 01 (PBFT replica) | — | O(n²) message complexity; HotStuff improves |
| Dominators / dominator tree | 05 (Cooper-Harvey-Kennedy) | — | Foundation of SSA construction |
| Branch prediction (TAGE) | 06 (geometric history) | — | Hardest single concept in the curriculum |
| Out-of-order execution (Tomasulo) | 06 (ROB, RS, RAT, CDB) | 03 (why TSO exists) | ROB retirement maintains program order |

---

## 6. Study Recommendations

### High-Synergy Back-to-Back Pairs

**Lesson 06 + Lesson 03 (either order, read together):** These two lessons are the hardware/software split of the same phenomenon. Read the MESI section in Lesson 03, then read the MESI section in Lesson 06. Read the TSO rules in Lesson 03, then read the store-buffer explanation in Lesson 06. The false sharing section in Lesson 06 benchmarks the MESI ping-pong that Lesson 03 explains theoretically. Doing these back-to-back (or interleaved chapter by chapter) is more effective than doing them weeks apart.

**Lesson 04 + Lesson 06 (TLB sections):** Lesson 04 covers TLB from the OS side (shootdowns, PCID, huge page promotion), Lesson 06 covers TLB from the hardware side (L1 iTLB/dTLB, L2 STLB, PCID in silicon). The `benchmark_tlb_effect()` in Lesson 06 directly measures the cost of the page table walks that Lesson 04 describes. Do these TLB sections in the same study session.

**Lesson 01 + Lesson 02 (WAL chapter):** After implementing Raft's log replication in Lesson 01, immediately read the WAL/ARIES section in Lesson 02. The Raft log is a distributed WAL; ARIES is the single-node crash recovery protocol that answers "what happens when Raft elects a new leader and the old one had partially written entries?" These two form a complete durability story.

**Lesson 05 + Lesson 07 (mathematical rigor):** Both lessons require careful reading of proofs — SSA correctness in Lesson 05 and ECDSA correctness in Lesson 07. If you've done Lesson 05's proof work (dominance frontier correctness, SCCP lattice monotonicity), you have the mindset for Lesson 07's algebraic proofs. Alternatively, doing Lesson 07's modular arithmetic proofs first warms up the proof-reading muscle for Lesson 05.

### Common Misconceptions to Pre-empt

1. **"Sequential consistency is the default."** It is not. Every x86 program runs under TSO by default. `std::atomic` with `memory_order_relaxed` gives you no ordering at all. You must explicitly request `seq_cst` to get SC semantics, at significant cost.

2. **"Lock-free means fast."** Lock-free means *a thread that is preempted cannot block other threads*. A lock-free queue can easily be slower than a mutex-protected queue under low contention. Lesson 03's benchmarks show this.

3. **"SSA means single assignment in the source."** SSA is a property of the IR, not the source language. Variables are renamed to versions (x → x_1, x_2, x_3); phi nodes merge versions at join points. Lesson 05's construction algorithm generates SSA from ordinary three-address code.

4. **"Huge pages are always faster."** Huge pages reduce TLB pressure but can increase page fault latency (one huge-page fault reads 2MB), increase memory waste (internal fragmentation), and break NUMA interleaving. Lesson 04 covers when not to use THP.

5. **"Raft is simpler than Paxos."** Raft is easier to *teach*; it is not simpler to implement correctly. The Figure 8 scenario, the no-op leader commitment, and the snapshot/log compaction interactions make production Raft implementations subtle. Lesson 01 implements all of them.

6. **"Elliptic curve crypto is quantum-safe."** It is not. Shor's algorithm runs ECDLP in polynomial time on a quantum computer. Post-quantum alternatives (lattice-based) are not covered in this curriculum. Lesson 07 makes this explicit.

### Exercise Approach

**Implement before reading the explanation.** Each lesson introduces a concept, then shows an implementation. Stop reading when the concept is stated, try to implement it yourself, then compare. The gap between your implementation and the lesson's implementation reveals exactly what you misunderstood.

**Run the benchmarks and change parameters.** Every performance claim in Lessons 03, 04, 06 is backed by a Python benchmark. Run it. Then change the array size, thread count, or alignment, and see if the results match the theory. This is more valuable than reading the numbers.

**Break the implementations.** Remove the `unsafe.compareAndSwap` from the Treiber Stack. Remove the `rwlock` from the SeqLock. Remove the no-op entry from the Raft leader implementation. Observe the failure mode. Understanding *why* each piece of the protocol exists requires seeing what breaks without it.

---

## 7. The Missing Topics

This curriculum is deep in the topics it covers and narrow in what it covers. An engineer who completes all seven modules should not confuse depth with breadth. The following are significant topics that are not covered:

**Networking internals.** TCP/IP stack implementation, kernel bypass (DPDK, io_uring, XDP/eBPF), RDMA, congestion control algorithms (CUBIC, BBR), and network function virtualization are absent. Lesson 01 uses message passing as an abstraction without explaining what the message transport does.

**Storage hardware.** NVMe queue depth, SSD write amplification (distinct from LSM write amplification), ZNS (Zoned Namespace) SSDs, and persistent memory (Intel Optane) semantics are not covered. Lesson 02 covers software storage structures but not the hardware they run on.

**GPU architecture.** SIMD is touched in Lesson 06 (AVX on CPU), but GPU compute (CUDA/ROCm, warp scheduling, memory coalescing, shared memory bank conflicts, tensor cores) is entirely absent. This is a significant gap for any engineer working on ML infrastructure.

**Post-quantum cryptography.** Lesson 07 acknowledges that ECC is quantum-vulnerable but does not cover lattice-based cryptography (Kyber, Dilithium), hash-based signatures (XMSS), or the NIST PQC standardization. As of 2024–2026, this is increasingly production-relevant.

**Garbage collection algorithms.** The curriculum covers memory management at the OS level (buddy allocator, SLUB) and mentions CPython reference counting and PyPy's JIT, but does not cover generational GC, tri-color marking, concurrent GC (G1, ZGC, Shenandoah), or the tradeoffs between GC pauses and throughput.

**Formal verification.** The lessons describe correctness arguments informally. TLA+/PlusCal for distributed systems, separation logic for concurrent code, or LLVM's alive2 for compiler correctness are not covered.

**Operating system scheduling.** The curriculum covers the OS's memory management extensively but does not cover the CPU scheduler (CFS, EDF, real-time scheduling, cgroup CPU quotas, priority inversion, or the Linux BPF scheduler).

**Distributed storage systems.** Lessons 01 and 02 together cover consensus and single-node storage. Distributed storage systems (Spanner, Dynamo, Cassandra, Kafka, HDFS) — which compose these primitives at scale — are not covered.

**Security and exploitation.** Spectre v1 is mentioned as a consequence of branch prediction, but exploit techniques, mitigation effectiveness (Retpoline, KPTI overhead), side-channel analysis more broadly, and the intersection of compiler internals with security (stack protector, CFI, ASAN) are not covered systematically.

---

*This document maps version: all 7 lessons as of June 2026. Lesson content evolves; re-verify cross-references if lessons are updated.*

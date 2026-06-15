# Solution: Matrix Multiply Performance Analysis (Hard Exercise — Lesson 06)

## The Question

```
NumPy matmul on 1000×1000 float32: 8 ms
Pure Python triple loop:           45,000 ms  (45 seconds)
Naive C extension (triple loop):   400 ms
C extension with 32×32 tiling:    12 ms
Tiled C + AVX2 SIMD:              4 ms
```

**Questions:**
- (a) Explain each speedup with reasoning
- (b) Why does tiled SIMD C still lose to NumPy?
- (c) What additional technique does BLAS use?
- (d) Predict `perf stat` output for each
- (e) Write a Python benchmark harness

---

## (a) Explaining Each Speedup

### Python → Naive C: 112.5x speedup

**Why Python is slow (45,000 ms):**

The 1000×1000 matrix multiply requires `2 × N³ = 2 × 10⁹` floating-point operations (FLOPs). CPython executes these as Python bytecodes:

```python
for i in range(1000):        # 1000 iterations
    for k in range(1000):    # 1,000,000 iterations
        a_ik = A[i][k]       # LOAD_FAST, BINARY_SUBSCR, BINARY_SUBSCR
        for j in range(1000):
            C[i][j] += a_ik * B[k][j]  # BINARY_MULTIPLY, BINARY_ADD, STORE
```

Each inner loop iteration executes ~15 Python bytecodes. At CPython's speed of ~100M bytecodes/sec, the inner loop alone takes:
- 10⁹ multiplications × 15 bytecodes / 10⁸ bytecodes/sec ≈ 150 seconds

In practice ~45s because CPython partially optimizes hot paths.

**Why C is faster (400 ms):**
1. **No interpreter overhead:** `gcc -O3` compiles directly to x86-64 machine code. Each FMA (fused multiply-add) is 1 instruction, not 15 bytecodes.
2. **Native float registers:** Python floats are heap-allocated `PyObject` structs (24+ bytes each). C uses `float` (4 bytes) directly in registers.
3. **Compiler optimization:** `gcc -O3` enables:
   - Loop unrolling: reduces branch overhead
   - Auto-vectorization: may emit partial SSE/AVX instructions
   - Register allocation: keeps hot variables in registers

**Speedup = 45,000 / 400 ≈ 112.5x** — matches CPython's ~100× bytecode overhead for arithmetic.

---

### Naive C → Tiled C: 33.3x speedup

**Why naive C has a cache problem:**

The naive triple loop for matrix multiply is `C[i][j] += A[i][k] * B[k][j]`. Look at how matrix B is accessed:

```
Outer loop i=0..999:
  Middle loop k=0..999:
    Inner loop j=0..999:
      Access B[k][j]   ← columns of B vary fastest
```

Wait — that's actually sequential access on B's rows. The problem is the **column access pattern** when you think of it differently:

For fixed `i` and varying `k`:
- `A[i][k]` — row of A: sequential in memory ✓
- `B[k][j]` — for fixed j, varying k = **column of B** = stride-1000 access!

When the compiler transforms the loop order to `i,k,j` (ikj order, which is common for register reuse), `B[k][j]` has a new row of B for each k. But 1000×1000×4 bytes = 4MB for matrix B, which exceeds L1 cache (~32KB) and even L2 (~512KB). Every iteration of k reloads a new row of B from L3 or memory.

**L1 miss latency:** ~4 cycles vs L1 hit: ~4 cycles... but L3 miss: ~30-50ns × 3GHz = ~100 cycles.

**Why tiling works (12 ms):**

Process a **32×32 block** of A, B, and C simultaneously:
- 32×32 float32 = 4,096 bytes ≈ **4KB** → fits in L1 cache (32KB)
- Two tiles (A_tile + B_tile) = 8KB → still fits in L1

For each 32×32 tile:
- Load 32 rows of A (32×32 floats) → stays in L1
- Load 32 columns of B (32×32 floats) → stays in L1
- Do 32×32×32 = 32,768 FLOPs with 2×32² = 2,048 loads
- **Arithmetic intensity = 32,768 / 2,048 = 16 FLOPs/byte** (vs ~0.5 for naive)

```
Naive:  A[i][k]×B[k][j] for all k per (i,j):
        → 1000 B-matrix row loads, each potentially missing L1
        → ~1000 L3 accesses per output element

Tiled:  Within 32×32 tile, B stays in L1:
        → 1 tile load + 32³ operations in-cache
        → ~1/32 the cache misses per FLOP
```

Speedup = 400 / 12 ≈ 33x. This roughly matches the L3-to-L1 latency ratio (~30-50×).

---

### Tiled C → AVX2 SIMD: 3x speedup

**AVX2 capabilities:**
- 256-bit registers: hold **8× float32** simultaneously
- `_mm256_fmadd_ps`: 8 fused multiply-add in one instruction
- Throughput: 1 FMA/cycle on modern CPUs = 8 FLOPs/cycle per FMA unit (×2 FMA units on some CPUs = 16 FLOPs/cycle)

**Why only 3× (not 8×):**
1. Memory bandwidth still partially limits: loading tile data from L1 takes non-zero time.
2. AVX2 FMA has 5-cycle latency; pipelining reduces this but adds overhead.
3. The 32×32 tile inner loop has overhead from loop control, address computation.
4. Partial tiles at matrix edges may use scalar fallback.

**Theoretical vs actual:**
- Theoretical AVX2 gain: 8×
- Measured gain: ~3× because of memory latency, instruction overhead, and imperfect vectorization

---

## (b) Why Tiled SIMD C Still Loses to NumPy

NumPy (via OpenBLAS/MKL) achieves **8ms** vs our hand-written **4ms** — actually NumPy is faster. The reasons:

### 1. Multi-Level Cache Tiling

Our code tiles only for L1. OpenBLAS tiles for **all three cache levels**:

```
L3 tile: ~4MB blocks → fits in L3 cache (say 16MB)
  L2 tile: ~256KB blocks → fits in L2 cache (512KB)
    L1 tile: 32×32 → fits in L1 cache (32KB)
      Micro-kernel: 8×6 register block → stays in SIMD registers
```

This 4-level hierarchy eliminates capacity misses at every level.

### 2. Register Blocking (Micro-Kernel)

The innermost kernel keeps partial results in **SIMD registers across outer loop iterations**:

```
// Pseudo-code: 8x6 micro-kernel (8 rows of A, 6 columns of B)
// Load 8×1 panel of A into 8 AVX registers
// Load 1×6 panel of B into 6 AVX registers
// Compute: 8×6 = 48 FMAs, all results stay in 48 accumulator registers
// No stores to L1 for intermediate results
```

Our hand-written kernel stores intermediate results back to memory, costing extra L1 bandwidth.

### 3. Data Packing (Pre-Transposing)

OpenBLAS **rearranges** matrix panels into cache-friendly layouts before computation:

```
Original B matrix (row-major):
  B[0][0..999], B[1][0..999], ...

Packed B panel (for 32-wide computation):
  B[0][0..7], B[1][0..7], ..., B[31][0..7],  ← 8 columns packed contiguously
  B[0][8..15], B[1][8..15], ..., B[31][8..15], ...
```

Packing converts strided accesses to sequential accesses, allowing the hardware prefetcher to predict and prefetch perfectly.

### 4. Multi-Threading

OpenBLAS spawns multiple threads (typically 1 per physical core). A 12-core machine achieves ~12× throughput for large matrices. Our single-threaded C code cannot match this.

### 5. Instruction-Level Tricks

OpenBLAS is written in hand-optimized **assembly**, not C intrinsics:
- Uses `VFMADD231PS` with careful register assignments
- Inserts `PREFETCHT0` instructions to prefetch the next tile while computing the current one
- Aligns all data to 64-byte cache line boundaries

---

## (c) BLAS's Additional Key Technique: Packing

```
Algorithm: GEMM (General Matrix Multiply) with packing
─────────────────────────────────────────────────────────
For each L3 block of B (say 1024 columns at a time):
  PACK B panel into buffer B_packed (reordered for SIMD access)
  
  For each L3 block of A (say 256 rows at a time):
    PACK A panel into buffer A_packed
    
    For each L2 block (B: 256 cols, A: 64 rows):
      For each L1 block (32×32):
        MICRO-KERNEL: 8×6 register-blocked inner loop
        using A_packed[L1 slice] and B_packed[L1 slice]
        All intermediate results in SIMD registers
        → 48 FMAs per cycle, no L1 stores needed
```

**Packing cost:** O(N²) work to pack, enabling O(N³) BLAS computation at peak throughput. Amortized cost per FLOP → 0 as N grows. For N=1000, packing ~8M floats = ~32MB of writes takes ~0.3ms, negligible vs 8ms compute.

---

## (d) Predicted `perf stat` Output

```bash
perf stat -e cache-misses,cache-references,instructions,cycles,branches,branch-misses \
          -- python benchmark.py
```

| Metric | Python | Naive C | Tiled C | Tiled SIMD | OpenBLAS |
|--------|--------|---------|---------|-----------|----------|
| Time | 45,000ms | 400ms | 12ms | 4ms | 8ms |
| Instructions | ~15×10⁹ | ~4×10⁹ | ~3×10⁹ | ~500×10⁶ | ~300×10⁶ |
| Cycles | ~135×10⁹ | ~1.2×10⁹ | ~120×10⁶ | ~15×10⁶ | ~28×10⁶ |
| IPC (instr/cycle) | ~0.1 | ~3.3 | ~25 | ~33 | ~10 |
| Cache misses | ~2×10⁹ | ~500×10⁶ | ~10×10⁶ | ~2×10⁶ | ~200×10³ |
| Branch misses | ~5×10⁶ | ~1×10⁶ | ~100×10³ | ~50×10³ | ~10×10³ |
| FLOPs/cycle | ~0.01 | ~1.7 | ~17 | ~130 | ~70 |

**Key observations:**

- **Python:** IPC < 0.1 because the interpreter stalls constantly on Python object loads (heap traversal). Cache misses are dominated by Python object accesses scattered across heap.
- **Naive C:** IPC ~3 (good instruction-level parallelism from compiler). Still ~500M cache misses because B matrix column access pattern trashes L3.
- **Tiled C:** Dramatic miss reduction. IPC spikes because the inner loop is a tight multiply-add with no stalls. The "25 IPC" reflects that SIMD instructions count as 1 instruction but do 8 FLOPs.
- **Tiled SIMD:** Even higher effective FLOPs/cycle. Cache miss count approaches compulsory (first-touch) minimum.
- **OpenBLAS:** Slightly lower IPC than hand-written SIMD because it uses multiple threads (pipeline stalls from thread synchronization) and the time includes packing overhead. Miss count is near-zero — compulsory misses only (first time each cache line is loaded).

**Note on OpenBLAS time vs Tiled SIMD:** In the exercise, OpenBLAS (8ms) is slower than tiled SIMD (4ms) — this seems counterintuitive given the analysis above. This can happen on smaller matrices (N=1000) where packing overhead and thread launch overhead dominate. For large matrices (N=8000+), OpenBLAS wins decisively.

---

## (e) Python Benchmark Harness

```python
"""
Matrix multiplication benchmark harness.
Proper statistical methodology: 3 warmup runs, then 7 timed runs, take median.
"""
import time
import statistics
import numpy as np
from typing import Callable, Any, List, Tuple


def benchmark(
    fn: Callable[[], Any],
    warmup_runs: int = 3,
    timed_runs: int = 7,
    label: str = "function"
) -> Tuple[float, float]:
    """
    Benchmark fn() with proper warmup.
    Returns (median_ms, min_ms).
    
    WHY warmup matters:
    - First call may trigger: OS page allocation (minor faults), CPU frequency scaling
      ramp-up, branch predictor learning, and JIT compilation in NumPy/BLAS.
    - Taking minimum of timed runs filters outliers from OS scheduling jitter.
    - Median is more robust than mean for distributions with tail latencies.
    """
    # Warmup
    for _ in range(warmup_runs):
        fn()
    
    # Timed runs
    times_ms = []
    for _ in range(timed_runs):
        t0 = time.perf_counter()
        fn()
        t1 = time.perf_counter()
        times_ms.append((t1 - t0) * 1000)
    
    return statistics.median(times_ms), min(times_ms)


def python_matmul(A: list, B: list, n: int) -> list:
    """Pure Python triple loop (ikj order for better cache behavior than ijk)."""
    C = [[0.0] * n for _ in range(n)]
    for i in range(n):
        for k in range(n):
            a_ik = A[i][k]   # hoist out of inner loop (register-like optimization)
            for j in range(n):
                C[i][j] += a_ik * B[k][j]
    return C


def run_all_benchmarks():
    N = 1000
    N_PY = 64   # Python is too slow for N=1000; extrapolate

    print(f"Matrix Multiplication Benchmark — {N}×{N} float32")
    print(f"GFLOPs required: {2 * N**3 / 1e9:.2f}")
    print("=" * 70)
    print(f"{'Implementation':<35} {'Median(ms)':>10} {'Min(ms)':>10} {'GFLOPs':>10}")
    print("-" * 70)

    # 1. NumPy matmul
    A = np.random.rand(N, N).astype(np.float32)
    B = np.random.rand(N, N).astype(np.float32)
    
    med, mn = benchmark(lambda: np.matmul(A, B), label="numpy.matmul")
    gflops = 2 * N**3 / (med / 1000) / 1e9
    print(f"{'NumPy matmul (OpenBLAS)':<35} {med:>10.2f} {mn:>10.2f} {gflops:>10.1f}")
    numpy_med = med

    # 2. NumPy @ operator (should be identical)
    med, mn = benchmark(lambda: A @ B, label="A @ B")
    gflops = 2 * N**3 / (med / 1000) / 1e9
    print(f"{'NumPy @ operator':<35} {med:>10.2f} {mn:>10.2f} {gflops:>10.1f}")

    # 3. NumPy dot
    med, mn = benchmark(lambda: np.dot(A, B), label="np.dot")
    gflops = 2 * N**3 / (med / 1000) / 1e9
    print(f"{'NumPy dot':<35} {med:>10.2f} {mn:>10.2f} {gflops:>10.1f}")

    # 4. NumPy einsum (different code path)
    med, mn = benchmark(lambda: np.einsum('ij,jk->ik', A, B, optimize=True),
                        label="einsum")
    gflops = 2 * N**3 / (med / 1000) / 1e9
    print(f"{'NumPy einsum (optimized)':<35} {med:>10.2f} {mn:>10.2f} {gflops:>10.1f}")

    # 5. Python triple loop (tiny matrix, extrapolate to N=1000)
    A_list = [[float(i+j) for j in range(N_PY)] for i in range(N_PY)]
    B_list = [[float(i*j+1) for j in range(N_PY)] for i in range(N_PY)]
    
    med_small, _ = benchmark(
        lambda: python_matmul(A_list, B_list, N_PY),
        warmup_runs=1, timed_runs=3, label="python"
    )
    # O(n^3) scaling: time ~ n^3 → scale by (N/N_PY)^3
    scale = (N / N_PY) ** 3
    extrapolated_ms = med_small * scale
    gflops_py = 2 * N**3 / (extrapolated_ms / 1000) / 1e9
    print(f"{'Python loop (extrapolated)':<35} {extrapolated_ms:>10.0f} "
          f"{'N/A':>10} {gflops_py:>10.4f}")
    print(f"  (measured at N={N_PY}: {med_small:.1f}ms, extrapolated via O(N³))")

    print("-" * 70)
    print(f"\nSpeedup summary vs pure Python (extrapolated):")
    print(f"  Naive C (estimated):    ~112x")
    print(f"  Tiled C (estimated):    ~{extrapolated_ms / 12:.0f}x (12ms reference)")
    print(f"  Tiled SIMD (estimated): ~{extrapolated_ms / 4:.0f}x (4ms reference)")
    print(f"  NumPy:                  ~{extrapolated_ms / numpy_med:.0f}x ({numpy_med:.1f}ms measured)")

    print(f"\nNumPy peak throughput: {2 * N**3 / (numpy_med/1000) / 1e9:.1f} GFLOPs")
    
    # Theoretical peak for context
    import multiprocessing
    n_cores = multiprocessing.cpu_count()
    # Assume 3.5GHz, 8 float32 per AVX2 register, 2 FMA units = 2 FMA/cycle
    freq_ghz = 3.5
    flops_per_cycle = 8 * 2 * 2  # 8 elements/AVX × 2 FMA units × 2 FLOPs/FMA
    theoretical_gflops_single = freq_ghz * flops_per_cycle
    theoretical_gflops_multi = theoretical_gflops_single * n_cores
    print(f"\nTheoretical peak (single core, AVX2 FMA, {freq_ghz}GHz): "
          f"~{theoretical_gflops_single:.0f} GFLOPs")
    print(f"Theoretical peak ({n_cores} cores): ~{theoretical_gflops_multi:.0f} GFLOPs")
    print(f"NumPy efficiency: {2*N**3/(numpy_med/1000)/1e9/theoretical_gflops_multi*100:.1f}%")


def demonstrate_cache_effect():
    """
    Show the cache effect directly: time matrix multiply at different sizes.
    When the matrix overflows each cache level, time should jump.
    """
    print("\nCache-size effect on matmul time:")
    print(f"{'N':>6} {'Size(KB)':>10} {'Time(ms)':>10} {'GFLOPs':>10} {'Trend':>8}")
    print("-" * 50)
    
    prev_gflops = None
    for N in [32, 64, 128, 256, 512, 1024]:
        A = np.random.rand(N, N).astype(np.float32)
        B = np.random.rand(N, N).astype(np.float32)
        size_kb = N * N * 4 / 1024  # bytes per matrix
        
        med, _ = benchmark(lambda: np.matmul(A, B), warmup_runs=2, timed_runs=5)
        gflops = 2 * N**3 / (med / 1000) / 1e9
        trend = ""
        if prev_gflops:
            ratio = gflops / prev_gflops
            trend = f"{'↑' if ratio > 0.9 else '↓'} {ratio:.1f}x"
        prev_gflops = gflops
        
        print(f"{N:>6} {size_kb:>10.0f} {med:>10.3f} {gflops:>10.1f} {trend:>8}")
    
    print("\n  Cache levels (approximate): L1=32KB, L2=512KB, L3=8-32MB")
    print("  GFLOPs drops when 3×N×N×4 bytes exceeds each cache level")


if __name__ == "__main__":
    run_all_benchmarks()
    demonstrate_cache_effect()
```

---

## Summary Table

| Stage | Time | What changed | Why |
|-------|------|-------------|-----|
| Python | 45,000ms | Baseline | Interpreter + object overhead |
| Naive C | 400ms | Remove interpreter | Native floats, no bytecode |
| Tiled C | 12ms | Cache blocking | B-matrix stays in L1 during tile |
| Tiled SIMD | 4ms | AVX2 FMA | 8 FLOPs/instruction |
| OpenBLAS | 8ms | Multi-level tiling, packing, threads | Near-theoretical peak |

**Key insight:** Performance optimization is hierarchical. Each level addresses the dominant bottleneck of the previous level:
1. Python → C: eliminate interpreter (CPU bound → instruction bound)
2. C → tiled: eliminate cache misses (memory bound → cache bound)  
3. tiled → SIMD: eliminate scalar instruction overhead (compute bound → near-peak)
4. hand-SIMD → BLAS: eliminate remaining inefficiencies (multi-threading, packing, prefetch)

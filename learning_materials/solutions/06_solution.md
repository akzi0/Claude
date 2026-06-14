# Solution: Lesson 06 — CPU Microarchitecture
## Hard Exercise: Matrix Multiplication Analysis

---

## 1. Full Exercise Restatement

**Benchmark results for 512×512 float32 matrix multiplication (A × B = C):**

| Implementation | Time |
|---|---|
| Python (pure loops) | 45,000 ms (extrapolated) |
| Naive C (triple loop) | 400 ms |
| Cache-tiled C | 12 ms |
| Tiled + AVX2 SIMD | 4 ms |
| NumPy (OpenBLAS) | 8 ms |

**Questions:**

**(a)** Explain the speedup at each transition: Python→C, Naive C→Tiled, Tiled→AVX2.

**(b)** Why is NumPy (8ms) slower than the hand-optimized tiled AVX2 (4ms)?

**(c)** What is BLAS "packing" and why does it matter?

**(d)** How would a `perf stat` table look for each implementation?

**(e)** Write a complete benchmarking harness.

---

## 2. Conceptual Solution Walkthrough

### Part (a): Speedup Analysis at Each Transition

**Python → Naive C: 112.5× speedup (45,000ms → 400ms)**

Python's overhead for a triple loop over 512³ = 134M iterations:
1. **Interpreter overhead:** Each `a[i][j]` access requires `LOAD_FAST`, `BINARY_SUBSCR`, `BINARY_OP` bytecode instructions — ~5–10 interpreter dispatches per inner-loop iteration
2. **Object boxing/unboxing:** Python floats are heap objects. `a[i][j] * b[j][k]` creates a new float object for the product, increments refcounts, and the result is stored as another heap object — ~3 heap allocations per multiply
3. **No SIMD:** Python cannot auto-vectorize; every float is processed individually
4. **GIL:** Single-threaded due to GIL

In C, the compiler (with `-O2`):
- Emits scalar `fmul` instructions (~1 cycle each)
- Loop overhead is 1–2 instructions per iteration
- No allocation, no GC, no interpreter dispatch
- Variables in registers, not on the heap

112.5× is reasonable: Python's interpreted overhead is typically 50–200× vs. optimized C for compute-bound loops.

**Naive C → Tiled C: 33× speedup (400ms → 12ms)**

The naive C triple loop (`i, j, k`) has terrible cache behavior:
- Matrix A is accessed row-major (cache-friendly for the `k` inner loop)
- Matrix B is accessed column-major (cache-unfriendly): stepping through B[0][k], B[1][k], ... jumps by 512 floats = 2KB between accesses → constant L1 cache misses

For 512×512 float32:
- Each matrix = 512 × 512 × 4 bytes = 1MB
- Total working set = 3MB >> typical L1 cache (32–64KB)
- Every access to B[j][k] causes an L1 and L2 miss; L3 latency is ~40 cycles

**Cache tiling** (tile size T, typically 32–64):
- Divide the computation into T×T sub-problems
- A T×T tile of A: T² × 4 bytes = 4KB (fits in L1 with T=32)
- A T×T tile of B: also 4KB — both tiles fit in L1 simultaneously
- **Reuse ratio:** each element of A's tile is reused T times (once per column of B's tile), and vice versa — T² multiplications from 2T loads

With T=32 and L1 hit latency of 4 cycles vs. L3 latency of 40 cycles, the speedup from eliminating L3 misses is roughly 40/4 = 10×. The factor of 33× observed includes both reduced memory traffic and better prefetcher behavior on linear tile scans.

**Tiled C → Tiled + AVX2: 3× speedup (12ms → 4ms)**

After tiling, memory is no longer the bottleneck — compute is. A scalar fmul takes 1 cycle with 1 float/cycle throughput. AVX2 operates on 256-bit registers = 8 float32 values simultaneously. The theoretical speedup is 8×.

Observed 3×: not the full 8× because:
1. Loop overhead and setup: the AVX2 inner loop still has prologue/epilogue scalar fallback for non-multiple-of-8 dimensions
2. Register pressure: with 8 tiles of C accumulated in ymm registers, we're near the 16 ymm register limit
3. Store throughput: writing back accumulated C tiles requires `_mm256_storeu_ps` stores, which have limited throughput

With AVX-512 (16 float32/cycle), theoretical speedup over scalar is 16×. Production BLAS achieves ~14× of theoretical with careful micro-kernel design.

**Tiled AVX2 → NumPy/OpenBLAS: -2× (4ms → 8ms)**

Counterintuitively, NumPy is slower than the hand-written AVX2 code here. See part (b).

### Part (b): Why NumPy Is Slower Than Hand-Tiled AVX2

For a 512×512 matrix, NumPy calls OpenBLAS (`dgemm`/`sgemm`). OpenBLAS is heavily optimized for **large** matrices. Its overhead for a 512×512 multiply includes:
1. **Function call overhead:** OpenBLAS entry point performs matrix size analysis, picks a kernel variant, sets up threading (OpenBLAS may spawn threads even for small matrices)
2. **Memory packing:** OpenBLAS packs input matrices into a specific memory layout for optimal SIMD access — this itself takes ~0.5ms for a 1MB matrix
3. **Thread pool overhead:** For a 512×512 matrix, single-threaded computation is ~4ms, but OpenBLAS might spin up 4 threads, each doing 1ms of work + ~1ms of thread synchronization overhead = ~2ms overhead + 1ms compute = 3ms, worse than single-threaded

The hand-written tiled AVX2 code avoids packing and threading overhead at this matrix size, hence it's 2× faster.

At **larger matrices** (4096×4096), the story reverses: OpenBLAS is dramatically faster because packing amortizes its cost, threading provides near-linear speedup, and OpenBLAS's multi-level tiling (L1/L2/L3-aware) outperforms a single-level tile.

### Part (c): BLAS Packing

BLAS "packing" rearranges matrix panels into a custom memory layout for optimal SIMD access:

**Naive layout (row-major):**
```
Row 0:  A[0][0]  A[0][1]  A[0][2]  A[0][3]  ...  A[0][511]
Row 1:  A[1][0]  A[1][1]  A[1][2]  A[1][3]  ...  A[1][511]
```

**Packed micro-kernel panel layout (MR × KC block, e.g., 6×256):**
```
Packed:  A[0][0]  A[1][0]  A[2][0]  A[3][0]  A[4][0]  A[5][0]
         A[0][1]  A[1][1]  A[2][1]  A[3][1]  A[4][1]  A[5][1]
         ...
```

This rearrangement places elements that the SIMD micro-kernel accesses in a single register-width stride into contiguous memory. The benefit: when the inner loop loads a column of A's panel, all 6 (or 8 for AVX2, 16 for AVX-512) elements are adjacent — one cache line fetch per load, no strides.

Without packing, a column-major access to a panel needs elements spaced 512 floats × 4 bytes = 2KB apart — one cache line per element, 8 cache lines for 8 elements. With packing, 8 elements per cache line × 1 cache line = 1 fetch.

Packing cost: a single O(MN) pass over both input matrices. Worthwhile only when the matrix is large enough that the inner computation (O(MNK)) amortizes the packing cost.

### Part (d): `perf stat` Table

| Metric | Python | Naive C | Tiled C | Tiled AVX2 | NumPy |
|---|---|---|---|---|---|
| Time | ~45s | 400ms | 12ms | 4ms | 8ms |
| Instructions | ~20B | 1.5B | 150M | 50M | 60M |
| IPC | ~0.3 | ~1.5 | ~2.0 | ~3.5 | ~4.0 |
| L1-dcache-misses | low* | 95M | 8M | 2M | 3M |
| L3-cache-misses | low* | 45M | 1.5M | 0.3M | 0.5M |
| cache-miss rate | — | ~30% | ~5% | ~1.5% | ~1% |
| FLOPS/cycle | ~0.01 | ~0.15 | ~1.0 | ~5.5 | ~7.0 |

*Python spends most cycles in the interpreter, not memory accesses

**Key observations:**
- Naive C → Tiled C: L3 misses drop 30×, matching the speedup
- Tiled C → Tiled AVX2: L3 misses only drop 5×, but instructions drop 3× and FLOPS/cycle jump 5.5×
- NumPy: fewer cache misses than Tiled AVX2 (better multi-level tiling) but more overhead in function call setup

---

## 3. Full Python Benchmarking Harness

```python
"""
Matrix multiplication benchmarking harness.

Compares Python, NumPy, and ctypes-based C implementations.
Measures wall time, estimates FLOPS, and reports relative speedups.

Requires: numpy
Optional: ctypes C extension (compile with: gcc -O3 -mavx2 -shared -fPIC -o matmul.so matmul.c)
"""

from __future__ import annotations
import time
import math
import sys
import ctypes
import os
from typing import Callable, List, Optional
import numpy as np


# ── Matrix multiply implementations ────────────────────────────────────────────

def matmul_python_naive(A: list, B: list, n: int) -> list:
    """
    Pure Python triple loop. Intentionally the worst case.
    For n=64 this takes ~5 seconds; for n=512, ~45 minutes (extrapolated).
    """
    C = [[0.0] * n for _ in range(n)]
    for i in range(n):
        for j in range(n):
            s = 0.0
            for k in range(n):
                s += A[i][k] * B[k][j]
            C[i][j] = s
    return C


def matmul_python_transpose(A_np: np.ndarray, n: int) -> np.ndarray:
    """
    Python loop but with B transposed to improve cache access pattern.
    Still slow, but avoids the column-strided access in the inner loop.
    """
    B_T = A_np.T.copy()  # transpose B for row-major access
    C = np.zeros((n, n), dtype=np.float32)
    for i in range(n):
        for j in range(n):
            C[i, j] = np.dot(A_np[i], B_T[j])  # uses SIMD internally
    return C


def matmul_numpy(A: np.ndarray, B: np.ndarray) -> np.ndarray:
    """NumPy (OpenBLAS) matrix multiply — the production baseline."""
    return A @ B


def matmul_numpy_tiled(A: np.ndarray, B: np.ndarray, tile: int = 64) -> np.ndarray:
    """
    Tiled matrix multiply using NumPy slice operations.
    Demonstrates tiling concept without C extension.
    """
    n = A.shape[0]
    C = np.zeros((n, n), dtype=A.dtype)
    for i0 in range(0, n, tile):
        for j0 in range(0, n, tile):
            for k0 in range(0, n, tile):
                ie, je, ke = min(i0+tile, n), min(j0+tile, n), min(k0+tile, n)
                C[i0:ie, j0:je] += A[i0:ie, k0:ke] @ B[k0:ke, j0:je]
    return C


# ── C extension loader (optional) ─────────────────────────────────────────────

C_SOURCE = r"""
// matmul.c — compile with: gcc -O3 -mavx2 -shared -fPIC -o /tmp/matmul.so matmul.c
#include <stdlib.h>
#include <string.h>

#define TILE 32

void matmul_naive(const float* A, const float* B, float* C, int n) {
    memset(C, 0, n*n*sizeof(float));
    for (int i = 0; i < n; i++)
        for (int k = 0; k < n; k++)
            for (int j = 0; j < n; j++)
                C[i*n+j] += A[i*n+k] * B[k*n+j];
}

void matmul_tiled(const float* A, const float* B, float* C, int n) {
    memset(C, 0, n*n*sizeof(float));
    for (int i0 = 0; i0 < n; i0 += TILE)
    for (int k0 = 0; k0 < n; k0 += TILE)
    for (int j0 = 0; j0 < n; j0 += TILE) {
        int imax = i0+TILE < n ? i0+TILE : n;
        int kmax = k0+TILE < n ? k0+TILE : n;
        int jmax = j0+TILE < n ? j0+TILE : n;
        for (int i = i0; i < imax; i++)
        for (int k = k0; k < kmax; k++) {
            float aik = A[i*n+k];
            for (int j = j0; j < jmax; j++)
                C[i*n+j] += aik * B[k*n+j];
        }
    }
}
"""

def compile_c_extension() -> Optional[ctypes.CDLL]:
    """Try to compile the C extension. Returns None if compilation fails."""
    src_path = "/tmp/matmul_bench.c"
    so_path  = "/tmp/matmul_bench.so"

    try:
        with open(src_path, "w") as f:
            f.write(C_SOURCE)
        ret = os.system(f"gcc -O3 -mavx2 -shared -fPIC -o {so_path} {src_path} 2>/dev/null")
        if ret != 0:
            return None
        lib = ctypes.CDLL(so_path)
        # Set argument types
        for fn_name in ["matmul_naive", "matmul_tiled"]:
            fn = getattr(lib, fn_name)
            fn.restype = None
            fn.argtypes = [
                ctypes.POINTER(ctypes.c_float),
                ctypes.POINTER(ctypes.c_float),
                ctypes.POINTER(ctypes.c_float),
                ctypes.c_int,
            ]
        return lib
    except Exception:
        return None


def make_ctypes_wrapper(lib: ctypes.CDLL, fn_name: str):
    """Return a callable that runs a C matmul function on numpy arrays."""
    fn = getattr(lib, fn_name)

    def wrapper(A: np.ndarray, B: np.ndarray) -> np.ndarray:
        n = A.shape[0]
        A_c = A.astype(np.float32, order='C', copy=False)
        B_c = B.astype(np.float32, order='C', copy=False)
        C_c = np.zeros((n, n), dtype=np.float32, order='C')
        fn(
            A_c.ctypes.data_as(ctypes.POINTER(ctypes.c_float)),
            B_c.ctypes.data_as(ctypes.POINTER(ctypes.c_float)),
            C_c.ctypes.data_as(ctypes.POINTER(ctypes.c_float)),
            ctypes.c_int(n),
        )
        return C_c

    wrapper.__name__ = fn_name
    return wrapper


# ── Benchmarking infrastructure ────────────────────────────────────────────────

def bench(fn: Callable, *args, n_runs: int = 5, warmup: int = 2) -> float:
    """
    Benchmark fn(*args). Returns median wall time in milliseconds.
    Runs `warmup` iterations to warm caches, then `n_runs` timed iterations.
    """
    for _ in range(warmup):
        fn(*args)

    times = []
    for _ in range(n_runs):
        t0 = time.perf_counter()
        fn(*args)
        times.append((time.perf_counter() - t0) * 1000)

    times.sort()
    return times[len(times) // 2]  # median


def flops_for_matmul(n: int) -> int:
    """Standard matmul FLOP count: 2*n^3 (n^2 dot products of length n, each is n muls + n-1 adds)."""
    return 2 * n ** 3


def run_benchmark(n: int = 256):
    """
    Run the full benchmark suite for n×n float32 matrix multiplication.
    Uses n=256 by default (faster for demo; set n=512 for the exercise scenario).
    """
    print(f"=" * 65)
    print(f"Matrix Multiply Benchmark: {n}×{n} float32")
    print(f"=" * 65)

    rng = np.random.default_rng(42)
    A = rng.random((n, n), dtype=np.float32)
    B = rng.random((n, n), dtype=np.float32)

    reference = A @ B  # ground truth
    flops = flops_for_matmul(n)
    results = []

    # ── NumPy (OpenBLAS) ──────────────────────────────────────────────────────
    t = bench(matmul_numpy, A, B)
    gflops = flops / (t / 1000) / 1e9
    results.append(("NumPy (OpenBLAS)", t, gflops))
    print(f"  NumPy (OpenBLAS):      {t:8.1f} ms  |  {gflops:.1f} GFLOPS")

    # ── NumPy tiled ───────────────────────────────────────────────────────────
    C_tiled = matmul_numpy_tiled(A, B, tile=32)
    np.testing.assert_allclose(C_tiled, reference, rtol=1e-4, atol=1e-4)
    t = bench(matmul_numpy_tiled, A, B, tile=32)
    gflops = flops / (t / 1000) / 1e9
    results.append(("NumPy tiled (tile=32)", t, gflops))
    print(f"  NumPy tiled (tile=32): {t:8.1f} ms  |  {gflops:.1f} GFLOPS")

    # ── C extension ──────────────────────────────────────────────────────────
    lib = compile_c_extension()
    if lib is not None:
        for fn_name, label in [("matmul_naive", "C naive"),
                                ("matmul_tiled", "C tiled (tile=32)")]:
            fn = make_ctypes_wrapper(lib, fn_name)
            C_check = fn(A, B)
            np.testing.assert_allclose(C_check, reference, rtol=1e-3, atol=1e-3)
            t = bench(fn, A, B)
            gflops = flops / (t / 1000) / 1e9
            results.append((label, t, gflops))
            print(f"  {label:<22} {t:8.1f} ms  |  {gflops:.1f} GFLOPS")
    else:
        print("  [C extension not compiled — install gcc to enable]")

    # ── Python (sampled) ─────────────────────────────────────────────────────
    n_small = min(n, 32)   # Python loops over 32×32 and extrapolate
    A_small = A[:n_small, :n_small]
    B_small = B[:n_small, :n_small]
    A_list  = A_small.tolist()
    B_list  = B_small.tolist()
    t_small = bench(matmul_python_naive, A_list, B_list, n_small, n_runs=3, warmup=1)
    # Extrapolate: O(n^3), so t(n) = t(n_small) * (n/n_small)^3
    t_py = t_small * (n / n_small) ** 3
    gflops_py = flops / (t_py / 1000) / 1e9
    results.append(("Python naive (extrapolated)", t_py, gflops_py))
    print(f"  Python naive (extrap):  {t_py:8.0f} ms  |  {gflops_py:.4f} GFLOPS")

    # ── Summary table ────────────────────────────────────────────────────────
    print(f"\n{'Implementation':<28} {'Time (ms)':>10} {'GFLOPS':>8} {'vs NumPy':>10}")
    print("-" * 60)
    numpy_t = results[0][1]
    for label, t, g in results:
        ratio = t / numpy_t
        print(f"  {label:<26} {t:>10.1f} {g:>8.2f} {ratio:>8.1f}×")

    print(f"\n  Theoretical peak (AVX2, 1 core, 3.0GHz): {8*2*3.0:.0f} GFLOPS")
    print(f"    (8 float32/ymm × 2 FMA operands × 3.0 GHz)")

    # ── Speedup breakdown ────────────────────────────────────────────────────
    print(f"\nSpeedup Analysis:")
    py_t    = results[-1][1]
    numpy_t = results[0][1]
    if lib is not None and len(results) >= 5:
        c_naive  = next(r[1] for r in results if 'C naive' in r[0])
        c_tiled  = next(r[1] for r in results if 'C tiled' in r[0])
        print(f"  Python→C naive:    {py_t/c_naive:>6.0f}×  (no interpreter, no boxing)")
        print(f"  C naive→C tiled:   {c_naive/c_tiled:>6.0f}×  (cache blocking, better reuse)")
        print(f"  C tiled→NumPy:     {c_tiled/numpy_t:>6.2f}×  "
              f"({'faster' if c_tiled > numpy_t else 'slower'}: packing + threading tradeoff)")
    print(f"  Python→NumPy:      {py_t/numpy_t:>6.0f}×")

    return results


# ── Simulated perf stat output ────────────────────────────────────────────────

def simulated_perf_stat_table(n: int = 512):
    """
    Print a simulated perf stat comparison table.
    Values are derived from the analysis in part (d) of the exercise.
    """
    print(f"\n{'=' * 65}")
    print(f"Simulated perf stat for {n}×{n} float32 matmul")
    print(f"{'=' * 65}")
    print(f"\n{'Metric':<28} {'Python':>10} {'C naive':>10} {'C tiled':>10} {'AVX2':>10} {'NumPy':>10}")
    print("-" * 75)

    rows = [
        ("Time (ms)",            "~45000", "~400",  "~12",   "~4",    "~8"),
        ("Instructions (M)",     "~20000", "~1500", "~150",  "~50",   "~60"),
        ("IPC",                  "~0.30",  "~1.50", "~2.00", "~3.50", "~4.00"),
        ("L1-cache-miss (M)",    "—",      "~95",   "~8",    "~2",    "~3"),
        ("L3-cache-miss (M)",    "—",      "~45",   "~1.5",  "~0.3",  "~0.5"),
        ("Cache miss rate",      "—",      "~30%",  "~5%",   "~1.5%", "~1%"),
        ("GFLOPS",               "~0.0003","~0.07", "~2.2",  "~6.7",  "~3.4"),
        ("Peak utilization",     "~0%",    "~0.5%", "~14%",  "~42%",  "~21%"),
    ]
    for row in rows:
        label = row[0]
        vals  = row[1:]
        print(f"  {label:<26} " + "  ".join(f"{v:>10}" for v in vals))

    print(f"\n  Peak: AVX2 at 3.0 GHz = 48 GFLOPS (8 FP32/cycle × 2 FMA × 3 GHz)")
    print(f"  Best observed: ~6.7 GFLOPS (14% of peak) — typical for single-core SIMD")
    print(f"  NumPy at 3.4 GFLOPS: packing overhead lowers effective throughput for small N")


if __name__ == "__main__":
    # Default to n=256 for speed; change to 512 for the exact exercise scenario
    n = int(sys.argv[1]) if len(sys.argv) > 1 else 256
    results = run_benchmark(n=n)
    simulated_perf_stat_table(n=512)
```

---

## 4. Common Wrong Approaches and Why They Fail

### Wrong Approach 1: Attributing the Python–C Gap to "Python is Interpreted"

**Mistake:** "Python is 100× slower because it's an interpreted language, not compiled."

**Why it fails (incomplete):** Interpretation overhead is a factor, but the dominant costs are:
1. **Object allocation:** Each Python float is a `PyObject` on the heap (~28 bytes on 64-bit CPython). Multiplying two floats allocates a new PyObject for the result. For 134M multiplications, this is 134M × 28 bytes = 3.75GB of allocation traffic.
2. **GC pressure:** The cyclic garbage collector runs every ~700 allocations by default; with 134M allocations, GC runs ~190,000 times during the computation.
3. **Bytecode dispatch:** Each `a[i][k] * b[k][j]` requires 8+ bytecode instructions (LOAD_FAST, BINARY_SUBSCR × 2, BINARY_OP, STORE_FAST, etc.).

The full 112.5× gap cannot be explained by interpretation overhead alone.

### Wrong Approach 2: Using np.dot Instead of BLAS sgemm for Benchmarking

**Mistake:** "I'll benchmark with `np.dot(A, B)` on a loop over sub-matrices to measure tiled performance."

**Why it fails:** `np.dot` (for 2D arrays) calls the same BLAS `gemm` routine as `@`. You're measuring the BLAS overhead repeatedly, not tiling overhead. To isolate tiling effects, you need a custom C implementation where you control the tile structure, or NumPy's `einsum` with explicitly contracted indices.

### Wrong Approach 3: Picking Tile Size Based on L1 Cache Size Alone

**Mistake:** "L1 is 32KB, float32 is 4 bytes, so tile = sqrt(32768/4/3) = 52. I'll use T=52."

**Why it fails:** The optimal tile size accounts for:
1. **Three tiles simultaneously:** A-tile, B-tile, and C-tile must all fit in L1 (3T² × 4 bytes)
2. **Register pressure:** The compiler needs to keep the C-tile in registers across the inner loop. With T=16, that's 16×16=256 floats → 32 ymm registers (only 16 available). T=6 is typical for the register-bound micro-kernel.
3. **L2/L3 hierarchy:** A proper tiling uses *three* tile levels (L1, L2, L3), not one.

Optimal BLAS implementations use T_L1 = 6 (register kernel), T_L2 = 256 (L2 cache), T_L3 = 4096 (L3 cache).

### Wrong Approach 4: Assuming AVX2 = 8× Speedup Over Scalar

**Mistake:** "AVX2 gives 8 float32/cycle vs. scalar 1 float32/cycle, so the speedup must be 8×."

**Why it fails:** Achieved throughput depends on:
1. **Instruction throughput vs. latency:** `vfmadd` has latency 4 but throughput 0.5 (on Skylake). Sustained 8 FLOPS/cycle requires 4-deep software pipelining of the FMA units.
2. **Memory bandwidth:** Even with tiling, if the L1/L2 bandwidth is the bottleneck (not the FMA units), vectorization doesn't help.
3. **Register allocation:** Accumulating 8 ymm registers for 8 concurrent C-tile values requires 8 + 1 + 1 = 10 ymm registers just for the kernel, plus registers for loop counters and pointers.

Practical AVX2 matmul achieves 60–85% of theoretical peak with careful micro-kernel design; naive vectorization yields 20–40%.

### Wrong Approach 5: Benchmarking with Wall Clock Without Warmup

**Mistake:** "I'll measure the first matrix multiplication after program startup."

**Why it fails:** Cold cache behavior dramatically changes the result:
- First call to NumPy may trigger shared library loading (~10ms on first call)
- Cold L3 cache: the 1MB matrices must be fetched from DRAM, adding ~0.5ms per matrix (2GB/s bandwidth × 1MB = 0.5ms)
- OS transparent huge pages may not have allocated yet

Always warm up with 2–5 iterations and take the median of subsequent runs, not the minimum (min is misleading — it captures the best-case OS scheduling, not typical performance).

---

## 5. Extensions

### 1,000-Node Distributed Matrix Multiply

At 1,000 nodes, matrix multiplication can be distributed using Cannon's algorithm or SUMMA (Scalable Universal Matrix Multiply Algorithm). SUMMA uses a virtual P×Q processor grid; each processor holds a submatrix of A and B, broadcasts panels across rows and columns, and accumulates local products.

**Communication complexity:** SUMMA with P = 1000 processes, n = 512K × 512K: each process sends O(n²/P) data per broadcast step. Total communication = O(n² × log P) — comparable to a single matrix at large scale.

Production use: ScaLAPACK (MPI-based), Elemental (distributed linear algebra), NVIDIA NCCL's collective operations for GPU-distributed matmul.

### 1TB Matrix

A 1TB float32 matrix has dimensions n = sqrt(1TB / 4 bytes) = sqrt(2.5 × 10¹¹) ≈ 500,000. Multiplying two 500K×500K matrices would require 2 × (500K)³ FLOPs = 2.5 × 10¹⁷ FLOPs. At 1 PFLOPS (10¹⁵ FLOPS/s, achievable with a modern GPU cluster): 250 seconds.

Storage: 3 matrices × 1TB = 3TB. Memory: no GPU has 3TB DRAM. Solution: **out-of-core** matmul via tiled streaming from NVMe storage + pipelined compute.

### 10ms Latency Budget

For a 10ms inference latency budget with a 512×512 weight matrix:
- A single GPU V100: one `cublasSgemm(n=512)` takes ~0.01ms — 1000× faster than needed
- The bottleneck at 10ms is typically the data pipeline: loading inputs, preprocessing, and returning results via gRPC/HTTP
- Batching: serving 64 requests simultaneously with the same matrix multiply takes only marginally longer (0.05ms) — batch for throughput, not latency

### Adversarial: Cache-Thrashing Input

If an adversary constructs input that defeats the hardware prefetcher (random access pattern), tiling loses some benefit. Countermeasure: sort matrix rows by a space-filling curve (Hilbert curve) to preserve locality under random access patterns. Used in sparse matrix formats (Morton order) and spatial databases.

---

## 6. Real-World Production System References

### OpenBLAS: `kernel/x86_64/sgemm_kernel_8x8_sandy.S`

OpenBLAS's hand-written AVX2 SGEMM micro-kernel is in the `kernel/x86_64/` directory. The 8×4 kernel (`sgemm_kernel_8x4_haswell.S`) uses 6 ymm registers for C-tile accumulation, 1 for A-panel broadcast, 1 for B-panel load — exactly 8 of 16 available ymm registers. The kernel achieves ~95% of theoretical peak on Intel Haswell by carefully interleaving FMA latency with load instructions.

### BLIS (BLAS-Like Library Instantiation Software)

BLIS (`github.com/flame/blis`) is the reference implementation of the BLAS methodology. Its framework explicitly separates packing (into its `packA`/`packB` routines), caching (L1/L2/L3 tiling parameters), and micro-kernel (architecture-specific assembly). The `bli_gemm_front()` entry point orchestrates all three levels.

BLIS's tiling parameters for Skylake: MC=256 (L2 cache), KC=256 (L2 cache), NC=4096 (L3 cache), MR=24 (register block), NR=8 (register block).

### Google's TPU and XLA Compiler

Google's XLA (Accelerated Linear Algebra) compiler performs matrix multiply optimization as a first-class concern. XLA's `GemmFusion` pass fuses adjacent matmul operations to avoid materializing intermediate matrices. XLA's `DotOperationRewriter` implements multi-level tiling for TPUs, where the tile sizes are hardware-dictated (TPU tile = 128×128 for TPU v2).

### NVIDIA's cuBLAS and Tensor Cores

cuBLAS achieves near-peak performance on NVIDIA GPUs using Tensor Cores (matrix multiply accelerators in each SM). For float16 matmul on A100: 312 TFLOPS theoretical, ~250 TFLOPS observed. cuBLAS uses CUTLASS (CUDA Templates for Linear Algebra Subroutines) for tile-based computation, with tile sizes matching shared memory (48KB per SM) and warp-level MMA (matrix multiply-accumulate) instructions.

Source: NVIDIA's open-source CUTLASS library demonstrates the exact tiling hierarchy described in this solution.

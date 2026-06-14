# Lesson 10: GPU Architecture and CUDA Internals
### Warps, Tensor Cores, Memory Hierarchy, and High-Performance Kernel Engineering

> **Prerequisites:** Familiarity with the CUDA Programming Guide, basic ML training loops, and Python. This lesson targets engineers who want to understand *what hardware actually does* when your kernel launches — not just how to write valid CUDA code, but how to make it fast.

---

## 1. GPU Architecture — The Big Picture

### The Fundamental Philosophy: Latency vs. Throughput

A modern CPU (say, an Intel Xeon Platinum 8380 with 40 cores) is an extraordinary single-thread performance machine. Every design choice — out-of-order (OOO) execution with a 512-entry reorder buffer, a 3-level cache hierarchy up to 60MB of L3, sophisticated branch predictors with 97%+ accuracy, speculative execution, register renaming — exists to make ONE thread finish as fast as possible. A CPU core can execute up to 5 instructions per cycle even when they have dependencies, because the OOO engine looks ahead, finds independent instructions, and issues them simultaneously. The chip area dedicated to this logic dwarfs the area dedicated to actual arithmetic units.

A GPU makes the opposite bet. On an NVIDIA A100:
- **6,912 CUDA cores** (FP32 units)
- **432 Tensor Cores** (3rd generation)
- **108 Streaming Multiprocessors (SMs)**
- **80 GB HBM2e** at **2 TB/s** memory bandwidth
- No out-of-order execution. No branch prediction. No speculative execution.

The GPU hides latency not by eliminating it, but by *having so many independent threads* that when one group stalls waiting for memory, another group is ready to execute. This is called **latency hiding through context switching**, and it is the entire basis of GPU performance. If you don't have enough independent work to fill the machine, a GPU performs worse than a CPU.

### Streaming Multiprocessor (SM): The Fundamental Compute Unit

Every GPU operation ultimately executes on an SM. The A100 has 108 SMs. Each SM is a self-contained compute cluster containing:

| Resource | A100 per SM |
|---|---|
| Warp schedulers | 4 |
| FP32 CUDA cores | 64 (256 total with 4 schedulers × 64... actually 128 FP32/SM but dual-issued, effectively 64 per scheduler) |
| FP64 cores | 32 |
| INT32 cores | 64 |
| Tensor Core units | 4 (3rd gen, handling MMA operations) |
| L1 cache / Shared memory | 192 KB (configurable split) |
| Register file | 256 KB = 65,536 × 32-bit registers |
| Max warps | 64 |
| Max threads | 2,048 |
| Max blocks | 32 |

The register file is the most critical resource for occupancy. With 65,536 registers and 2,048 maximum threads, each thread can use up to 32 registers at 100% occupancy. A kernel using 64 registers per thread can only run 1,024 threads (half occupancy). A kernel using 128 registers: 512 threads (quarter occupancy).

### The Warp: The Fundamental Scheduling Unit

A **warp** is 32 threads that execute in lockstep under the SIMT (Single Instruction, Multiple Thread) execution model. This is the GPU's version of SIMD: one instruction fetched, 32 data lanes executing it simultaneously.

The 4 warp schedulers in each SM each track up to 16 warps (64 total per SM). Every clock cycle, each scheduler:
1. Examines all its assigned warps
2. Selects one warp that is **ready** (all operands available, no outstanding memory operations)
3. Issues that warp's next instruction to the execution units

A warp becomes **stalled** when it's waiting on a memory transaction, a long-latency arithmetic operation (e.g., division, transcendental), or a barrier. The scheduler simply picks a different ready warp. This zero-overhead context switching is what makes GPUs work: global memory access takes ~300 clock cycles, but if there are enough warps, the SM stays busy the entire time.

### Thread Hierarchy

```
Grid
└── Block (up to 1024 threads, assigned to one SM)
    └── Warp (32 threads, fundamental scheduling unit)
        └── Thread (one SIMT lane)
```

A block is the unit of resource allocation. When a block is launched, the SM reserves registers for ALL threads in that block simultaneously (even if only one warp is executing). This is why register pressure matters so much: a block of 256 threads using 64 registers/thread consumes 256×64 = 16,384 registers — 25% of the SM's register file, limiting you to 4 simultaneous blocks per SM.

Blocks are assigned to SMs by the hardware scheduler. Multiple blocks can run on one SM simultaneously if resources permit. All blocks on one SM share that SM's shared memory.

---

## 2. Memory Hierarchy

Understanding the memory hierarchy is the single most important skill for writing fast GPU kernels. Every performance problem in GPU code ultimately traces back to memory.

### Register File
- **Location:** On-chip, inside each SM
- **Size:** 65,536 × 32-bit registers per SM (256 KB)
- **Latency:** 0 extra cycles (reads available next cycle after write, with bypassing)
- **Scope:** Private to each thread — no other thread can read your registers
- **Critical constraint:** Register spilling — when a kernel needs more registers than available, the compiler spills to local memory, which maps to L1 cache or (worst case) global memory. A 100x latency penalty.

### Shared Memory
- **Location:** On-chip SRAM, inside each SM
- **Size:** 192 KB per SM, configurable as L1 cache / shared memory split (e.g., 128KB shared + 64KB L1)
- **Latency:** ~5 ns (~23 cycles at 1.41 GHz A100 SM clock)
- **Scope:** Shared among ALL threads in a block. The primary communication mechanism between threads.
- **Bandwidth:** 19 TB/s aggregate across all SMs (A100)

Shared memory is divided into **32 banks**, each 4 bytes wide, accessed in a round-robin pattern. Address `a` maps to bank `a/4 % 32`. When all 32 threads in a warp access different banks, the access is served in one cycle. When two threads access the same bank (but different addresses), the accesses are **serialized** — a bank conflict. A worst-case 32-way bank conflict takes 32 cycles instead of 1. The exception: if ALL threads access the SAME address, the hardware broadcasts — one cycle, no penalty.

**Bank conflict example:**
```python
# No conflict: thread i accesses address i (different banks)
# sA[threadIdx.x]  → banks 0,1,2,...,31

# 2-way conflict: thread i accesses address 2*i
# Threads 0,16 → bank 0; threads 1,17 → bank 1; etc.
# sA[threadIdx.x * 2]

# 32-way conflict: all threads same bank
# sA[threadIdx.x * 32]  → all map to bank 0 (32*4 = 128, 128/4 % 32 = 0)
```

### L1 Cache and L2 Cache
- **L1:** 192 KB per SM (shared with shared memory). Caches global memory accesses with spatial locality. Texture memory is also cached here with 2D spatial locality guarantees.
- **L2:** 40 MB shared across all SMs on A100 (vs 6 MB on V100 — a massive upgrade). ~30 ns latency. Caches both global memory reads and atomics. With 40 MB, a transformer's KV cache for short sequences may fit entirely in L2.

### Global Memory (HBM2e)
- **Size:** 80 GB
- **Latency:** ~300–400 ns (~400+ cycles)
- **Bandwidth:** 2 TB/s (peak)
- **Access pattern matters enormously:** Coalesced access approaches peak bandwidth. Strided or random access can achieve <1% of peak bandwidth.

### Pinned (Page-Locked) Memory
Normal CPU memory allocated with `malloc` is pageable — the OS can swap it to disk. The GPU DMA engine cannot transfer data to/from pageable memory without first copying it to a staging buffer. **Pinned memory** is locked in physical RAM; the DMA engine can transfer directly at ~12–24 GB/s (PCIe 4.0 limit).

```python
import torch
import time

size = 1024 * 1024 * 256  # 256M floats = 1 GB

# Regular (pageable) memory
regular = torch.zeros(size)
gpu = torch.zeros(size, device='cuda')

t0 = time.perf_counter()
for _ in range(10):
    gpu.copy_(regular)
torch.cuda.synchronize()
pageable_bw = size * 4 / ((time.perf_counter() - t0) / 10) / 1e9
print(f"Pageable H2D: {pageable_bw:.1f} GB/s")

# Pinned memory
pinned = torch.zeros(size, pin_memory=True)
t0 = time.perf_counter()
for _ in range(10):
    gpu.copy_(pinned)
torch.cuda.synchronize()
pinned_bw = size * 4 / ((time.perf_counter() - t0) / 10) / 1e9
print(f"Pinned H2D: {pinned_bw:.1f} GB/s")
# Pinned is typically 2-4x faster
```

### CUDA Streams and Overlapping
A CUDA stream is an ordered queue of GPU operations. Operations in the same stream execute in order; operations in different streams can overlap:

```python
import torch

stream1 = torch.cuda.Stream()
stream2 = torch.cuda.Stream()

A = torch.randn(4096, 4096, device='cuda', pin_memory=False)
B = torch.randn(4096, 4096, device='cuda', pin_memory=False)
pinned_A = torch.randn(4096, 4096, pin_memory=True)

with torch.cuda.stream(stream1):
    # Kernel on stream 1
    C = A @ B

with torch.cuda.stream(stream2):
    # H2D transfer on stream 2 — can overlap with the matmul above
    gpu_A = pinned_A.cuda(non_blocking=True)

torch.cuda.synchronize()
```

This enables the classic **double-buffering** pattern: while the GPU processes batch N, copy batch N+1 to GPU memory simultaneously.

---

## 3. Warp Execution and Divergence

### SIMT Execution Model

SIMT is conceptually similar to SIMD (Single Instruction, Multiple Data), but with an important twist: each thread has its own **program counter**, its own **registers**, and its own **predicate registers** for masking. This lets threads theoretically take different code paths — but at a cost.

When all 32 threads in a warp take the same branch, execution is perfectly efficient: one instruction issued, 32 results produced. When threads diverge, the warp controller serializes the paths, masking inactive threads:

```python
from numba import cuda
import numpy as np
import time

@cuda.jit
def divergent_kernel(x, out):
    i = cuda.grid(1)
    if i < x.shape[0]:
        if x[i] % 2 == 0:   # Thread 0,2,4... take this path
            out[i] = x[i] * x[i]
        else:                 # Thread 1,3,5... take this path
            out[i] = x[i] + 1
        # Both paths execute serially; half the lanes are masked each time

@cuda.jit
def coherent_kernel(x, out):
    i = cuda.grid(1)
    if i < x.shape[0]:
        # Mathematically equivalent but computed without branching
        is_even = float32(1.0 - (x[i] % 2))
        squared = x[i] * x[i]
        incremented = x[i] + 1
        out[i] = is_even * squared + (1.0 - is_even) * incremented

N = 1 << 24  # 16M elements
x_host = np.arange(N, dtype=np.int32)
x_dev = cuda.to_device(x_host)
out_dev = cuda.device_array(N, dtype=np.int32)

threads_per_block = 256
blocks = (N + threads_per_block - 1) // threads_per_block

# Warm up
divergent_kernel[blocks, threads_per_block](x_dev, out_dev)
cuda.synchronize()

t0 = time.perf_counter()
for _ in range(100):
    divergent_kernel[blocks, threads_per_block](x_dev, out_dev)
cuda.synchronize()
print(f"Divergent:  {(time.perf_counter()-t0)/100*1000:.2f} ms")

t0 = time.perf_counter()
for _ in range(100):
    coherent_kernel[blocks, threads_per_block](x_dev, out_dev)
cuda.synchronize()
print(f"Coherent:   {(time.perf_counter()-t0)/100*1000:.2f} ms")
```

### Warp Divergence in Modern ML

Divergence is not just a CUDA textbook concern — it appears in real ML workloads:

1. **Attention masking:** Causal attention applies a mask where upper-triangular positions are set to -inf before softmax. Depending on implementation, this can cause divergent paths across the sequence length dimension.
2. **Dynamic control flow in MoE:** In Mixture-of-Experts routing, different tokens route to different experts. Naive implementation has severe warp divergence; state-of-the-art MoE kernels (like those in Megablocks) reorganize tokens to avoid it.
3. **Sparse operations:** SparseMM and similar kernels often have value-dependent branches — threads handling nonzero values do computation while threads on zeros are masked.

**Mitigation strategies:**
- Reorder data so threads in the same warp handle similar cases (warp-coalesced grouping).
- Replace branches with arithmetic (`mask * value + (1-mask) * other`).
- Use predicated instructions where the compiler naturally avoids divergence overhead.
- Kernel fusion: fusing an elementwise mask with the preceding matmul keeps results in registers and avoids re-loading them.

---

## 4. Memory Coalescing

Coalescing is probably the single most impactful optimization you can apply to a memory-bound kernel.

### The Coalescing Rule

When a warp issues a load, the GPU memory controller inspects the 32 addresses requested by the 32 threads. If those addresses fall within one or two 128-byte cache lines (i.e., they're consecutive), the controller issues **one memory transaction** serving all 32 threads. This is fully coalesced access.

If the addresses are scattered, the controller must issue one transaction per cache line touched — up to 32 transactions for 32 completely random addresses. You get 1/32 of peak bandwidth.

```python
import cupy as cp
import time

N = 1024 * 1024 * 64  # 64M float32 = 256 MB

a = cp.random.rand(N, dtype=cp.float32)
start = cp.cuda.Event()
end = cp.cuda.Event()

# --- Coalesced: sequential access ---
start.record()
for _ in range(100):
    result = a.sum()
end.record()
end.synchronize()
coalesced_ms = start.elapsed_time(end) / 100
bw_coalesced = (N * 4) / (coalesced_ms / 1000) / 1e9
print(f"Coalesced: {coalesced_ms:.2f} ms → {bw_coalesced:.0f} GB/s")

# --- Strided: every 32nd element ---
idx_stride32 = cp.arange(0, N, 32, dtype=cp.int32)
start.record()
for _ in range(100):
    result = a[idx_stride32].sum()
end.record()
end.synchronize()
strided_ms = start.elapsed_time(end) / 100
bw_strided = (len(idx_stride32) * 4) / (strided_ms / 1000) / 1e9
print(f"Stride-32: {strided_ms:.2f} ms → {bw_strided:.0f} GB/s (effective)")
print(f"Bandwidth regression factor: {bw_coalesced/bw_strided:.1f}x")
```

### Array of Structs vs. Struct of Arrays

This is the most common coalescing mistake in practice:

```python
import cupy as cp
import numpy as np

N = 1_000_000  # 1M particles

# AoS: bad for per-component access
# Layout in memory: [x0,y0,z0, x1,y1,z1, x2,y2,z2, ...]
# Thread i accessing x[i] → stride-3 access pattern
particles_AoS = cp.random.rand(N, 3).astype(cp.float32)

# SoA: good for per-component access
# Layout in memory: [x0,x1,...,xN, y0,y1,...,yN, z0,z1,...,zN]
# Thread i accessing x[i] → sequential (coalesced!)
x = cp.random.rand(N, dtype=cp.float32)
y = cp.random.rand(N, dtype=cp.float32)
z = cp.random.rand(N, dtype=cp.float32)

# Benchmark: sum all x-coordinates
start = cp.cuda.Event(); end = cp.cuda.Event()

start.record()
for _ in range(1000):
    res = particles_AoS[:, 0].sum()  # Strided access over AoS
end.record(); end.synchronize()
print(f"AoS x-sum: {start.elapsed_time(end)/1000:.3f} ms")

start.record()
for _ in range(1000):
    res = x.sum()  # Sequential access over SoA
end.record(); end.synchronize()
print(f"SoA x-sum: {start.elapsed_time(end)/1000:.3f} ms")
```

In particle simulations and ML feature processing, SoA can deliver 3–10x speedups over AoS when the access pattern is per-component.

---

## 5. Tensor Cores and Mixed Precision

### What Tensor Cores Actually Do

A 3rd-generation Tensor Core on the A100 performs a **warp-synchronous matrix multiply-accumulate (MMA)** operation:

```
D[m×n] = A[m×k] × B[k×n] + C[m×n]
```

At the PTX level, this is the `mma.sync.aligned.m16n8k16` instruction family. One Tensor Core processes a 16×8×16 MMA in one operation. With 4 Tensor Cores per SM and 108 SMs, the A100 achieves:

- **FP16/BF16:** 312 TFLOP/s
- **TF32:** 156 TFLOP/s (19.5 TFLOP/s without sparsity, ×8 with structured sparsity)
- **FP64:** 19.5 TFLOP/s
- **INT8:** 624 TOPS/s
- **INT4:** 1,248 TOPS/s (with structured sparsity)

### Numerical Formats Deep Dive

| Format | Sign | Exponent | Mantissa | Range | Notes |
|---|---|---|---|---|---|
| FP32 | 1 | 8 bits | 23 bits | ±3.4×10³⁸ | Standard precision |
| FP16 | 1 | 5 bits | 10 bits | ±65504 | Overflow risk in training |
| BF16 | 1 | 8 bits | 7 bits | ±3.4×10³⁸ | Same range as FP32, less precision |
| TF32 | 1 | 8 bits | 10 bits | ±3.4×10³⁸ | A100-internal for matmul only |

**Why BF16 for training:** FP16's limited exponent range (±65504) means that gradient values larger than 65504 or smaller than ~6×10⁻⁵ (with limited precision) either overflow or underflow. This requires loss scaling, which adds complexity. BF16 has the same exponent range as FP32, so gradients almost never overflow. The reduced mantissa precision (7 vs 23 bits) is acceptable for most gradient accumulation.

**TF32:** You don't write TF32. PyTorch/cuBLAS automatically uses TF32 for FP32 matmuls on A100 when `torch.backends.cuda.matmul.allow_tf32 = True`. Inputs are silently rounded to TF32 precision before the Tensor Core operation, and the result is stored in FP32. The speedup is real (3-8x over FP32 Tensor Cores) with minimal accuracy impact.

```python
import torch
import time

device = 'cuda'
N = 4096

A = torch.randn(N, N, device=device, dtype=torch.float32)
B = torch.randn(N, N, device=device, dtype=torch.float32)

def benchmark_matmul(A, B, label, n_warmup=10, n_iter=100):
    for _ in range(n_warmup):
        C = A @ B
    torch.cuda.synchronize()
    t0 = time.perf_counter()
    for _ in range(n_iter):
        C = A @ B
    torch.cuda.synchronize()
    elapsed_ms = (time.perf_counter() - t0) / n_iter * 1000
    flops = 2 * N**3  # N^3 multiplies + N^3 adds
    tflops = flops / (elapsed_ms / 1000) / 1e12
    print(f"{label:30s}: {elapsed_ms:.2f} ms  |  {tflops:.1f} TFLOP/s")
    return elapsed_ms

# FP32 without Tensor Cores (software fallback)
torch.backends.cuda.matmul.allow_tf32 = False
benchmark_matmul(A, B, "FP32 (no Tensor Core)")

# FP32 via TF32 Tensor Cores (A100 only)
torch.backends.cuda.matmul.allow_tf32 = True
benchmark_matmul(A, B, "TF32 (Tensor Core)")

# FP16 Tensor Cores
A16 = A.half()
B16 = B.half()
benchmark_matmul(A16, B16, "FP16 (Tensor Core)")

# BF16 Tensor Cores
Abf = A.bfloat16()
Bbf = B.bfloat16()
benchmark_matmul(Abf, Bbf, "BF16 (Tensor Core)")
```

### Tensor Core Requirements

To actually land in Tensor Core code paths:
1. Matrix dimensions must be **multiples of 16** (for FP16/BF16) or **multiples of 8** (for TF32).
2. For PyTorch: cuBLAS uses Tensor Cores automatically via `torch.matmul` or `torch.nn.Linear`.
3. For custom kernels: use the `wmma` API (CUDA C++) or the `mma.sync` PTX instruction. In Python (Numba), Tensor Core access is limited and requires careful warp-level programming.
4. For INT8: use `torch.quantization` or TensorRT, which invokes cuBLAS INT8 GEMM paths.

---

## 6. Occupancy and the Roofline Model

### Occupancy: What It Is and What It Isn't

**Occupancy** = (active warps per SM) / (maximum warps per SM).

On A100, max warps = 64. If your kernel launch has 32 resident warps per SM, occupancy = 50%.

Occupancy determines how many warps the scheduler has to choose from when hiding latency. With 100% occupancy, a 300-cycle memory stall can be hidden if there are 300/latency_per_instruction other warps to keep the SM busy. With 25% occupancy and a long-latency operation, the SM may go idle.

**But:** Occupancy is NOT a direct measure of performance. Consider two kernels:
- **Kernel A:** 100% occupancy, every other cycle is a global memory access (L1 miss). The SM is busy, but half the work is stall cycles.
- **Kernel B:** 25% occupancy, perfect register reuse, L1 hit rate 95%. Despite fewer warps, the SM issues useful instructions more often.

Kernel B often wins. The real goal is **maximizing ISSUED instructions per cycle**, not maximizing warps.

### Roofline Model

The roofline model characterizes any kernel by its **arithmetic intensity** (AI): FLOP per byte of DRAM traffic.

```
Arithmetic Intensity = (Total FLOPs) / (Total DRAM Bytes Read + Written)
```

Two hardware limits constrain performance:
1. **Peak compute:** P_compute TFLOP/s (e.g., 312 TFLOP/s FP16 on A100)
2. **Peak bandwidth:** P_bw TB/s (e.g., 2 TB/s on A100)

The **ridge point** is `P_compute / P_bw = 312 / 2 = 156 FLOP/byte`.

- If your kernel's AI < 156 FLOP/byte → **memory-bandwidth bound**. Adding faster cores won't help; you need better memory access patterns or fewer DRAM transactions.
- If your kernel's AI > 156 FLOP/byte → **compute bound**. Better memory access patterns don't help; you need faster arithmetic (Tensor Cores, INT8, etc.).

**Common ML operation arithmetic intensities:**
- Elementwise ReLU on N floats: 1 FLOP / 8 bytes (read+write) = **0.125 FLOP/byte** → severely memory bound
- LayerNorm: ~10 FLOPs / 8 bytes = **1.25 FLOP/byte** → memory bound
- Matmul (N×N square): 2N³ FLOPs / 3N² × 4 bytes = **2N/6 FLOP/byte** → for N=4096, ~1365 FLOP/byte → compute bound

This is why **kernel fusion** matters: fusing a matmul with its subsequent bias add and activation function doesn't change the FLOP count but reduces DRAM traffic, increasing AI toward the compute-bound regime.

### Profiling with Nsight Compute

```bash
# Profile a PyTorch script
ncu --metrics sm__warps_active.avg.pct_of_peak_sustained_active,\
dram__bytes_read.sum,dram__bytes_write.sum,\
sm__throughput.avg.pct_of_peak_sustained_elapsed \
python my_kernel.py
```

Key metrics to examine:
- `sm__warps_active.avg.pct_of_peak_sustained_active`: achieved occupancy
- `l1tex__t_bytes_pipe_lsu_mem_global_op_ld.sum`: L1 global load traffic
- `dram__bytes_read.sum`: actual DRAM reads (vs. L2 served)
- `smsp__warp_issue_stalled_long_scoreboard_per_warp_active.ratio`: fraction of cycles stalled waiting for a long-latency op (most commonly: global memory load)

---

## 7. Writing an Optimized CUDA Kernel in Python (Numba)

### Naive Matrix Multiply

```python
from numba import cuda, float32
import numpy as np
import time

@cuda.jit
def matmul_naive(A, B, C):
    """Each thread computes one element of C."""
    row, col = cuda.grid(2)
    if row < C.shape[0] and col < C.shape[1]:
        tmp = float32(0.0)
        for k in range(A.shape[1]):
            tmp += A[row, k] * B[k, col]
        C[row, col] = tmp
```

Every read of `A[row, k]` loads one float from global memory. For a 1024×1024 matrix multiply, thread (0,0) alone issues 1024 global memory reads. With 1024² threads, that's **1 billion** global memory transactions — with zero reuse. The same element `A[0, k]` is read independently by every thread in column 0.

### Tiled Matrix Multiply (Shared Memory)

The standard optimization loads **tiles** of A and B into shared memory, then each thread reads from the fast on-chip shared memory instead of DRAM:

```python
from numba import cuda, float32
import numpy as np
import time

TILE = 16  # Tile dimension — must match blockDim

@cuda.jit
def matmul_tiled(A, B, C):
    # Allocate shared memory tiles — statically sized, on-chip
    sA = cuda.shared.array(shape=(TILE, TILE), dtype=float32)
    sB = cuda.shared.array(shape=(TILE, TILE), dtype=float32)

    # Thread indices within this block
    tx = cuda.threadIdx.x  # column index within tile
    ty = cuda.threadIdx.y  # row index within tile

    # Global row and column this thread is responsible for
    row = cuda.blockIdx.y * TILE + ty
    col = cuda.blockIdx.x * TILE + tx

    tmp = float32(0.0)

    # Loop over tiles along the K dimension
    n_tiles = (A.shape[1] + TILE - 1) // TILE
    for tile_k in range(n_tiles):

        # Collaboratively load tile of A: thread (ty,tx) loads A[row, tile_k*TILE+tx]
        if row < A.shape[0] and (tile_k * TILE + tx) < A.shape[1]:
            sA[ty, tx] = A[row, tile_k * TILE + tx]
        else:
            sA[ty, tx] = float32(0.0)  # Pad with zeros for boundary tiles

        # Collaboratively load tile of B: thread (ty,tx) loads B[tile_k*TILE+ty, col]
        if (tile_k * TILE + ty) < B.shape[0] and col < B.shape[1]:
            sB[ty, tx] = B[tile_k * TILE + ty, col]
        else:
            sB[ty, tx] = float32(0.0)

        # CRITICAL: all threads in the block must finish loading before any thread
        # reads from shared memory. syncthreads() is a block-wide barrier.
        cuda.syncthreads()

        # Compute partial dot product for this tile
        # This reads from shared memory (~5ns) not DRAM (~300ns)
        for k in range(TILE):
            tmp += sA[ty, k] * sB[k, tx]

        # CRITICAL: all threads must finish computing before we overwrite sA, sB
        # in the next iteration. Second barrier.
        cuda.syncthreads()

    # Write result to global memory (one write per thread total)
    if row < C.shape[0] and col < C.shape[1]:
        C[row, col] = tmp


def benchmark_matmul_all(N=1024):
    A_host = np.random.randn(N, N).astype(np.float32)
    B_host = np.random.randn(N, N).astype(np.float32)

    # NumPy CPU baseline
    t0 = time.perf_counter()
    C_np = A_host @ B_host
    cpu_ms = (time.perf_counter() - t0) * 1000
    print(f"NumPy (CPU):       {cpu_ms:.1f} ms")

    A_dev = cuda.to_device(A_host)
    B_dev = cuda.to_device(B_host)
    C_dev = cuda.device_array((N, N), dtype=np.float32)

    threads_2d = (TILE, TILE)
    blocks_2d = ((N + TILE - 1) // TILE, (N + TILE - 1) // TILE)

    # Naive GPU kernel
    matmul_naive[blocks_2d, threads_2d](A_dev, B_dev, C_dev)
    cuda.synchronize()
    t0 = time.perf_counter()
    for _ in range(20):
        matmul_naive[blocks_2d, threads_2d](A_dev, B_dev, C_dev)
    cuda.synchronize()
    naive_ms = (time.perf_counter() - t0) / 20 * 1000
    print(f"Naive GPU kernel:  {naive_ms:.1f} ms")

    # Tiled GPU kernel
    matmul_tiled[blocks_2d, threads_2d](A_dev, B_dev, C_dev)
    cuda.synchronize()
    t0 = time.perf_counter()
    for _ in range(20):
        matmul_tiled[blocks_2d, threads_2d](A_dev, B_dev, C_dev)
    cuda.synchronize()
    tiled_ms = (time.perf_counter() - t0) / 20 * 1000
    print(f"Tiled GPU kernel:  {tiled_ms:.1f} ms")

    # CuPy (uses cuBLAS, Tensor Cores if available)
    import cupy as cp
    A_cp = cp.asarray(A_host)
    B_cp = cp.asarray(B_host)
    for _ in range(5): C_cp = A_cp @ B_cp
    cp.cuda.Stream.null.synchronize()
    t0 = time.perf_counter()
    for _ in range(100): C_cp = A_cp @ B_cp
    cp.cuda.Stream.null.synchronize()
    cupy_ms = (time.perf_counter() - t0) / 100 * 1000
    flops = 2 * N**3
    tflops = flops / (cupy_ms / 1000) / 1e12
    print(f"CuPy (cuBLAS):     {cupy_ms:.2f} ms  |  {tflops:.1f} TFLOP/s")

    print(f"\nSpeedup: Naive→Tiled = {naive_ms/tiled_ms:.1f}x, CPU→cuBLAS = {cpu_ms/cupy_ms:.0f}x")

benchmark_matmul_all(N=1024)
```

**Why the tiled kernel is faster:**

The tile loop reduces global memory traffic by a factor of `TILE`. For a TILE=16 kernel:
- Naive: Each element of A is read N/TILE times by the same thread across tiles. But globally, A is read by N columns of blocks → each element read N times.
- Tiled: Each tile loads TILE² elements of A and TILE² elements of B, then computes TILE³ multiply-adds. The memory traffic per FLOP is reduced by TILE.

The shared memory bandwidth (19 TB/s) vs DRAM bandwidth (2 TB/s) ratio on A100 is ~10x, so the tiling speedup is substantial — but cuBLAS still wins because it uses Tensor Cores, register-blocking, and vectorized loads.

---

## 8. GPU Memory Management in Practice

### Memory Pool: The Hidden Performance Tax

```python
import torch
import time

# Naive allocation — slow
t0 = time.perf_counter()
for i in range(1000):
    x = torch.randn(1024, 1024, device='cuda')
    del x
torch.cuda.synchronize()
print(f"Without pool: {(time.perf_counter()-t0)*1000:.0f} ms")
# ~500ms: each allocation/free hits the CUDA allocator

# PyTorch's caching allocator reuses freed memory
# The first 1000 iterations may pay allocation cost
# Subsequent ones are fast free-list lookups
t0 = time.perf_counter()
for i in range(1000):
    x = torch.randn(1024, 1024, device='cuda')  # Returns from cache
    del x  # Returns to cache, not freed
torch.cuda.synchronize()
print(f"With cache:   {(time.perf_counter()-t0)*1000:.0f} ms")
# ~50ms: ~10x faster for repeated same-size allocations

# Check cache state
print(torch.cuda.memory_stats()['reserved_bytes.all.current'] / 1e9, "GB reserved")
torch.cuda.empty_cache()  # Release to OS (slow, rarely needed)
```

### Unified Memory

```python
import cupy as cp

# cudaMallocManaged equivalent in Python (CuPy):
# CuPy's default allocator is device-only; unified memory is more complex
# In PyTorch, use:
x = torch.zeros(1024, 1024)  # CPU
x_gpu = x.cuda()              # Explicit copy (preferred for perf)

# Unified memory (automatic migration, page-fault driven):
# torch doesn't expose this directly; use CUDA Python (pycuda) if needed
import pycuda.driver as cuda_driver
# cuda_driver.mem_alloc_managed(size, cuda_driver.mem_attach_flags.GLOBAL)
# Not recommended for production; latency from page faults can be 10x worse
```

---

## 9. What Textbooks Don't Tell You

### 9.1 The Real Bottleneck is Almost Always Memory

Consider an elementwise operation on N float32 values:

```python
# y = relu(x)  — one FLOP per element (max with 0)
# Reads 4 bytes, writes 4 bytes = 8 bytes total DRAM traffic
# Arithmetic intensity = 1 FLOP / 8 bytes = 0.125 FLOP/byte
# A100 ridge point = 156 FLOP/byte
# This kernel can NEVER be compute-bound, no matter how fast your cores are.
# Peak performance = min(2TB/s × 0.125 FLOP/byte, 312 TFLOP/s) = 250 GFLOP/s
# A100's FP32 peak = 19.5 TFLOP/s → we use 1.3% of compute capability
```

This is why Flash Attention, memory-efficient attention, and fused kernels exist: they reduce DRAM traffic, which is the actual bottleneck.

### 9.2 The NVLink Topology Matters for Multi-GPU

An A100 has NVLink 3.0: 12 NVLink links × 50 GB/s bidirectional = **600 GB/s GPU-to-GPU bandwidth** in an NVSwitch topology (DGX A100). PCIe 4.0 delivers 64 GB/s. For all-reduce on 8 GPUs:

```python
import torch
import torch.distributed as dist

# All-reduce bandwidth test
dist.init_process_group('nccl')
data = torch.randn(1024*1024*256, device='cuda')  # 1 GB

# NVLink path: all-reduce over NVSwitch
# Bandwidth = 600 GB/s (write to NVSwitch) / 2 = 300 GB/s effective per direction
# For 8 GPUs, ring all-reduce: each GPU sends/receives 2*(N-1)/N ≈ 2GB per GPU
# Time ≈ 2GB / 300 GB/s ≈ 6.7ms for 1GB tensor

dist.all_reduce(data, op=dist.ReduceOp.SUM)
torch.cuda.synchronize()
```

For tensor parallelism with sequence_length=1024, head_dim=64, num_heads=16: the all-reduce after each attention output projection sends `batch × seq × model_dim = 1 × 1024 × 1024 = 4MB`. At 300 GB/s NVLink, that's **13 microseconds** — completely negligible vs. the attention compute time.

### 9.3 Quantization Changes the Economics

INT8 Tensor Cores run at **624 TOPS/s** on A100 — 2x FP16. INT4 at 1,248 TOPS/s. Equally important: a model in INT8 is half the size of FP16, so the memory bandwidth ceiling (2 TB/s) limits you at half the tokens/second for INT8 vs. FP16 for **memory-bandwidth-bound inference** (short context, large batch). For very long contexts (KV cache fills DRAM), INT8 quantization of KV cache reduces memory footprint directly.

Quantization-aware training (QAT) using `torch.quantization`:

```python
import torch
import torch.quantization

model = MyTransformer()

# Insert fake-quantization observers during training
model.qconfig = torch.quantization.get_default_qat_qconfig('fbgemm')
torch.quantization.prepare_qat(model, inplace=True)

# Train normally — fake quant nodes simulate INT8 rounding during forward pass
# ... training loop ...

# Convert to actual INT8 for inference
torch.quantization.convert(model, inplace=True)
# Now model.linear layers use INT8 GEMM (on CPU) or via TensorRT on GPU
```

### 9.4 The CUDA Context Tax

Every process that uses CUDA creates a **CUDA context** — the GPU state (page tables, streams, synchronization primitives) for that process. Switching between contexts requires draining the GPU pipeline. On a shared inference server running 8 model replicas as 8 processes, context switching overhead can consume 10–20% of GPU time.

**Multi-Process Service (MPS):** A daemon that multiplexes multiple CUDA processes through a single context, eliminating context switching overhead. Critical for multi-tenant inference:

```bash
# Enable MPS
nvidia-cuda-mps-control -d
# All subsequent CUDA processes share one context
```

---

## 10. Hard Exercise: Transformer Attention Optimization

### (a) Attention Arithmetic Intensity Analysis

For GPT-2: batch=32, heads=12, seq=512, head_dim=64.

The Q@K^T operation has dimensions `[batch, heads, seq, head_dim] × [batch, heads, head_dim, seq]` → output `[batch, heads, seq, seq]`.

```
FLOPs = batch × heads × seq × seq × head_dim × 2
      = 32 × 12 × 512 × 512 × 64 × 2
      = 12,884,901,888 ≈ 12.9 GFLOP

DRAM reads:  Q matrix = 32 × 12 × 512 × 64 × 2 bytes (FP16) = 25,165,824 bytes
             K matrix = same = 25,165,824 bytes
DRAM writes: S matrix = 32 × 12 × 512 × 512 × 2 bytes = 201,326,592 bytes

Total DRAM traffic = 25.2MB + 25.2MB + 201.3MB ≈ 251.7 MB

Arithmetic Intensity = 12.9 GFLOP / 0.252 GB ≈ 51.2 FLOP/byte
```

**Conclusion:** 51.2 FLOP/byte < 156 FLOP/byte ridge point → **memory-bandwidth bound** on A100. The large attention matrix (201 MB) dominates. This is what Flash Attention is designed to fix.

### (b) Flash Attention: Tiling for Reduced HBM Traffic

Naive attention:
1. Load Q, K from HBM → compute S = Q@K^T → write S to HBM
2. Load S → compute P = softmax(S) → write P to HBM
3. Load P, V → compute O = P@V → write O to HBM

Total HBM reads/writes: ~4 seq² per head (S read + P write + P read).

Flash Attention (Dao et al., 2022) uses **online softmax** to compute the softmax in tiles without ever materializing the full S matrix in HBM:

```
For each block of Q (rows):
    For each block of K, V (columns):
        Load Q_block, K_block, V_block into SRAM
        Compute S_block = Q_block @ K_block^T  (in SRAM)
        Update running max m_new = max(m_old, rowmax(S_block))
        Rescale accumulated output: O_old *= exp(m_old - m_new)
        P_block = exp(S_block - m_new)  (stable numerics)
        O_block += P_block @ V_block    (partial sum into SRAM)
        l_new = exp(m_old - m_new) * l_old + rowsum(P_block)
    O_final = O_block / l_final  (normalize by partition function)
    Write O_final to HBM
```

HBM traffic: Load Q, K, V once each; write O once. Total: 4 × seq × head_dim (vs. 4 × seq² + ...). For seq=512, head_dim=64: Flash Attention reduces HBM traffic by ~512/64 = **8x**, and correspondingly achieves 8x higher arithmetic intensity.

**Flash Attention arithmetic intensity:**
```
FLOPs same as naive ≈ 2 × batch × heads × seq² × head_dim
HBM traffic ≈ 4 × batch × heads × seq × head_dim × bytes_per_elem
AI = 2 × seq² × head_dim / (4 × seq × head_dim) = seq/2 = 256 FLOP/byte
```

256 FLOP/byte > 156 ridge point → **compute bound**. Flash Attention turns a memory-bound operation into a compute-bound one — this is its fundamental contribution.

### (c) Diagnosing "Warp Stall: Long Scoreboard"

"Long Scoreboard" stalls mean a thread is waiting for a result that has not yet been produced by a long-latency instruction — almost always a **global memory load** (L2 miss or DRAM access).

```
Root cause: The warp issued a global memory load instruction.
The load goes to L1 → misses → goes to L2 → misses (or hits) → goes to DRAM.
The load latency is 200–400 cycles.
The warp cannot execute its next instruction (which needs that loaded value)
until the data arrives.
The scoreboard tracks this dependency.
```

60% of cycles spent stalling on Long Scoreboard means the SM issues a useful instruction only 40% of cycles. Causes and fixes:

| Root Cause | Fix |
|---|---|
| Insufficient warps to hide latency | Reduce register/smem usage → higher occupancy |
| Uncoalesced access (low L1 hit rate) | Restructure data layout (SoA), align allocations |
| L2 thrashing (working set > 40MB) | Tile computations to fit in L2 |
| Sequential dependency chains | Software pipelining: prefetch next tile while computing current |

### (d) Flash Attention Benchmark in PyTorch

```python
import torch
import torch.nn.functional as F
import time
import math

def benchmark_attention(batch=32, heads=12, seq=512, head_dim=64, dtype=torch.float16):
    device = 'cuda'
    scale = math.sqrt(head_dim)

    Q = torch.randn(batch, heads, seq, head_dim, device=device, dtype=dtype)
    K = torch.randn(batch, heads, seq, head_dim, device=device, dtype=dtype)
    V = torch.randn(batch, heads, seq, head_dim, device=device, dtype=dtype)

    def run_naive(Q, K, V):
        # 3 separate HBM round-trips: QK^T write, softmax, PV
        S = (Q @ K.transpose(-2, -1)) / scale
        P = F.softmax(S, dim=-1)
        return P @ V

    def run_flash(Q, K, V):
        # PyTorch >= 2.0 uses Flash Attention kernel automatically
        # when inputs are on CUDA and no attention_mask is given
        return F.scaled_dot_product_attention(Q, K, V, scale=1.0/scale)

    # Warmup
    for _ in range(10):
        _ = run_naive(Q, K, V)
        _ = run_flash(Q, K, V)
    torch.cuda.synchronize()

    # Profile naive
    start = torch.cuda.Event(enable_timing=True)
    end = torch.cuda.Event(enable_timing=True)

    start.record()
    for _ in range(100):
        out_naive = run_naive(Q, K, V)
    end.record()
    torch.cuda.synchronize()
    naive_ms = start.elapsed_time(end) / 100

    # Profile Flash
    start.record()
    for _ in range(100):
        out_flash = run_flash(Q, K, V)
    end.record()
    torch.cuda.synchronize()
    flash_ms = start.elapsed_time(end) / 100

    # HBM bandwidth estimate
    elem_bytes = 2 if dtype == torch.float16 else 4
    # Naive: reads Q+K (2 × B×H×S×D), writes S (B×H×S×S), reads S, writes P, reads P+V, writes O
    naive_hbm_gb = (
        2 * batch * heads * seq * head_dim * elem_bytes  # Q, K read
        + batch * heads * seq * seq * elem_bytes          # S write
        + batch * heads * seq * seq * elem_bytes          # S read (softmax)
        + batch * heads * seq * seq * elem_bytes          # P write
        + batch * heads * seq * seq * elem_bytes          # P read (PV)
        + batch * heads * seq * head_dim * elem_bytes     # V read
        + batch * heads * seq * head_dim * elem_bytes     # O write
    ) / 1e9

    # Flash: reads Q, K, V once; writes O once
    flash_hbm_gb = (
        3 * batch * heads * seq * head_dim * elem_bytes  # Q, K, V read
        + batch * heads * seq * head_dim * elem_bytes    # O write
    ) / 1e9

    naive_bw = naive_hbm_gb / (naive_ms / 1000)  # GB/s
    flash_bw = flash_hbm_gb / (flash_ms / 1000)

    print(f"Naive attention:  {naive_ms:.2f} ms | {naive_hbm_gb*1000:.0f} MB HBM | {naive_bw:.0f} GB/s effective")
    print(f"Flash attention:  {flash_ms:.2f} ms | {flash_hbm_gb*1000:.0f} MB HBM | {flash_bw:.0f} GB/s effective")
    print(f"Speedup: {naive_ms/flash_ms:.1f}x  |  HBM traffic reduction: {naive_hbm_gb/flash_hbm_gb:.1f}x")
    print(f"A100 peak BW: 2000 GB/s — Flash achieves {flash_bw/2000*100:.0f}% of peak\n")

benchmark_attention()
```

### (e) Tensor Parallelism for Batch=1 Online Inference

For batch=1 inference, the bottleneck is **loading model weights from HBM** (not compute). A 117M parameter model in FP16 = 234 MB. Each forward pass must stream those weights through the GPU:

```
Single A100: Time = 234 MB / 2000 GB/s = 0.117 ms per forward pass
→ Throughput = 1 / 0.117ms ≈ 8,500 tokens/sec (theoretical peak)
```

With 8 A100s in tensor parallelism (Megatron-style, column/row parallel linear layers):
```
Each GPU holds 1/8 of each weight matrix
HBM traffic per GPU = 234 MB / 8 = 29.25 MB
Time = 29.25 MB / 2000 GB/s = 14.6 μs compute

Plus all-reduce: 234 MB × 2(send+receive) / 8 GPUs = 58.5 MB per all-reduce
NVLink time = 58.5 MB / 300 GB/s ≈ 0.2 ms

Total time per layer ≈ 0.015 ms + 0.2 ms = 0.215 ms → SLOWER for small models
```

**Wait — why does tensor parallelism help?** For batch=1, it helps only when:
1. The model is large enough that single-GPU HBM bandwidth is the bottleneck (true for 70B+ models in FP16: 140 GB >> fits in one A100's 80 GB).
2. The communication-to-compute ratio is favorable (longer sequence lengths increase compute relative to communication).

**Amdahl's Law:** If fraction `f` of work is parallelizable and `1-f` is serial:
```
Speedup ≤ 1 / ((1-f) + f/N)
```

For tensor parallelism, the all-reduce is the "serial" portion. If all-reduce takes 0.2ms and compute takes 0.015ms per layer:
```
f ≈ 0.015 / (0.015 + 0.2) = 6.5% (compute fraction)
Serial fraction = 93.5% (communication)
Max speedup = 1 / 0.935 = 1.07x — barely useful!
```

For a 70B model with same 8-GPU split:
```
HBM per GPU = 140GB/8 = 17.5 GB → bandwidth limited
Compute per GPU = 14.6μs × (70B/117M) = 8.7 ms
All-reduce = 0.2 ms per layer
f ≈ 8.7 / (8.7 + 0.2) = 97.8%
Speedup = 1/(0.022 + 0.978/8) = 1/0.1443 ≈ 6.9x — near linear
```

**The lesson:** Tensor parallelism is profitable for batch=1 *only* for large models where the per-GPU compute time substantially exceeds the all-reduce latency. For smaller models (<30B at FP16), pipeline parallelism (layer splitting, less communication) or simply running on fewer GPUs is often better.

---

## Summary: The Mental Model for GPU Performance

```
1. MEMORY first: almost all ML kernels are memory-bandwidth-bound.
   Measure arithmetic intensity. If AI < ridge point, focus on:
   → Kernel fusion (reduce DRAM round-trips)
   → Better access patterns (coalescing, tiling, SoA)
   → Quantization (reduce bytes per element)

2. COMPUTE second: if AI > ridge point, focus on:
   → Tensor Core utilization (align to 16×, use FP16/BF16/INT8)
   → Occupancy tuning (reduce register pressure, shared memory usage)
   → Warp efficiency (eliminate divergence)

3. COMMUNICATION third (multi-GPU): minimize all-reduce frequency and volume.
   Tensor parallelism: suited for very large models, fast interconnects.
   Pipeline parallelism: suited for large batch, tolerant of pipeline bubbles.
   Data parallelism: suited for all models, communication is gradient sync (infrequent).

4. PROFILE: do not guess. Use Nsight Compute. Look at:
   → dram__bytes.sum (actual DRAM traffic vs. theoretical minimum)
   → sm__throughput (compute utilization)
   → Long Scoreboard stalls (memory latency not hidden)
   → L1/L2 hit rates (cache effectiveness)
```

---

## References

1. NVIDIA. *NVIDIA A100 Tensor Core GPU Architecture Whitepaper* (2020). https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/nvidia-ampere-architecture-whitepaper.pdf
2. Dao, T., Fu, D., Ermon, S., Rudra, A., & Ré, C. *FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness.* NeurIPS 2022. https://arxiv.org/abs/2205.14135
3. NVIDIA. *CUDA C++ Programming Guide, v12.x.* https://docs.nvidia.com/cuda/cuda-c-programming-guide/
4. Williams, S., Waterman, A., Patterson, D. *Roofline: An Insightful Visual Performance Model for Floating-Point Programs and Multicore Architectures.* CACM 2009.
5. NVIDIA. *Nsight Compute CLI Documentation.* https://docs.nvidia.com/nsight-compute/NsightComputeCli/
6. Shoeybi, M. et al. *Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism.* 2019. https://arxiv.org/abs/1909.08053

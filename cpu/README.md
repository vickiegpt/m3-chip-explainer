# M3 CPU Architecture

Apple's M3 CPU continues the hybrid P-core/E-core design, with improvements in IPC and efficiency. This document covers microarchitectural details derived from empirical measurements.

## Core Configuration

| Chip | P-cores | E-cores | Total | L2 (Shared) |
|------|---------|---------|-------|-------------|
| M3 | 4 | 4 | 8 | 16 MB |
| M3 Pro | 6 | 6 | 12 | 24 MB |
| M3 Max | 12 | 4 | 16 | 48 MB |
| M3 Ultra | 24 | 8 | 32 | 96 MB |

## P-Core (Performance Core) Architecture

### Frontend

| Component | Specification |
|-----------|---------------|
| Decode Width | 8 instructions/cycle |
| Fetch Width | 32 bytes/cycle |
| Branch Predictor | TAGE-like, 4K+ entry BTB |
| L1 I-Cache | 192 KB, 6-way |
| I-TLB | 192 entries |

#### Breaking the Fetch Bottleneck

The M3 targets an average of 10+ instructions fetched per cycle. To achieve this, Apple has moved beyond standard branch prediction to handle "boring" code more efficiently.

#### Biased & Stable Branches

A significant portion of code consists of error checks or loops that rarely change direction.

- **Biased Branches**: Branches that are almost always taken (or not taken)
- **Optimization**: Strongly biased branches are treated as effective NOPs (if not taken) or GOTOs (if taken) early in the Fetch stage
- **Predictor Efficiency**: These branches are handled by the fast "Next Fetch Predictor" and do not pollute the expensive, power-hungry TAGE predictor tables

#### Trace Cache (Reinvented)

Unlike the Pentium 4 trace cache, Apple's implementation constructs "traces" of instructions that skip over stable/biased branches:

- Allows fetching a sequence of instructions spanning multiple cache lines in a single cycle
- Effectively increases fetch bandwidth for code with frequent short jumps
- Traces are dynamically constructed based on branch behavior patterns

#### Return Prediction

**Return Fetch Stack**: A mechanism to avoid the 1-cycle bubble usually associated with RET instructions:

- Stores either the instructions immediately following a CALL or pointers to the Fetch Predictor entry
- Allows the CPU to resume fetching immediately upon return without waiting for address calculation
- Critical for deep call stacks common in modern software

#### Signature-based Instruction Prefetch

Apple uses a TAGE-like history of function call/return paths to predict long-distance jumps:

- Predicts function calls well in advance
- Allows I-Cache to prefetch target function before the CPU gets there
- Particularly effective for indirect calls with predictable patterns

### Execution Engine

| Component | Specification |
|-----------|---------------|
| Issue Width | 8 micro-ops/cycle |
| ROB Size | ~680 entries (estimated) |
| Integer ALUs | 6 |
| FP/SIMD Units | 4 (128-bit NEON) |
| Load/Store Units | 3 Load + 2 Store |
| Branch Units | 2 |

### Middle-End: Decode and Rename Optimizations

Between Fetch and Execution, Apple optimizes resource allocation through several mechanisms.

#### Load-Op Fusion

A major throughput unlock that fuses a Load instruction with a subsequent ALU operation (e.g., ADD, XOR, CMP):

```
// Before fusion (2 separate ops):
LDR X1, [X0]      // Load from memory
ADD X2, X1, X3    // Add loaded value

// After fusion (single internal op):
// Load+ADD executed as one operation
```

**Mechanism**:
- The load and ALU op are fused into a single internal operation
- The Load/Store Unit (LSU) gains a small ALU to perform the operation immediately upon data arrival
- Mimics x86 memory-operand instructions but retains RISC encoding

**Benefits**:
- Reduces register pressure (no temporary register needed for the load)
- Reduces latency by combining operations
- Higher effective throughput

#### Virtualized Resource Allocation (Lazy Allocation)

Traditionally, physical registers are allocated at Rename. M3 moves toward allocating only "ordering tags" at Rename:

- Actual heavy physical register storage allocated only when the instruction executes
- Effectively increases the size of the Reorder Buffer (ROB) without exploding register file size
- Reduces power consumption for instructions that may be squashed

#### Loop Stream Detector (LSD)

Present in hardware to save power on tight loops:

- Detects small loops that fit in a loop buffer
- Bypasses fetch and decode for subsequent iterations
- Saves significant power for hot loops

**Note**: This was famously disabled on Intel Skylake and potentially early Apple designs due to complex bugs. Newer implementations include fixes for edge cases involving hyperthreading-like scenarios.

### Backend

| Component | Specification |
|-----------|---------------|
| L1 D-Cache | 128 KB, 8-way |
| D-TLB | 160 entries |
| Load Latency | 3 cycles (L1 hit) |
| Store Buffer | 108 entries |
| Load Queue | 130 entries |

#### Hierarchical Store Queues

The M3 implements a two-tier store queue system for handling latency and memory barriers more intelligently:

**Fast Store Queue**:
- Small, fast queue handles majority of store-to-load forwarding
- Critical path for performance-sensitive code
- Low latency for common cases

**Slow Store Queue**:
- Larger, slower secondary queue handles older stores or less critical traffic
- Prevents machine from stalling when the fast store buffer fills up
- Allows graceful degradation under heavy store pressure

#### Optimized Barriers (ISB/DSB)

Traditional implementations of `ISB` (Instruction Synchronization Barrier) and `TLBI` (TLB Invalidate) force a full pipeline flush. M3 implements "poisoning" optimization:

**Traditional Approach**:
```
ISB → Full pipeline flush (expensive)
```

**M3 Approach**:
```
ISB → Mark in-flight instructions as "poisoned"
    → Instructions flow through normally
    → Only flush if poisoned instruction touches stale context
```

**Benefits**:
- ISB/DSB operations become much cheaper in the common case
- Only pay the flush cost when actually necessary
- Significant speedup for code with frequent barriers (e.g., JIT compilers, OS kernels)

#### LSU-Based Prefetcher Training

Unlike ARM reference designs that train prefetchers based on L1 cache misses, Apple trains based on the Load/Store Unit (LSU):

- Prefetcher sees the content of data loaded (e.g., pointer values)
- Enables **Content-Directed Prefetching**: chasing pointers in linked lists/graphs
- More intelligent prefetch decisions based on actual data patterns

## E-Core (Efficiency Core) Architecture

| Component | Specification |
|-----------|---------------|
| Decode Width | 4 instructions/cycle |
| Issue Width | 4 micro-ops/cycle |
| ROB Size | ~96 entries |
| L1 I-Cache | 64 KB |
| L1 D-Cache | 64 KB |
| L2 Cache | 4 MB shared (E-cluster) |

## Cache Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                         System Level                         │
├─────────────────────────────────────────────────────────────┤
│                    Unified Memory (LPDDR5)                   │
│                    100-400 GB/s bandwidth                    │
├─────────────────────────────────────────────────────────────┤
│                     System Level Cache (SLC)                 │
│                    ~32 MB (M3 Max example)                   │
├───────────────────────┬─────────────────────────────────────┤
│    P-Core Cluster     │           E-Core Cluster             │
│    L2: 16-48 MB       │           L2: 4 MB                   │
├───────────────────────┼─────────────────────────────────────┤
│ P-Core 0   P-Core 1   │    E-Core 0   E-Core 1              │
│ L1I: 192K  L1I: 192K  │    L1I: 64K   L1I: 64K              │
│ L1D: 128K  L1D: 128K  │    L1D: 64K   L1D: 64K              │
└───────────────────────┴─────────────────────────────────────┘
```

### Cache Line Size

All caches use 128-byte cache lines (not 64 bytes like x86):

```c
// Cache line size detection
#include <stdint.h>
size_t get_cache_line_size() {
    uint64_t ctr;
    asm volatile("mrs %0, ctr_el0" : "=r"(ctr));
    return 4 << ((ctr >> 16) & 0xF);  // Returns 128
}
```

### Memory Latency

Measured latencies (P-core, random access):

| Level | Latency |
|-------|---------|
| L1D | 3 cycles |
| L2 | 15-17 cycles |
| SLC | ~50 cycles |
| DRAM | ~120-150 cycles |

## Hardware Prefetchers

M3 implements multiple prefetcher types:

### 1. [1D Array Prefetcher](1d-array-prefetcher.md)
Sequential stride detection for linear array traversal.

### 2. [Multiple Dereference Prefetcher](multiple-deref-prefetcher.md)
Pointer-chasing prefetcher for linked data structures.

### 3. Spatial Prefetcher
Adjacent cache line prefetching within a page.

### 4. Temporal Prefetcher
History-based prefetching for recurring access patterns.

## Memory Ordering

Apple Silicon implements ARM's relaxed memory model with:

- **Load-Load**: Can be reordered
- **Load-Store**: Can be reordered
- **Store-Store**: Ordered (write buffer drains in order)
- **Store-Load**: Can be reordered

### Barrier Instructions

```asm
DMB ISH     // Data Memory Barrier, Inner Shareable
DMB OSH     // Data Memory Barrier, Outer Shareable
DSB SY      // Data Synchronization Barrier
ISB         // Instruction Synchronization Barrier
```

### Load-Acquire / Store-Release

```asm
LDAR  x0, [x1]     // Load-acquire
STLR  x0, [x1]     // Store-release
LDAPR x0, [x1]     // Load-acquire RCpc (weaker)
```

## Measuring with Performance Counters

AArch64-Explore provides infrastructure for PMU access:

```c++
#include <sys/sysctl.h>

// Force execution on P-core
void pin_to_pcore() {
    pthread_set_qos_class_self_np(QOS_CLASS_USER_INTERACTIVE, 0);
}

// Read cycle counter (requires elevated privileges on macOS)
static inline uint64_t read_cycles() {
    uint64_t val;
    asm volatile("mrs %0, cntvct_el0" : "=r"(val));
    return val;
}
```

### Key Performance Events

| Event | Description |
|-------|-------------|
| `INST_RETIRED` | Instructions retired |
| `CPU_CYCLES` | Core clock cycles |
| `L1D_CACHE_MISS` | L1 data cache misses |
| `L2_CACHE_MISS` | L2 cache misses |
| `BRANCH_MISPREDICT` | Branch mispredictions |
| `DISPATCH_STALL` | Dispatch stalls |

## Instruction Latencies (P-Core)

Selected instruction timings:

| Instruction | Latency | Throughput |
|-------------|---------|------------|
| ADD/SUB (reg) | 1 | 6/cycle |
| MUL (32-bit) | 3 | 2/cycle |
| MUL (64-bit) | 3 | 1/cycle |
| DIV (32-bit) | 7-10 | Variable |
| DIV (64-bit) | 10-14 | Variable |
| FADD (scalar) | 3 | 4/cycle |
| FMUL (scalar) | 4 | 4/cycle |
| FMADD | 4 | 4/cycle |
| Load (L1 hit) | 3 | 3/cycle |
| Store | 1 | 2/cycle |

## SIMD/NEON Capabilities

128-bit NEON with full support:

```asm
// Vector add example
ADD v0.4s, v1.4s, v2.4s    // 4x32-bit integer add
FADD v0.4s, v1.4s, v2.4s   // 4x32-bit float add
FMLA v0.4s, v1.4s, v2.4s   // Fused multiply-add
```

### SVE/SVE2 Status

M3 does **not** implement SVE or SVE2. Apple uses fixed 128-bit NEON vectors.

## References

- [AArch64-Explore Volume 1](https://github.com/name99-org/AArch64-Explore) - CPU Core Analysis
- [AArch64-Explore Volume 2](https://github.com/name99-org/AArch64-Explore) - Load/Store and Cache
- [Dougall Johnson Instruction Timings](https://dougallj.github.io/applecpu/firestorm.html)

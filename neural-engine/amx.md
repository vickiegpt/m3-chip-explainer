# AMX (Apple Matrix Extensions)

AMX is Apple's SIMD extension for matrix operations, accessible from CPU cores. Unlike the Neural Engine (ANE), AMX is tightly coupled with the CPU and shares the same memory space.

## Overview

AMX is a coprocessor attached to each CPU cluster:

```
┌─────────────────────────────────────────────┐
│              P-Core Cluster                  │
├─────────────────────────────────────────────┤
│  ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │ P-Core 0│ │ P-Core 1│ │ P-Core N│       │
│  └────┬────┘ └────┬────┘ └────┬────┘       │
│       │           │           │             │
│       └───────────┼───────────┘             │
│                   │                         │
│           ┌───────▼───────┐                 │
│           │  AMX Block    │                 │
│           │  (Shared)     │                 │
│           └───────────────┘                 │
└─────────────────────────────────────────────┘
```

## AMX vs ANE

| Aspect | AMX | ANE |
|--------|-----|-----|
| Location | CPU cluster | Separate block |
| Memory | CPU virtual memory | Dedicated SRAM |
| Precision | FP16/FP32/FP64 | INT4/INT8/FP16 |
| Latency | Low (same core) | Higher (DMA) |
| Throughput | Medium | High |
| Flexibility | High | Model-limited |
| Programmability | Direct instructions | Compiler only |

## Register File

AMX has its own register file, separate from NEON:

| Register Type | Count | Size |
|---------------|-------|------|
| X registers | 8 | 512 bits |
| Y registers | 8 | 512 bits |
| Z registers | 8 | 512 bits |

Registers can hold:
- 32× FP16 values
- 16× FP32 values
- 8× FP64 values
- 64× INT8 values

## Instruction Classes

### Load/Store

```
AMX LDX  [addr]       // Load into X register
AMX LDY  [addr]       // Load into Y register
AMX STZ  [addr]       // Store from Z register
```

### Matrix Operations

```
AMX FMA32             // FP32 fused multiply-add
AMX FMA16             // FP16 fused multiply-add
AMX FMA64             // FP64 fused multiply-add
AMX MAC16             // INT16 multiply-accumulate
```

### Vector Operations

```
AMX VECFP             // Vector floating-point
AMX VECINT            // Vector integer
AMX EXTR              // Extract and rearrange
```

## Matrix Multiply

The core AMX operation: outer product accumulation

### Outer Product Model

```
Z += X ⊗ Y^T

Where:
  X = 1×64 row vector (64 × FP16)
  Y = 1×64 row vector (64 × FP16)
  Z = 64×64 matrix (accumulated FP32)
```

### GEMM Implementation

To compute C = A × B:

```c
// Pseudocode for GEMM using AMX outer products
for each 64×64 tile of C:
    clear Z registers
    for k = 0 to K step 1:
        load A[tile_i, k] into X    // 64 elements
        load B[k, tile_j] into Y    // 64 elements
        AMX FMA16                    // Z += X ⊗ Y^T
    store Z to C[tile_i, tile_j]
```

### Performance

| Operation | Throughput (ops/cycle) |
|-----------|------------------------|
| FP16 FMA | 2048 |
| FP32 FMA | 1024 |
| FP64 FMA | 512 |
| INT8 MAC | 4096 |

## Programming AMX

### Direct Access (Undocumented)

AMX is accessed via special system registers:

```c
// WARNING: Undocumented, may change
static inline void amx_set() {
    __asm__ volatile("mov x0, #0x201\n"
                     ".word 0x00201020");  // AMX enable
}

static inline void amx_clr() {
    __asm__ volatile("mov x0, #0x201\n"
                     ".word 0x00201021");  // AMX disable
}
```

### Via Accelerate Framework (Recommended)

Apple provides AMX access through Accelerate:

```c
#include <Accelerate/Accelerate.h>

// Matrix multiply using vDSP (uses AMX internally)
vDSP_mmul(A, 1, B, 1, C, 1, M, N, K);

// BLAS GEMM (uses AMX for large matrices)
cblas_sgemm(CblasRowMajor, CblasNoTrans, CblasNoTrans,
            M, N, K, 1.0f, A, K, B, N, 0.0f, C, N);
```

### Via BNNS

```c
#include <Accelerate/BNNSDefines.h>

// Create matrix multiply layer
BNNSFilterParameters params = {
    .i_desc = input_desc,
    .o_desc = output_desc,
    .weights = weight_desc,
};

BNNSFilter filter = BNNSFilterCreateLayerFullyConnected(&params, NULL);
BNNSFilterApply(filter, input, output);
```

## Memory Access Patterns

### Optimal Access

AMX performs best with:
- Contiguous memory access
- 64-byte aligned data
- Large matrices (>64×64)

```c
// Good: aligned, contiguous
float* A = aligned_alloc(64, M * K * sizeof(float));

// Bad: strided access
float* B = malloc((M * K + padding) * sizeof(float));
```

### Cache Considerations

- AMX loads bypass L1 cache
- Direct path to L2/SLC
- Reduces cache pollution for large matrices

## Use Cases

### Machine Learning Inference

- Transformer attention heads
- Convolutional layers
- Fully connected layers

### Scientific Computing

- Linear algebra (BLAS Level 3)
- Signal processing (FFT via matrix)
- Physics simulations

### Image Processing

- Convolution kernels
- Color space conversion
- Image resizing

## Debugging

### Performance Counters

Check AMX utilization via Instruments:
- AMX cycles active
- AMX stalls
- Memory bandwidth to AMX

### Common Pitfalls

1. **Matrix too small**: Overhead dominates for matrices < 64×64
2. **Misaligned data**: Causes extra memory transactions
3. **Memory bound**: AMX can outrun memory bandwidth
4. **Contention**: Multiple cores sharing AMX block

## M3 Improvements

| Aspect | M1/M2 | M3 |
|--------|-------|-----|
| FP16 throughput | 1x | 1.5x |
| INT8 support | Limited | Full |
| Register file | 8 × 512b | 8 × 512b |
| Memory bandwidth | Good | Better |

## Comparison with x86 AMX

| Feature | Apple AMX | Intel AMX |
|---------|-----------|-----------|
| Tile size | 64×64 | 16×16 |
| Data types | FP16/32/64, INT8/16 | BF16, INT8 |
| Accumulator | Implicit (Z regs) | Explicit tiles |
| Access | Undocumented | ISA extension |
| Power | Integrated | High TDP |

## References

- [Dougall Johnson's AMX Exploration](https://github.com/dougallj/applegpu)
- [Accelerate Framework Documentation](https://developer.apple.com/documentation/accelerate)
- [corsix/amx](https://github.com/corsix/amx) - AMX reverse engineering

# M3 GPU Architecture (AGX G15)

The M3 GPU represents a major architectural leap, introducing hardware ray tracing, mesh shaders, and dynamic caching. This document covers the microarchitectural details derived from Asahi Linux reverse engineering and Metal API analysis.

## Overview

| Feature | M3 (AGX G15) | vs M2 (AGX G14) |
|---------|--------------|-----------------|
| GPU Cores | 10 | +25% |
| Peak FP32 | 4.1 TFLOPS | +20% |
| Ray Tracing | Hardware accelerated | New |
| Mesh Shaders | Native support | New |
| Dynamic Caching | Yes | New |
| Register File | Virtualized | New |

## Core Architecture

### GPU Core Structure

Each GPU core contains:

```
┌─────────────────────────────────────────────────────┐
│                    GPU Core                          │
├─────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐                 │
│  │ Shader Unit 0│  │ Shader Unit 1│  ... (x8-16)   │
│  │  128 ALUs    │  │  128 ALUs    │                 │
│  └──────────────┘  └──────────────┘                 │
├─────────────────────────────────────────────────────┤
│  ┌────────────┐  ┌────────────┐  ┌────────────┐    │
│  │ Texture    │  │  L1 Cache  │  │ Ray Trace  │    │
│  │ Unit       │  │  (32KB)    │  │ Unit       │    │
│  └────────────┘  └────────────┘  └────────────┘    │
└─────────────────────────────────────────────────────┘
```

### Execution Model

- **SIMT Width**: 32 threads (wavefront/warp)
- **ALUs per Core**: 128 (4 SIMDs × 32 lanes)
- **Threads per Core**: Up to 1024 concurrent
- **Occupancy**: Limited by registers, shared memory, or thread count

## Dynamic Caching (Register Virtualization)

### The Problem with Static Allocation

Traditional GPUs (including M1/M2):
```
Shader declares: 64 registers per thread
GPU allocates:   64 × max_threads registers
Result:          Low occupancy if many registers declared
```

### M3's Solution: Virtual Register File

M3 virtualizes GPU registers into a memory-backed hierarchy:

```
┌─────────────────────────────────────┐
│     Virtual Register Address Space   │
├─────────────────────────────────────┤
         │
         ▼
┌─────────────┐  Miss  ┌─────────────┐  Miss  ┌─────────────┐
│ Register    │───────▶│ L1 Cache    │───────▶│ L2/SLC      │
│ File (Fast) │        │ (32KB/core) │        │             │
└─────────────┘        └─────────────┘        └─────────────┘
     Hit: 0 cycles        Hit: ~4 cycles       Hit: ~20 cycles
```

### Benefits

1. **Higher Occupancy**: Shader can declare many registers; only active ones consume register file space
2. **Flexible Allocation**: Registers dynamically migrate between RF and cache based on usage
3. **Better Utilization**: No wasted register file entries for inactive threads

### Implementation Details

- Registers mapped to virtual addresses (similar to CPU virtual memory)
- Small, fast "register file" acts as L0 cache for hottest registers
- Spilled registers flow to L1 → L2 → SLC → DRAM
- Hardware tracks "liveness" to avoid unnecessary spills

## Kicks: GPU Work Virtualization

### Traditional Model

GPU work = "Command buffer" submitted atomically, runs to completion

### M3 Model: Kicks

Work is broken into "kicks" that can be:
- **Suspended**: Pause execution mid-work
- **Resumed**: Continue from suspension point
- **Migrated**: Move between GPU cores

```
┌─────────────────────────────────────────────────────────────┐
│                     Kick Lifecycle                           │
├─────────────────────────────────────────────────────────────┤
│  Created → Queued → Running → [Suspended] → Resumed → Done  │
└─────────────────────────────────────────────────────────────┘
```

### Benefits

1. **Preemption**: High-priority work can interrupt long-running shaders
2. **Load Balancing**: Work can migrate between cores
3. **Power Management**: Suspend work during thermal throttling
4. **Context Switching**: Multiple applications share GPU smoothly

## Hardware Ray Tracing

### Architecture

M3 includes dedicated ray tracing hardware:

```
┌─────────────────────────────────────────────────┐
│              Ray Tracing Unit                    │
├─────────────────────────────────────────────────┤
│  ┌────────────┐  ┌────────────┐  ┌───────────┐ │
│  │ BVH        │  │ Ray-Box    │  │ Ray-Tri   │ │
│  │ Traversal  │  │ Intersect  │  │ Intersect │ │
│  └────────────┘  └────────────┘  └───────────┘ │
│                                                  │
│  ┌────────────────────────────────────────────┐ │
│  │         Transform Unit (Instancing)         │ │
│  └────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### Components

**BVH Traversal**:
- Hardware traversal of Bounding Volume Hierarchies
- Supports both top-level (TLAS) and bottom-level (BLAS) acceleration structures

**Ray-Box Intersection**:
- Tests ray against axis-aligned bounding boxes
- Multiple boxes tested in parallel

**Ray-Triangle Intersection**:
- Final geometry intersection test
- Returns hit distance, barycentric coordinates

### Transform Unit (Instancing Optimization)

A key M3 innovation for handling instanced geometry:

**Traditional approach** (NVIDIA, AMD):
```
For each instance:
  1. Load instance transform matrix
  2. Transform ray from world to object space
  3. Traverse object-space BVH
  4. Transform hit back to world space
```

**M3 approach**:
```
Transform Unit handles instancing internally:
  - Rays transformed inside RT unit
  - No shader involvement for instancing
  - Enables efficient "forest of trees" scenes
```

### Metal Ray Tracing API

```metal
// M3 ray tracing example
intersection_function<triangle_data>
rayIntersection(ray r,
                acceleration_structure accel,
                intersection_params params) {
    // Hardware-accelerated intersection query
    return accel.intersect(r, params);
}
```

## Neural Accelerators in GPU

### Integer Matrix Multiply Unit

M3 integrates integer matrix operations into the GPU datapath:

```
Unlike NVIDIA Tensor Cores (autonomous, complex units)
Apple integrates block-scaled integer math into execution pipelines
```

**Supported Operations**:
- INT8 × INT8 → INT32 accumulate
- INT4 × INT4 → INT32 accumulate
- Block-scaled quantized operations

### FP16 Matrix Throughput

M3 increases FP16 matrix math throughput with mixed-precision support:

```metal
// Metal example using simdgroup matrix operations
simdgroup_matrix<float, 8, 8> C;
simdgroup_matrix<half, 8, 8> A, B;

// Mixed precision: half * half + float accumulate
simdgroup_multiply_accumulate(C, A, B, C);
```

| Operation | M2 Throughput | M3 Throughput |
|-----------|---------------|---------------|
| FP32 SIMD | 1.0x | 1.0x |
| FP16 SIMD | 2.0x | 2.0x |
| FP16 Matrix | 2.0x | ~3.0x |
| INT8 Matrix | N/A | 4.0x |

### Future: A17/M4 Generation

Expected improvements:
- Triple FP16 matrix math throughput
- Mixed precision FP16 × FP16 + FP32 accumulation pipeline
- Larger matrix tile sizes

## Mesh Shaders

### Traditional Pipeline

```
Vertex Shader → Hull → Tessellator → Domain → Geometry → Rasterizer
    │                                           │
    └─── Per-vertex processing ─────────────────┘
```

### Mesh Shader Pipeline

```
Object Shader → Mesh Shader → Rasterizer
       │              │
       └── Flexible geometry generation ──┘
```

**Benefits**:
- Programmable culling before vertex processing
- Meshlet-based rendering
- Reduced memory bandwidth

### Metal Mesh Shader API

```metal
[[mesh]]
void meshMain(mesh_grid_properties props,
              object_data ObjectPayload* payload,
              meshThreadgroups threadgroups) {
    // Generate geometry procedurally
    SetMeshOutputCounts(vertex_count, prim_count);

    // Output vertices and primitives
    SetMeshOutput(vertex_index, position);
    SetMeshPrimitive(prim_index, indices);
}
```

## Memory Hierarchy

### Per-Core Caches

| Cache | Size | Latency | Purpose |
|-------|------|---------|---------|
| Register File | ~64KB effective | 0 cycles | Thread registers |
| L1 Data | 32KB | ~4 cycles | Thread data |
| L1 Texture | 64KB | ~8 cycles | Texture samples |
| Shared Memory | 32KB | ~4 cycles | Threadgroup data |

### Shared Caches

| Cache | Size (M3 Max) | Latency |
|-------|---------------|---------|
| L2 Cache | 8MB | ~40 cycles |
| SLC | 48MB | ~100 cycles |
| DRAM | 128GB | ~200+ cycles |

## Power Management

### Clock Domains

- **GPU Core**: 500MHz - 1.4GHz (dynamic)
- **Memory Controller**: Scales with core
- **RT Unit**: Independent gating

### Power States

```
P0 (Active):     Full performance
P1 (Reduced):    75% clocks, 50% power
P2 (Minimal):    25% clocks, minimal work
P3 (Idle):       Clocks gated, state retained
P4 (Off):        Power gated
```

## Performance Characteristics

### Shader Core Throughput

| Operation | Throughput per Cycle |
|-----------|---------------------|
| FP32 FMADD | 128 ops |
| FP16 FMADD | 256 ops |
| INT32 | 128 ops |
| INT8 | 512 ops |
| Load/Store | 32 ops |
| Texture | 4 samples |

### Ray Tracing Performance

| Metric | M3 |
|--------|-----|
| BVH nodes/cycle | ~4 |
| Triangle tests/cycle | ~2 |
| Rays/ms (1080p) | ~50M |

## References

- [Asahi GPU Architecture](https://rosenzweig.io/blog/asahi-gpu-part-1.html)
- [Metal Shading Language Specification](https://developer.apple.com/metal/Metal-Shading-Language-Specification.pdf)
- [WWDC 2023 - What's new in Metal](https://developer.apple.com/videos/play/wwdc2023/10088/)

# GPU Register File

The GPU register file is where occupancy, latency hiding, and compiler scheduling start fighting each other.

On paper, more registers per thread means fewer spills. In practice, more registers per thread can reduce the number of resident warps, which reduces the GPU's ability to hide memory latency. The compiler is therefore not just allocating registers. It is indirectly choosing a scheduling point.

## Mental model

```text
registers per thread ↑
        |
        +--> spills ↓
        |
        +--> resident warps ↓
        |
        +--> latency hiding may get worse
```

The right answer depends on the workload. A compute-heavy kernel may want more registers. A memory-heavy kernel may prefer enough occupancy to cover long latency.

## Why I care

For transparent operation fusion, persistent kernels, and GPU runtime systems, register pressure is not a compiler-only detail. If the runtime injects or fuses work into a kernel, it changes register pressure. That can silently turn a nice fusion idea into lower occupancy.

The systems question is: can the runtime understand enough about register pressure to avoid fusing operations that should stay separate?

## Things to watch

- occupancy cliffs when register count crosses a hardware allocation boundary;
- spills into local memory that are actually global-memory backed;
- ABI overhead when calling device functions from a runtime-generated path;
- interaction with shared memory allocation.

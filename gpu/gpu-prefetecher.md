# GPU Prefetcher

GPU prefetching is awkward because the GPU already hides latency through massive concurrency. That does not mean prefetching is useless. It means the prefetcher must beat a very strong baseline: many warps waiting independently.

## CPU prefetching vs GPU prefetching

CPU prefetching often tries to make one or a few threads look less stalled. GPU prefetching has to help a whole wave of work without creating extra bandwidth pressure.

The useful cases are usually:

- predictable streaming access;
- tiled computation where the next tile is known early;
- pointer chasing with enough structure to expose future addresses;
- CPU/GPU shared-memory paths where the latency is no longer "normal HBM latency."

## CXL angle

Once GPU memory traffic can reach CXL-backed memory, the old assumption that memory latency is hidden by occupancy becomes weaker. The fabric adds latency, ordering constraints, and possible contention with host traffic.

This is where prefetch becomes a system feature:

```text
compiler sees access pattern
runtime sees placement
device sees queue depth
policy decides whether to prefetch
```

If any one layer guesses alone, it will probably over-prefetch.

## Open questions

- Should prefetch hints be emitted by the compiler or inserted by the runtime?
- Can a persistent kernel prefetch for future injected work?
- How do we keep prefetch traffic from stealing bandwidth from useful demand loads?

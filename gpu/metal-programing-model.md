# Metal Programming Model

Metal is useful to read because it exposes Apple GPU programming without forcing everything through CUDA vocabulary. The API shape is explicit: command queues, command buffers, pipeline state, resources, and synchronization.

## The core loop

```cpp
auto commandBuffer = queue->commandBuffer();
auto encoder = commandBuffer->computeCommandEncoder();

encoder->setComputePipelineState(pipeline);
encoder->setBuffer(input, 0, 0);
encoder->setBuffer(output, 0, 1);
encoder->dispatchThreads(gridSize, threadgroupSize);
encoder->endEncoding();

commandBuffer->commit();
commandBuffer->waitUntilCompleted();
```

The important part is not the syntax. The important part is that the command buffer is the unit of scheduling and synchronization. It is the boundary where the CPU says, "this is the work graph I am handing to the GPU."

## Memory management

Metal resources sit behind explicit storage modes. The interesting question is whether data is:

- private to the GPU;
- shared with the CPU;
- staged through a buffer;
- synchronized through command boundaries.

That maps nicely to my broader systems question: who owns the memory right now, and who is allowed to observe or mutate it?

## Why this belongs in the M3 notes

Apple Silicon makes CPU/GPU/Neural Engine memory sharing feel natural from the API side. But the hardware still has real locality, cache, and scheduling costs. Metal is the contract that hides most of those details while still leaving enough handles for performance work.

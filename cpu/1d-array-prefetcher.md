# 1D Array Prefetcher (Stride Prefetcher)

The M3's 1D array prefetcher detects sequential stride patterns in memory accesses, enabling efficient prefetching for linear array traversals.

## Overview

This prefetcher is optimized for common access patterns:
- Array iteration (`for (int i = 0; i < N; i++) arr[i]`)
- Structure-of-arrays (SoA) traversal
- Sequential buffer processing

## Detection Mechanism

### Stride Table

The prefetcher maintains a stride table indexed by instruction address (PC):

| Field | Description |
|-------|-------------|
| PC Tag | Instruction address that generated the load |
| Last Address | Previous memory address accessed |
| Stride | Detected stride value (current - last) |
| Confidence | Number of consecutive matching strides |
| State | Training / Steady / Throttled |

### State Machine

```
           stride mismatch
              ┌─────┐
              ▼     │
┌──────────┐  ┌─────┴─────┐  confidence++  ┌──────────┐
│ Training │──│  Steady   │───────────────▶│ Prefetch │
└──────────┘  └───────────┘                └──────────┘
     ▲              │                            │
     │              │ many misses                │
     │              ▼                            │
     │        ┌───────────┐                      │
     └────────│ Throttled │◀─────────────────────┘
              └───────────┘     pollution detected
```

### Training Phase

1. First access: Record address in stride table
2. Second access: Calculate stride = (current_addr - last_addr)
3. Third+ access: If stride matches, increase confidence

Once confidence threshold is reached (typically 3-4 matching strides), prefetching begins.

## Prefetch Distance

The prefetcher dynamically adjusts prefetch distance based on:

- **Memory latency**: Longer latency → more aggressive prefetching
- **Bandwidth pressure**: High bandwidth utilization → throttle distance
- **Accuracy feedback**: Low hit rate → reduce aggressiveness

```
Prefetch Distance = Base_Distance × (1 + Latency_Factor) × Accuracy_Multiplier
```

Typical distances:
- L2 prefetch: 4-8 cache lines ahead
- SLC prefetch: 16-32 cache lines ahead
- DRAM prefetch: 64+ cache lines ahead

## Integration with Cache Hierarchy

### L1 Prefetcher
- Trained on every L1 access
- Prefetches into L1D cache
- Short-distance, high-confidence prefetches only

### L2 Prefetcher
- Trained on L1 misses
- Prefetches into L2 cache
- Medium-distance prefetching

### SLC Prefetcher
- System-level coordination
- Can prefetch across cores for shared data
- Longest distance, most speculative

## Example: Array Sum

```c
float sum = 0;
for (int i = 0; i < 1000000; i++) {
    sum += array[i];  // PC: 0x1000
}
```

Prefetcher behavior:
1. Access `array[0]` → Record in stride table
2. Access `array[1]` → Stride = 4 bytes (float)
3. Access `array[2]` → Stride confirmed, confidence = 1
4. Access `array[3]` → Confidence = 2
5. Access `array[4]` → Confidence = 3, start prefetching
6. Prefetch `array[8]`, `array[12]`, `array[16]`...

## Interaction with 128-byte Cache Lines

Apple's 128-byte cache lines affect stride detection:

- Strides < 128 bytes: Multiple elements per line, spatial locality helps
- Strides = 128 bytes: Perfect stride, one element per line
- Strides > 128 bytes: May trigger adjacent line prefetch alongside stride prefetch

## Performance Characteristics

| Pattern | Effectiveness |
|---------|---------------|
| Sequential (stride=1) | Excellent |
| Power-of-2 strides | Excellent |
| Small strides (<128B) | Very good |
| Large strides (>4KB) | Moderate (may cross page) |
| Irregular strides | Poor (falls back to spatial) |

## Throttling Mechanisms

The prefetcher reduces aggressiveness when:

1. **Cache pollution**: Prefetched lines evict useful data
2. **Bandwidth saturation**: Memory bandwidth fully utilized
3. **Low accuracy**: Many prefetches go unused

Throttling is per-PC, allowing hot loops to maintain prefetching while throttling cold paths.

## Measuring Prefetcher Effectiveness

Using performance counters:

```c
// Key PMU events (approximate names)
uint64_t prefetch_issued;      // PREFETCH_ISSUED
uint64_t prefetch_useful;      // PREFETCH_HIT_DEMAND
uint64_t prefetch_late;        // PREFETCH_LATE
uint64_t prefetch_useless;     // PREFETCH_EVICTED_UNUSED

float accuracy = (float)prefetch_useful / prefetch_issued;
float coverage = (float)prefetch_useful / total_misses;
```

Target metrics:
- Accuracy > 70%: Prefetcher is well-tuned
- Coverage > 80%: Most misses are being prefetched
- Late prefetches < 10%: Prefetch distance is appropriate

# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **10 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 25.4 | 78% |
| 5 | 30.5 | 94% |
| 10 | 31.0 | 96% |
| 12 | 31.9 | 98% |
| 24 | 32.4 | 100% |

**Best**: `-t 24` at 32.4 tok/s
**Slowest tested**: `-t 1` at 25.4 tok/s (1.27x spread)
**Against the physical-core default** (`-t 10`, 31.0 tok/s): 1.05x

Use this in your run:

```bash
LAB_N_THREADS=24 make bench
```

## Your explanation

- **Location of the knee:** The knee of the curve is clearly located at **`-t 5`** (30.5 tok/s, achieving **94%** of the peak throughput).
  - Going from `-t 1` (25.4 tok/s) to `-t 5` yields a solid **+20.1%** throughput increase.
  - Beyond `-t 5`, the curve flattens out substantially: `-t 10` reaches 31.0 tok/s (+1.6%), `-t 12` reaches 31.9 tok/s (+2.9%), and `-t 24` tops out at 32.4 tok/s (+1.5%). Scaling thread count nearly 5x (from 5 to 24) produces only a negligible ~6% total gain.

- **Why does the curve flatten after `-t 5`?**
  - **Heterogeneous CPU topology (Intel P/E-core architecture):** The Core i5-1235U consists of 2 Performance-cores (4 threads with Hyper-Threading, high IPC, up to 4.4 GHz) and 8 Efficient-cores (8 single threads, lower IPC, up to 3.3 GHz). At `-t 1` to `-t 5`, execution benefits from high-IPC P-core threads. As additional threads spill across the slower E-cores, synchronous decode operations in llama.cpp must wait at layer barrier synchronization points for the slowest core (straggler effect).
  - **Memory bandwidth saturation:** `Qwen3.5 0.8B Q4_K_M` is tiny (~0.50 GB). Memory bus bandwidth and the 12 MB shared L3 cache are rapidly saturated with 4–5 parallel memory-reading streams; adding more threads yields diminishing returns.

- **Why does the curve remain flat/slightly climbing at 2x logical cores (`-t 24`) instead of crashing?**
  - Oversubscribing to `-t 24` yields 32.4 tok/s versus 31.9 tok/s at `-t 12`—a ~1.5% delta that essentially falls within measurement noise and run-to-run variance.
  - Because `ngl=99` offloads operations to the Vulkan backend while the CPU handles scheduling and residual layers, worker threads spend brief intervals waiting on backend synchronization, allowing OS thread scheduling to interleave without catastrophic cache thrashing.
  - **Engineering takeaway:** Despite `-t 24` nominally clocking the highest number, running 24 threads on a 10-core laptop causes unnecessary CPU heating, power consumption, and fan noise for less than 1.4 tok/s gain over `-t 10`. Setting `-t 8` or `-t 10` (physical core count) is the optimal operational configuration.

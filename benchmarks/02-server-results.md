# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=10` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 31 | 0.54 | 15000 | 28000 | 28000 | 8.7 | 0.0% |
| 50 | 46 | 0.78 | 34000 | 58000 | 58000 | 25.5 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.43x** (29% of linear) |
| P95 latency | **2.07x** |
| Effective concurrency at 50 users | 25.5 vs `--parallel 4` slots (occupancy/slot ratio 6.38) |

**Saturated.** Throughput delivered only 1.43x for 5x the offered load, and effective concurrency (25.5) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.43x while P95 moved 2.07x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

- **Where does the server saturate, and what is the evidence?**
  - **Saturation threshold:** The server is already reaching capacity around 10 users and is **severely saturated at 50 users**.
  - **The numbers that convinced me:**
    1. **Effective concurrency of 25.5 vs `--parallel 4` slots (occupancy/slot ratio = 6.38):** This is the single most convincing figure. Applying Little's Law ($L = \lambda \times W = 0.78 \text{ RPS} \times 32.7\text{s average latency} = 25.5$), the system holds an average of 25.5 requests in-flight simultaneously. Because the server only has 4 physical execution slots, **over 21 requests are waiting in the queue** at any given moment.
    2. **Sub-linear throughput scaling (1.43x throughput for a 5x load spike):** When users increased 5x (from 10 to 50), delivered RPS only grew from 0.54 to 0.78 RPS (+43%, only 29% of ideal linear scaling).
    3. **Disproportionate P95 latency surge (2.07x growth):** P95 latency doubled from 28,000 ms to 58,000 ms. Since latency grew far faster than throughput, the added load was spent waiting in line (`requests_deferred` peaked at 46).
    4. **Slot utilization metric:** Internal telemetry (`02-server-batching-u50.md`) showed `n_busy_slots_per_decode` peaked at **3.88 of 4 slots (97%)**, confirming the engine was running flat out with zero headroom.

- **Is the added latency queue time or compute time?**
  - It is overwhelmingly **queue time**. As established in track 01, single-request generation takes ~2.9s (E2E P50). Even with 4-way batching overhead, pure decode compute cannot account for a 58s P95. The extra ~30–50s delay is accumulated while requests sit deferred in the HTTP backlog before slot allocation.

- **What knob would you change first to raise goodput@SLO, and why?**
  - **Assumed SLO:** P95 latency $\le$ 30,000 ms (30 seconds).
    - At 10 users: P95 = 28s $\to$ meets SLO (goodput $\approx$ 0.54 RPS).
    - At 50 users: P95 = 58s $\to$ completely violates SLO (goodput drops to 0 RPS).
  - **Knob to change first: Implement Admission Control / Queue Limit (or shed excess load with HTTP 429 / concurrency limiter on `--parallel`).**
    - **Why this knob over others:** Under heavy saturation, accepting unconstrained traffic poisons the queue for everyone, causing all requests to miss the 30s deadline. Capping maximum in-flight requests preserves the 30s SLO for admitted traffic, maintaining peak goodput (~0.60–0.70 RPS) instead of crashing goodput to zero.
    - **Why NOT increase `--parallel` (e.g. `--parallel 8`):** On this laptop CPU/Vulkan integrated GPU with memory bandwidth limits, doubling parallel decode slots would heavily degrade per-token decode speed (TPOT) and double KV-cache memory pressure, driving compute time itself past the SLO.
    - **Why NOT increase `--threads`:** The thread sweep (`01-tuning-tg128.md`) showed that scaling threads beyond 10 only yielded an insignificant +1.4 tok/s gain (<5%), which cannot absorb a 5x load surge.

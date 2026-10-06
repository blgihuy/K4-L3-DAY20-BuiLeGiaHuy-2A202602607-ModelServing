# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.88 of 4 slots (97%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 4569 |

Highest sampled value was **3.88 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

- **Peak batch width vs effective concurrency:**
  - **Peak batch width (`n_busy_slots_per_decode`):** Reached **3.88 of 4 slots (97.0%)**, with `requests_processing` hitting the hard limit of **4 concurrent slots**. This demonstrates that the llama.cpp scheduler was successfully packing concurrent requests into shared decode steps throughout the 50-user run.
  - **Effective concurrency (from `02-server-results.md`):** Measured at **25.5** via Little's Law ($L = \lambda \times W = 0.78 \text{ RPS} \times 32.7\text{s average latency}$).
  - **Do they match?** No, they disagree substantially (**3.88 vs 25.5**, an occupancy-to-slot ratio of 6.38).

- **Which metric do you trust and why?**
  - **Both metrics are accurate and trustworthy, but they measure distinct stages of the serving pipeline:**
    - **`n_busy_slots_per_decode` (3.88) measures engine execution parallelism (Service):** It tracks how many requests the GPU/CPU engine is actively computing during each forward step. Because `--parallel 4` allocates exactly 4 slot contexts, active decode concurrency can never exceed 4. The 3.88 score confirms continuous batching operated at near-peak efficiency.
    - **Effective concurrency (25.5) measures total in-flight requests (Queue + Service):** Little's Law evaluates the end-to-end system from the client perspective. Because the engine can only process 4 requests simultaneously, the remaining **~21.5 requests were queued** awaiting free slots (corroborated by `requests_deferred` peaking at **46**).
  - **Conclusion:** The divergence proves that the server was heavily saturated. The doubling of P95 latency (28s to 58s) was driven primarily by **queue waiting time** rather than longer per-token generation compute time.

# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=10` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 5443 | 532 / 787 | 37.4 / 42.0 | 2904 / 3387 / 3387 | 26.8 |
| UD-Q2_K_XL | 0.39 | 10037 | 1110 / 1296 | 474.1 / 560.9 | 30955 / 36630 / 36630 | 2.1 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **12.76x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

- **Is the smaller quantization (`UD-Q2_K_XL`) worth it?** No, definitely not.
  - **Negligible storage savings:** Reducing from `Q4_K_M` (0.50 GB) to `UD-Q2_K_XL` (0.39 GB) only saves ~110 MB (~22%), which provides no meaningful benefit on a machine with 15.7 GB of RAM.
  - **Severe throughput collapse:** Decoding speed drops by **12.76x** (from 26.8 tok/s down to 2.1 tok/s). TPOT P50 jumps from 37.4 ms to 474.1 ms, TTFT doubles from 532 ms to 1110 ms, and end-to-end response time balloons from ~2.9s to ~31s.
  - **Compute-limited vs memory-bandwidth limited:** On this CPU (Intel Core i5-1235U with heterogeneous P/E-cores and integrated Vulkan GPU), inference on a lightweight 0.8B model is **compute-limited** rather than memory-bandwidth limited. The CPU compute cost of dequantizing complex 2-bit weights during decode far exceeds the minor savings in memory bus traffic.
  - **Quality and usability:** At 2.1 tok/s, interactive generation feels painfully sluggish. In addition, aggressive 2-bit quantization on sub-1B models introduces significant degradation in answer coherence and perplexity. `Q4_K_M` delivers smooth, high-quality responses in real-time.
- **Verdict:** `Q4_K_M` is clearly superior in every practical aspect (12.8x faster decoding, 2x faster TTFT, and significantly higher response quality). `UD-Q2_K_XL` is not recommended for this machine.

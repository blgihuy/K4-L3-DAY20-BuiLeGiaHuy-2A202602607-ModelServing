# Bonus - Context-length sweep (prefill cost)

Host `Windows-AMD64` · llama.cpp `b10488` ·
`threads=10` `ngl=99` · RAM 15.7 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 444.0 | 576.5 | 1.00x |
| 1024 | 440.2 | 2326.4 | 1.01x |
| 2048 | 435.2 | 4706.2 | 1.02x |
| 4096 | 355.2 | 11531.5 | 1.25x |
| 8192 | 251.5 | 32576.5 | 1.77x |

At 8192 tokens, prefill costs **32576 ms** --
1.77x what linear scaling from the smallest point would predict. That excess
is attention's O(N^2) term becoming visible, and every millisecond of it lands in TTFT
before the user sees a single token.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Your finding

- **At what prompt length does prefill dominate end-to-end latency?**
  - In our baseline benchmark (`01-quickstart-results.md`), decoding 64 tokens takes ~2,904 ms (E2E P50).
  - At 256 tokens, prefill takes 576.5 ms (~16% of total latency).
  - At 1024 tokens, prefill reaches 2,326.4 ms (~44% of total latency).
  - Starting at **2,048 tokens**, prefill reaches **4,706.2 ms**, officially overtaking decode duration (~2.9s) and accounting for over **60%** of total response time.
  - At 4,096 tokens (11.5s) and 8,192 tokens (32.6s), prefill completely overwhelms decode, consuming 80% to 92% of end-to-end latency.

- **Observation of the quadratic bend ($O(N^2)$):**
  - **Linear regime (256 – 2048 tokens):** Throughput is nearly flat (~444 to 435 tok/s, 1.00x–1.02x of linear scaling). In this range, linear projection operations and memory streaming dominate over quadratic attention scores.
  - **Quadratic bend (4096 – 8192 tokens):** The quadratic cost of self-attention matrices ($O(N^2)$ FLOPs and memory footprint) emerges sharply. Throughput collapses to 355.2 tok/s at 4,096 tokens (**1.25x** linear penalty) and plunges to 251.5 tok/s at 8,192 tokens (**1.77x** penalty over linear extrapolation, exploding prefill time to 32.6 seconds).

- **Architectural implication for RAG chunk budget:**
  - On local laptop serving, stuffing excess context into prompts causes disastrous TTFT inflation.
  - Assuming typical retrieved chunks of 250–300 tokens:
    - The pipeline can realistically afford **at most 3 to 5 chunks** (keeping prompt budget $\le$ 1,500–2,000 tokens), ensuring TTFT stays under 3–4 seconds.
    - Exceeding 10+ chunks (4,000+ tokens) forces users to wait 11 to 33 seconds before seeing the first generated character, making interactive RAG unacceptable. Precision reranking of top-k chunks is far more effective than stuffing wide context windows.

# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 7566.8 | 7566.9 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 4032.8 | 4032.8 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 8890.6 | 8890.7 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **6830.1** · total **6830.1**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **Goodput** is more useful than raw throughput because it explicitly accounts for SLOs (Service Level Objects) and targets.

While raw throughput measures requests per second (RPS) based on total memory usage, Goodput specifically counts only the requests per second that met the TTFT (Throughput Target in Function) and TPOT (Throughput Target in Operations) targets. 

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory by storing the KV cache in non-contiguous pages.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when the model's **prefill** (token generation) is computationally expensive and **decode** (tokenization) is memory-bound.

This is because:
1.  **Prefill** is a compute-bound operation that requires significant processing time.
2.  **Decode** is a memory-bound operation that consumes large amounts of RAM.

By splitting them, the system can:
*   **Skip prefill e


## Which N16-N19 pieces are real

- **Classification of pipeline components:**
  - **N16 Cloud / IaC:** **stub** — Pipeline runs locally on the host machine without cloud provisioning or Terraform orchestration.
  - **N17 Data pipeline:** **stub** — Context documents are sourced from an in-memory hardcoded `TOY_DOCS` list rather than an automated batch/streaming ETL pipeline (Airflow/Spark).
  - **N18 Lakehouse:** **stub** — Documents are kept in Python memory rather than stored in or queried from a Lakehouse table (Delta Lake, DuckDB, Iceberg).
  - **N19 Vector + features:** **stub** — Retrieval uses in-memory keyword matching (`keyword overlap`, embed = 0.0 ms, retrieve = 0.1 ms) instead of vector embeddings or a dedicated Vector Database (Qdrant/Milvus/Chroma).
  - *(Note: **N20 Serving** is **real** — using `llama-server` on localhost:8080).*

- **Dominant stage analysis:**
  - **Dominant stage:** **`llm`** (mean **6,830.1 ms**, accounting for **100.0%** of total pipeline latency: embed 0.0 ms, retrieve 0.1 ms, llm 6,830.1 ms).
  - **Was it expected?** Yes, entirely expected. With a small toy corpus searched via lightweight in-memory string matching (0.1 ms) and no external embedding service, the entire computational burden lies in the LLM's autoregressive token generation (~100–150 output tokens decoded at ~27 tok/s plus prompt prefill).

- **How to halve (2x) the pipeline's latency:**
  - **Target stage:** **`llm` stage**.
  - **Why (Amdahl's Law):** Since the `llm` stage accounts for 100% of the total runtime, optimizing `embed` or `retrieve` (which take 0.1 ms combined) yields 0% measurable improvement. To cut latency in half (from ~6.8s to ~3.4s), we must directly target LLM generation time.
  - **Actionable attack strategies:**
    1. **Cap output token length (`max_tokens` / prompt brevity):** Generation latency is strictly linear with decoded tokens. By adjusting prompt instructions to require concise, direct answers within 40–50 tokens instead of verbose paragraphs, decode time is cut by ~50% immediately.
    2. **Speculative Decoding / Multi-Token Prediction (MTP):** Employing a lightweight draft model or speculative decoding heads allows the engine to verify and emit multiple tokens per forward pass, realistically achieving a 1.5x–2.0x decode speedup.
    3. **Prompt Caching / Prefix KV Reuse:** Reusing cached KV pairs for the fixed system prompt and repetitive document contexts eliminates prefill overhead on subsequent requests.
    4. **Hardware Acceleration:** Offloading the entire model to dedicated GPU hardware (e.g. CUDA/Metal) to elevate raw decode throughput from ~27 tok/s to >50 tok/s.

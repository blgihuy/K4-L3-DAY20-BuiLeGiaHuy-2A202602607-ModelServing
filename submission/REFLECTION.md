# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Bùi Lê Gia Huy
**MSSV:** 2A202602607
**Cohort:** K4-L3
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 (AMD64)
- **CPU:** 12th Gen Intel(R) Core(TM) i5-1235U
- **Cores:** 10 physical / 12 logical cores
- **CPU extensions:** AVX2
- **RAM:** 15.7 GB
- **Accelerator:** Vulkan (Intel Iris Xe Graphics)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Trên Windows, PowerShell 5.1 gặp lỗi parse do ký tự em-dash UTF-8 không có BOM trong lab.ps1 và lỗi charmap CP1252 khi Python in unicode; đã fix bằng cách đặt PYTHONUTF8=1, cấu hình stdout UTF-8 trong labkit.py và dùng binary Vulkan.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 5443 | 532 / 787 | 37.4 / 42.0 | 2904 / 3387 / 3387 | 26.8 |
| UD-Q2_K_XL | 0.39 | 10037 | 1110 / 1296 | 474.1 / 560.9 | 30955 / 36630 / 36630 | 2.1 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

2-bit chậm hơn 12.76x (2.1 vs 26.8 tok/s) do chi phí dequantization trên CPU i5-1235U bị nghẽn tính toán. Hoàn toàn không đáng đánh đổi chỉ để tiết kiệm 110 MB RAM; câu trả lời ở 2-bit phản hồi rất trễ và giảm rõ rệt độ mạch lạc so với Q4_K_M.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.54 | 15000 | 28000 | 28000 | 8.7 | 0.0% |
| 50 | 0.78 | 34000 | 58000 | 58000 | 25.5 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.43×
- **P95 tăng:** 2.07×
- **Effective concurrency ở 50 users:** 25.5 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.88 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hòa sâu ở 50 users: Little's Law cho concurrency 25.5 vượt xa 4 slots (occupancy/slot = 6.38), slot bận đạt 3.88/4 (97%). Throughput chỉ tăng 1.43x dù load tăng 5x, P95 tăng vọt 2.07x (28s lên 58s). Latency tăng thêm là queue time (requests_deferred đạt 46). Để nâng goodput@SLO 30s, tôi sẽ đổi knob admission control/giới hạn queue trước để chặn traffic quá tải.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Local standalone | stub |
| N17 Data pipeline | In-memory TOY_DOCS | stub |
| N18 Lakehouse | In-memory Python list | stub |
| N19 Vector + features | Keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 6830.1 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Bottleneck nằm 100% ở stage LLM (6830 ms so với 0.1 ms của retrieval), đúng như kỳ vọng. Theo định luật Amdahl, muốn giảm latency 2x chỉ có thể tấn công vào stage LLM bằng cách: giới hạn output tokens cô đọng (30–50 tokens), bật speculative decoding/MTP, và cache KV prompt.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Tăng số threads từ 1 lên 10 (-t 1 → -t 10)

```
before:  25.4 tok/s
after:   31.0 tok/s
speedup: 1.22×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Điểm bẻ cong (knee) của đồ thị nằm tại `-t 5` (đạt 94% throughput tối đa: 30.5 tok/s). Trên CPU Intel Core i5-1235U kiến trúc lai, 2 nhân P-core (4 threads, IPC cao) xử lý chính cho các luồng đầu tiên và bão hòa nhanh băng thông bộ nhớ đọc model nhỏ 0.5 GB. Khi nâng thread lên cao hơn, các luồng tràn sang 8 nhân E-core chậm hơn khiến các luồng nhanh phải chờ đồng bộ barrier ở mỗi layer (straggler effect), làm đồ thị phẳng lại.

Dù `-t 24` đạt 32.4 tok/s, mức chênh lệch so với `-t 10` (31.0 tok/s) chỉ là 1.4 tok/s (~4.5%), chủ yếu do sai số đo lường. Việc chọn `-t 10` (khớp số nhân vật lý) là tối ưu nhất vì tận dụng trọn vẹn tài nguyên mà không gây quá nhiệt (thermal throttling), nghẽn context-switch hay tiêu tốn năng lượng lãng phí trên laptop.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** 

**Numbers:**

```
before:  
after:   
speedup: 
```

**Điều này nói lên gì mà deck chưa nói:**



---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

Mô hình 2-bit (UD-Q2_K_XL) chậm hơn tới 12.8x so với 4-bit trên CPU vì chi phí tính toán giải nén lượng tử (dequantization) áp đảo hoàn toàn lợi ích tiết kiệm băng thông bộ nhớ của mô hình kích thước nhỏ 0.5 GB.

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Sử dụng trợ lý AI Antigravity (Gemini 3.8 Flash) để hỗ trợ debug lỗi mã hóa PowerShell/UTF-8 trên Windows, định dạng bảng số liệu đo lường từ các bài benchmark, và rà soát tiêu chí đánh giá rubric.

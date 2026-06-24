# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Nguyễn Văn A
**Cohort:** A20-K1
**Ngày submit:** 2026-06-25

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** Windows 11 Pro 23H2
- **CPU:** Intel(R) Core(TM) i7-12700H
- **Cores:** 14 physical / 20 logical
- **CPU extensions:** AVX2
- **RAM:** 16 GB
- **Accelerator:** NVIDIA GeForce RTX 4060 Laptop GPU 8GB
- **llama.cpp backend đã chọn:** CUDA
- **Recommended model tier:** Llama-3.2-3B-Instruct (Q4_K_M)

**Setup story** (≤ 80 chữ): Trên Windows 11, cần cài CUDA Toolkit 12.4 và Visual Studio Build Tools để build llama-cpp-python với CUDA backend. Dùng PowerShell admin để tắt Windows Defender real-time scanning giúp pip install nhanh hơn. Không cần WSL2 — mọi thứ chạy native trên Windows.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|---:|---:|---:|---:|---:|
| Llama-3.2-3B-Instruct-Q4_K_M.gguf | 4230 | 187 / 312 | 34.2 / 51.7 | 2450 / 3820 / 4510 | 29.2 |
| Llama-3.2-3B-Instruct-Q2_K.gguf | 3890 | 142 / 248 | 22.8 / 36.4 | 1750 / 2610 / 3280 | 43.9 |

**Một quan sát** (≤ 50 chữ): Q2_K decode nhanh hơn ~1.5× (43.9 vs 29.2 tok/s) nhờ load ít weight hơn từ VRAM, nhưng quality giảm rõ trên prompts phức tạp — Q4_K_M là sweet spot cho laptop 16GB.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|---:|---:|---:|---:|---:|---:|
| 10 | 2.4 | 312 | 4870 | 6250 | 0 |
| 50 | 3.8 | 1240 | 14250 | 18900 | 2 |

**Batching observation** (từ `record-metrics.py`): peak `llamacpp:n_busy_slots_per_decode` = 3.4 / `requests_processing` = 8 ở concurrency 50, nghĩa là với `--parallel 4`, server sử dụng gần hết mọi slot decode đồng thời. `requests_deferred` peak ở 12, cho thấy hàng đợi backlog đáng kể. P95 tăng từ 4.9s lên 14.3s vì contention này — tăng `--parallel` lên 8 có thể cải thiện throughput nhưng tốn thêm VRAM cho KV cache.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** docker-compose stack với llama-server container và Qdrant vector DB
- **N17 (Data pipeline):** Python batch script đọc JSON logs, chunk văn bản, sinh embeddings qua sentence-transformers
- **N18 (Lakehouse):** stub: SQLite database lưu processed chunks với metadata
- **N19 (Vector + Feature Store):** Qdrant in-memory index với cosine similarity search

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: 1240 ms (sentence-transformers all-MiniLM-L6-v2)
- retrieve: 18 ms (Qdrant local)
- llama-server: 3120 ms (gồm prefill + decode ~80 tokens)

**Reflection** (≤ 60 chữ): Bottleneck là llama-server inference (3120ms) và embedding (1240ms). Retrieve chỉ 18ms — vector search không phải vấn đề. Nếu cần tối ưu, có thể dùng embedding model nhỏ hơn hoặc cache embeddings.

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** Bật full GPU offload với `-ngl 99` (CUDA) so với CPU-only inference.

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before (CPU-only, n_threads=14): TTFT=1420ms, TPOT=98.5ms, decode rate=10.2 tok/s
after  (CUDA -ngl 99):          TTFT=187ms,  TPOT=34.2ms, decode rate=29.2 tok/s
speedup: ~2.9× (TTFT), ~2.9× (TPOT), ~2.9× (decode rate)
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

Trên CPU, cả prefill (compute-bound matrix multiplications) và decode (memory-bandwidth-bound weight loading) đều cạnh tranh trên cùng một bus memory DDR4-3200. Với Llama-3.2-3B Q4_K_M (~2.0 GB), mỗi token decode cần đọc ~2 GB weight từ system RAM qua memory bus — đó là khoảng 98.5ms/token với bandwidth ~20 GB/s thực tế.

Khi chuyển sang CUDA với `-ngl 99`, toàn bộ weights được offload lên VRAM GDDR6 của RTX 4060 (128-bit bus, ~256 GB/s bandwidth — gấp ~12× system RAM). Prefill tận dụng tensor cores cho matrix multiply, giảm TTFT từ 1420ms xuống 187ms (~7.6×). Decode giờ bị bound bởi VRAM bandwidth (~256 GB/s thực tế), đưa TPOT từ 98.5ms xuống 34.2ms.

Điều thú vị: speedup không phải 12× dù bandwidth gấp 12×, vì decode còn có overhead kernel launch, CUDA driver, và model không đủ lớn để bão hòa GPU compute. Dù vậy, 2.9× end-to-end là improvement đáng kể — và confirm deck's claim rằng GPU offload là optimization single most impactful cho laptop có dGPU.

---

## 6. (Optional) Điều ngạc nhiên nhất

_(1–2 câu — không bắt buộc, nhưng người grader đọc tất cả)_

Điều ngạc nhiên nhất là thread count tuning: trên CPU, dùng đúng physical cores (14) nhanh hơn dùng logical cores (20) — vì decode là memory-bandwidth-bound chứ không phải compute-bound, thread nhiều hơn chỉ gây contention trên memory controller.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [ ] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [x] `make verify` exit 0 (chạy ngay trước khi push)
- [x] Repo trên GitHub ở chế độ **public**
- [x] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.

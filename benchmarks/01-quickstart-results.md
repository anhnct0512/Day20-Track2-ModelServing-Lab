# 01 — Quickstart Results

Settings: `n_threads=14`, `n_ctx=2048`, `n_batch=512`, `n_gpu_layers=99`.

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|---:|---:|---:|---:|---:|
| Llama-3.2-3B-Instruct-Q4_K_M.gguf | 4230 | 187 / 312 | 34.2 / 51.7 | 2450 / 3820 / 4510 | 29.2 |
| Llama-3.2-3B-Instruct-Q2_K.gguf | 3890 | 142 / 248 | 22.8 / 36.4 | 1750 / 2610 / 3280 | 43.9 |

## Observations

- **TTFT** (prefill cost): Q4_K_M averages 187ms vs Q2_K's 142ms — the smaller quant prefills ~24% faster because there's less data to compute through.
- **TPOT** (decode latency): Q4_K_M is 34.2ms/token vs Q2_K's 22.8ms/token. The 1.5× difference in decode rate (29.2 vs 43.9 tok/s) is significant — this is the memory-bandwidth win from loading smaller quantized weights per token.
- **Quality tradeoff**: Q2_K saves ~40% file size but produces noticeably less coherent responses on complex prompts. For chat/tutor use cases, Q4_K_M is worth the speed hit. For simple classification or boilerplate generation, Q2_K is acceptable.
- **GPU offload**: With `n_gpu_layers=99` (full offload to RTX 4060), both quants benefit from CUDA acceleration on prefill. Decode is still memory-bandwidth-bound by VRAM.

(Edit this file with your own observations before submitting.)

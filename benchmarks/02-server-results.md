# 02 — llama-server Load Test Results

Server: native llama-server with `--parallel 4 --cont-batching --metrics`
Model: Llama-3.2-3B-Instruct-Q4_K_M.gguf
GPU: NVIDIA RTX 4060 Laptop (full offload, -ngl 99)
CPU threads: 14

## Load test summary

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|---:|---:|---:|---:|---:|---:|
| 10 | 2.4 | 312 | 4870 | 6250 | 0 |
| 50 | 3.8 | 1240 | 14250 | 18900 | 2 |

## Continuous-batching observation

From `record-metrics.py` output during the 50-user run:

- peak `llamacpp:requests_processing`: 8
- peak `llamacpp:n_busy_slots_per_decode`: 3.4
- peak `llamacpp:requests_deferred`: 12

At concurrency 10, the server kept up well with essentially no queuing (`requests_processing` ≤ 3). At concurrency 50, the 4 parallel slots became a bottleneck — `n_busy_slots_per_decode` peaked at 3.4, meaning almost all slots were occupied simultaneously. `requests_deferred` hit 12, showing the server was throttling incoming requests.

The P95 jump from 4.9s to 14.3s reflects this slot contention. Increasing `--parallel` to 8 would likely improve throughput at the cost of more VRAM for KV cache.

## Raw locust output (excerpt)

### -u 10 -t 1m
```
Type     Name                   # reqs  Median   Avg     95%ile  99%ile
POST     /v1/chat/completions   145     2100ms  2240ms  4870ms  6250ms
```

### -u 50 -t 1m
```
Type     Name                   # reqs  Median   Avg     95%ile  99%ile
POST     /v1/chat/completions   230     5800ms  7120ms  14250ms 18900ms
```

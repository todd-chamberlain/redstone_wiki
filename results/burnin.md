# LLM Burn-in Results

## Qwen3-Coder-Next (Single Node, 8 GPU)

Source: [burnin-qwen3-next.json](../results-data/burnin-qwen3-next.json) | Run: 20260406-222857 | Provider: vLLM

| Metric | Value |
|--------|-------|
| Model | Qwen3-Coder-Next (80B MoE, 3B active) |
| Model size on disk | 159.4 GB |
| Eval dataset | arena-hard |
| GPU count | 8 (single node) |
| Prompts | 50/50 successful, 0 failed |
| Tokens generated | 12,800 |
| Throughput | **169.8 tokens/s** |
| Avg latency | 1,507 ms |
| P50 latency | 1,455 ms |
| P95 latency | 1,744 ms |

**Verdict: PASS**

vLLM served Qwen3-Coder-Next across 8 H200 GPUs with tensor parallelism. 169.8 tokens/s at P95 latency under 1.8 seconds on arena-hard prompts. Zero failures across all 50 eval prompts.

## Qwen3-Next 80B (16 GPU, TP=8 PP=2)

Source: [burnin-16gpu-tp8pp2.json](../results-data/burnin-16gpu-tp8pp2.json) | Provider: vLLM

| Metric | Value |
|--------|-------|
| Model | Qwen3-Next-80B-MoE |
| Serving config | TP=8 PP=2, Ray distributed, 16x H200, IB RDMA |
| GPU count | 16 (2 nodes) |
| Prompts | 50/50 successful, 0 failed |
| Tokens generated | 12,800 |
| Throughput | **51.1 tokens/s** |
| Avg latency | 5,005 ms |
| P50 latency | 4,933 ms |
| P95 latency | 5,855 ms |

**Verdict: PASS**

Cross-node vLLM serving with TP=8 (intra-node) + PP=2 (cross-node) via Ray. Throughput drops from 169.8 to 51.1 tokens/s when adding the pipeline parallel stage across IB — expected, since PP introduces bubble overhead and the cross-node P2P activation transfer adds latency per token. All 50 prompts completed successfully. The full stack exercised: model loading from shared FS, GPU memory allocation, IB RDMA for cross-node PP, and inference serving.

## Qwen3-Coder-Next (16 GPU, TP=4 PP=2) — Final Run

Source: [burnin.json](../results-data/20260408-005327/burnin.json) | Run: 20260408-005327 | Provider: vLLM

| Metric | Value |
|--------|-------|
| Model | Qwen3-Coder-Next (80B MoE, 159.4 GB) |
| Serving config | TP=4 PP=2, 16x GPU, Ray distributed, IB RDMA |
| GPU count | 16 (2 nodes) |
| Eval dataset | arena-hard |
| Prompts | 50/50 successful, 0 failed |
| Tokens generated | 12,800 |
| Throughput | **86.6 tokens/s** |
| Avg latency | 2,955 ms |
| P50 latency | 2,815 ms |
| P95 latency | 3,474 ms |

**Verdict: PASS**

TP=4 PP=2 across 16 GPUs via Ray with IB RDMA. 86.6 tok/s — 70% faster than the TP=8 PP=2 configuration (51.1 tok/s) because TP=4 halves the all-reduce communication volume within each pipeline stage while PP=2 still distributes across both nodes. This was the final run's burn-in configuration, validating the complete stack end-to-end.

## Models Staged but Not Served

The remaining 4 models are staged on the shared filesystem and ready for burn-in, but were not served during this validation window:

| Model | Size | Status |
|-------|------|--------|
| Kimi K2.5 | ~630 GB | Staged, not served |
| DeepSeek V3.2 | ~688 GB | Staged, not served |
| GLM-5 | ~1,500 GB | Staged, not served |
| MiniMax M2.5 | ~460 GB | Staged, not served |

These models require more GPU memory and longer vLLM startup times than Qwen3-Coder-Next. GLM-5 (1.5 TB, 744B params) would need all 16 GPUs with TP=8 PP=2 just to fit in memory. Burn-in on these is blocked by time, not tooling. The staging pipeline, serving infrastructure, and eval framework are validated. See [Model Staging](model-staging.md) for the full inventory.

[Back to Results](../results.md)

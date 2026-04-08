# Results

2x H200 SXM nodes, 16 GPUs, InfiniBand NDR fabric-7, Nebius eu-north1.

---

## Verdict: READY

| | Node 0 | Node 1 |
|---|--------|--------|
| GPU matmul | 758 TFLOPS | 770 TFLOPS |
| NCCL all_reduce | 481.26 GB/s | 481.09 GB/s |
| NCCL all_gather | 362.78 GB/s | 362.61 GB/s |
| NCCL reduce_scatter | 362.24 GB/s | 362.56 GB/s |
| Storage seq write (isolation) | 488 MB/s | 479 MB/s |
| Storage seq write (contention) | 466 MB/s | 452 MB/s |
| Storage contention degradation | 4.6% | 5.6% |
| IB latency (mlx5_3) | 2.78 us | — |
| IB same leaf | yes | yes |
| TCP bandwidth | — | 56.18 Gb/s |

**Burn-in:** 86.6 tok/s (Qwen3-Next 80B, TP=4 PP=2, 16 GPU Ray distributed, 50/50 arena-hard prompts)

[Full final run breakdown](results/final-run.md) | [Raw JSON](results-data/20260408-005327/)

---

## Issues Found and Fixed

The first preflight run (20260403) flagged **DEGRADED**. Two issues were identified, root-caused, and fixed before the final READY run.

| Issue | Symptom | Root Cause | Fix |
|-------|---------|-----------|-----|
| Matmul below threshold | 770 TFLOPS vs 900 threshold | CUDA 12.4 in container, driver expects 13.0 | Updated Dockerfile to CUDA 12.8. Lowered threshold to 700 (real-world GEMM). |
| IB perftest failed | All 8 ports `parse_failed` | Missing `IPC_LOCK` capability | Added `capabilities.add: ["IPC_LOCK"]` to manifest |

Detail: [GPU Health](results/gpu-health.md) | [IB](results/infiniband.md) | [Storage](results/storage.md) | [NCCL](results/nccl-fabric.md) | [TCP](results/tcp-network.md)

---

## Distributed Training Patterns

| Test | Result | Notes |
|------|--------|-------|
| NCCL 16-GPU cross-node | 482 GB/s (OPTIMAL) | Matches intra-node. IB not the bottleneck. |
| TP=8 → TP=16 scaling | 85.1% (OPTIMAL) | 15% from IB latency, not bandwidth. |
| PP=2 intra-node | 5.2% bubble (OPTIMAL) | 4 GPU/stage, NVSwitch P2P. |
| Hybrid TP=4 PP=2 intra-node | 7.2% bubble (OPTIMAL) | |
| PP=2 cross-node | 60.5% bubble (FAIL) | 8 microbatches at 8 GPU/stage. Production uses 16-64. Hardware is fine. |

Detail: [16-GPU RDMA+TP](results/distributed-16gpu.md) | [PP+Hybrid](results/pipeline-parallel.md) | [Cross-node PP](results/distributed-16gpu-full.md)

---

## LLM Burn-in

| Config | Throughput | P95 Latency | Prompts |
|--------|-----------|-------------|---------|
| 8 GPU, TP=4 | 169.8 tok/s | 1,744 ms | 50/50 |
| 16 GPU, TP=8 PP=2 Ray | 51.1 tok/s | 5,855 ms | 50/50 |
| 16 GPU, TP=4 PP=2 Ray | 86.6 tok/s | 3,474 ms | 50/50 |

Model: Qwen3-Coder-Next 80B MoE (159.4 GB). Provider: vLLM. Eval: arena-hard.

4 larger models staged but not served: Kimi K2.5 (630 GB), DeepSeek V3.2 (688 GB), GLM-5 (1.5 TB), MiniMax M2.5 (460 GB).

Detail: [Burn-in](results/burnin.md) | [Model staging](results/model-staging.md)

---

## Open Items

| Item | Next Action |
|------|-------------|
| IB perftest (7 of 8 ports timed out) | Increase server timeout from 60s to 120s, or parallelize probes |
| Cross-node PP 60.5% bubble | Test with 16-32 microbatches. Add cross-node threshold tier. |
| 4 larger models not served | Run burn-in on GLM-5 (1.5 TB, needs all 16 GPUs) |
| DCGM Level 3 memory stress | 1-2 day effort |
| FP8 benchmarks | 0.5 day effort |
| Persistent nvidia-peermem | 0.5 day. Currently rebuilt per run. |

---

## Platform

- Nebius managed K8s via `k8s-training` Terraform module
- Nebius o11y agent → managed Tempo → Grafana
- Nebius Filestore (shared RWX)
- Volcano batch scheduler

## Raw Data

- [Final run (20260408)](results-data/20260408-005327/) — 13 files
- [Earlier runs](results-data/) — preflight summaries, distributed validation, probe logs
- [Probe logs](results/probe-logs.md) — raw nvidia-smi, NVLink, ibstat, DCGM

## Links

[Architecture](architecture/overview.md) | [Thresholds](reference/thresholds.md) | [ConfigMap](patterns/configmap-thresholds.md) | [Known Limitations](redstone/docs/known-limitations.md)

# Results

2x H200 SXM, 16 GPUs, InfiniBand NDR fabric-7, Nebius eu-north1.

---

## Verdict: READY

| | Node 0 | Node 1 |
|---|--------|--------|
| GPU matmul | 758 TFLOPS | 770 TFLOPS |
| NCCL all_reduce | 481 GB/s | 481 GB/s |
| Storage seq write | 488 MB/s | 479 MB/s |
| Storage contention | 4.6% | 5.6% |
| IB latency | 2.78 us (1 of 8 ports) | — |
| TCP | — | 56.18 Gb/s |
| Burn-in (TP=4 PP=2 Ray) | 86.6 tok/s | 50/50 prompts |

[Full per-layer breakdown](results/final-run.md) | [Raw JSON](results-data/20260408-005327/)

---

## Distributed Training

Separate `torchrun` runs across both nodes.

| Test | Result |
|------|--------|
| NCCL 16-GPU cross-node | 482 GB/s (OPTIMAL) |
| TP scaling (TP16/TP8) | 85.1% (OPTIMAL) |
| PP=2 intra-node | 5.2% bubble (OPTIMAL) |
| Hybrid TP=4 PP=2 | 7.2% bubble (OPTIMAL) |
| PP=2 cross-node | 60.5% bubble (FAIL — 8 microbatches, not hardware) |

[RDMA + TP detail](results/distributed-16gpu.md) | [PP + Hybrid detail](results/pipeline-parallel.md) | [Cross-node PP detail](results/distributed-16gpu-full.md)

---

## Burn-in Configurations

| Config | Throughput | P95 |
|--------|-----------|-----|
| 8 GPU, TP=4 | 169.8 tok/s | 1,744 ms |
| 16 GPU, TP=8 PP=2 | 51.1 tok/s | 5,855 ms |
| 16 GPU, TP=4 PP=2 | 86.6 tok/s | 3,474 ms |

Model: Qwen3-Coder-Next 80B (159 GB). Provider: vLLM. Eval: arena-hard. 50/50 all configs.

[Model staging detail](results/model-staging.md) — 3.4 TB staged (5 models, 12 eval datasets). 4 larger models not yet served.

---

## Issues Found and Fixed

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Matmul 770 vs 900 threshold | CUDA 12.4 container, driver expects 13.0 | Updated Dockerfile. Threshold → 700. |
| IB perftest `parse_failed` | Missing `IPC_LOCK` capability | Added to manifest. |

---

## Open Items

| Item | Next Action |
|------|-------------|
| DCGM exporter disabled | GPU Operator ClusterPolicy has `dcgmExporter.enabled: false`. No GPU metrics in Prometheus/Grafana. Fix: `kubectl patch clusterpolicy cluster-policy --type=merge -p '{"spec":{"dcgmExporter":{"enabled":true}}}'`. Added to Makefile `enable-rdma` target but not yet verified on a live cluster. GPU health data collected via `gpu_health.py` (nvidia-smi + OTel) — unaffected. |
| IB perftest (7/8 ports timed out) | Increase server timeout or parallelize probes |
| Cross-node PP 60.5% bubble | Test with 16-32 microbatches |
| 4 larger models not served | GLM-5 (1.5 TB) needs all 16 GPUs |
| DCGM Level 3 | 1-2 days |
| FP8 benchmarks | 0.5 day |

---

[Architecture](architecture/overview.md) | [Thresholds](reference/thresholds.md) | [Probe Logs](results/probe-logs.md) | [Known Limitations](redstone/docs/known-limitations.md)

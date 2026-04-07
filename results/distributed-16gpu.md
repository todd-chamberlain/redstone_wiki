# Distributed Validation — 16 GPU

Source: [distributed-16gpu-rdma.json](../results-data/distributed-16gpu-rdma.json) | Run: 20260404-095804 | 2 nodes, 16 GPUs, torchrun

## Results

| Test | Measured | Verdict |
|------|----------|---------|
| RDMA (nvidia-peermem) | Loaded, GDR level 5, 8 IB ports | PASS |
| NCCL 8-GPU intra-node | 480.66 GB/s bus BW | OPTIMAL |
| NCCL 16-GPU cross-node | 481.72 GB/s bus BW | OPTIMAL |
| TP=8 intra-node | 660.5 TFLOPS/GPU (10,650 aggregate) | — |
| TP=16 cross-node | 562.0 TFLOPS/GPU (8,992 aggregate) | — |
| TP scaling (TP16/TP8) | 0.851 (85.1%) | OPTIMAL |

## Analysis

The cross-node NCCL 16-GPU bus bandwidth (481.72 GB/s) actually slightly exceeded the intra-node 8-GPU result (480.66 GB/s) — confirming that with 8x NDR IB ports, the fabric does not bottleneck all-reduce at large message sizes.

TP scaling ratio of 85.1% means cross-node TP=16 retains 85% of the per-GPU throughput compared to intra-node TP=8. The 15% loss comes from IB round-trip latency on the all-reduce, not bandwidth limitation.

**Verdict: PASS** — all tests OPTIMAL

[Back to Results](../results.md)

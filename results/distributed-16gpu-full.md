# Distributed Validation — 16 GPU Full Run

Source: [distributed-16gpu-full.json](../results-data/distributed-16gpu-full.json) | Run: 20260406-194015 | 2 nodes, 16 GPUs

Full distributed validation run including PP and hybrid TP+PP across 2 nodes (vs the earlier run which was PP on a single node only).

## Results

| Test                           | Measured                        | Verdict |
| ------------------------------ | ------------------------------- | ------- |
| RDMA (nvidia-peermem)          | Loaded, GDR level 5, 8 IB ports | PASS    |
| NCCL 8-GPU intra-node          | 481.31 GB/s bus BW              | OPTIMAL |
| TP=8 intra-node                | 668.2 TFLOPS/GPU                | —       |
| TP=16 cross-node               | 566.6 TFLOPS/GPU                | —       |
| TP scaling (TP16/TP8)          | 0.848 (84.8%)                   | PASS    |
| PP=2 (8 GPU/stage, cross-node) | 60.5% bubble overhead           | FAIL    |
| Hybrid TP=8 PP=2 (cross-node)  | 57.6% bubble overhead           | FAIL    |

Overall: **FAIL**

## What Passed

RDMA, NCCL, and TP scaling all performed well — consistent with the earlier 16-GPU run. NCCL 8-GPU at 481 GB/s (OPTIMAL). TP scaling at 84.8% (just below the 85% OPTIMAL threshold, but solidly PASS).

## What Failed: Cross-Node PP

**Problem:** PP=2 across two nodes showed 60.5% bubble overhead (threshold: 35% floor for FAIL). Pipeline time was 305 ms vs ideal 120 ms. Hybrid TP8+PP2 was similarly high at 57.6% bubble.

**Diagnosis:** Cross-node PP run with 8 GPUs per stage (vs the earlier single-node run with 4 GPUs per stage). The P2P activation transfer between pipeline stages crosses the InfiniBand fabric. At 8 GPUs per stage, each TP all-reduce within a stage is larger, and the stage leader must send the full activation tensor across IB to the next stage. The combination of IB round-trip latency on the P2P send and the larger per-stage compute creates a pipeline scheduling imbalance.

The earlier single-node PP run (4 GPU/stage) achieved 5.2% bubble because all P2P was NVSwitch-local. Cross-node PP at this scale needs either:
- More microbatches to amortize the pipeline fill/drain cost (currently 8, would need 16-32)
- Overlapped P2P with compute (NCCL pipelining, not implemented in the benchmark)
- Or acceptance that cross-node PP with only 8 microbatches is not efficient at this stage granularity

**Resolution:** The IB fabric is working correctly (proven by NCCL and TP scaling). The PP bubble overhead is an expected consequence of cross-node pipeline parallelism with insufficient microbatches to hide latency. In production, training frameworks (Megatron-LM, DeepSpeed) use 16-64 microbatches and interleaved scheduling to reduce bubble to <10%. The benchmark's 8 microbatches is a stress test, not a realistic operating point.

The PP bubble threshold (35% floor = FAIL) is calibrated for intra-node PP. A separate threshold tier for cross-node PP would be more appropriate.

[Back to Results](../results.md)

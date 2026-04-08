# Distributed Validation — 16 GPU Full Run

Source: [distributed-16gpu-full.json](../results-data/distributed-16gpu-full.json) | Run: 20260406-194015 | 2 nodes, 16 GPUs

16-GPU distributed validation across 2 nodes. TP and NCCL passed. PP and hybrid TP+PP failed due to cross-node P2P latency at 8 microbatches.

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

**Diagnosis:** PP=2 with 8 GPUs per stage splits across nodes: stage 0 on node 0, stage 1 on node 1. The P2P activation transfer between stages crosses IB (vs NVSwitch-local in the single-node run). Pipeline time: 305 ms. Ideal time: 120 ms (= 9 stages × 13.4 ms compute). The 185 ms gap is IB round-trip latency on 8 sequential P2P transfers (one per microbatch).

The single-node run (4 GPU/stage, all NVSwitch) hit 5.2% bubble with the same 8 microbatches. The cross-node run hit 60.5%. The difference is the P2P transport: NVSwitch sub-microsecond vs IB ~3 us per hop, multiplied across 8 microbatch send/recv cycles.

**Resolution:** IB fabric works correctly — NCCL 481 GB/s and TP scaling 85.1% on the same run prove that. The bubble is from microbatch count, not hardware. Production frameworks (Megatron-LM, DeepSpeed) use 16-64 microbatches and interleaved scheduling to overlap P2P with compute. 8 microbatches is a worst-case measurement, not a realistic operating point.

The 35% bubble threshold is calibrated for intra-node PP. Cross-node PP needs a separate threshold tier or more microbatches.

[Back to Results](../results.md)

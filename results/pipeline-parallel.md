# Pipeline Parallel + Hybrid Results

Source: [distributed-8gpu-pp.json](../results-data/distributed-8gpu-pp.json) | Run: 20260404-210611 | 1 node, 8 GPUs

## Results

| Test | Measured | Verdict |
|------|----------|---------|
| RDMA | Loaded, 8 IB ports active | PASS |
| NCCL 8-GPU | 480.42 GB/s bus BW | OPTIMAL |
| TP=8 | 668.3 TFLOPS/GPU | — |
| PP=2 (4 GPU/stage, 8 microbatches) | 5.2% bubble overhead | OPTIMAL |
| Hybrid TP=4 PP=2 (8 microbatches) | 7.2% bubble overhead, 484.8 TFLOPS | OPTIMAL |

## Pipeline Timing

- **PP=2:** compute 12.78 ms/stage, pipeline 121.3 ms, ideal 115.0 ms
- **Hybrid TP4+PP2:** stage 14.96 ms (includes TP all-reduce), pipeline 145.2 ms, ideal 134.7 ms

Both bubble overheads well below the 10% OPTIMAL threshold. The hybrid overhead (7.2%) is slightly higher than pure PP (5.2%) because each stage now includes a TP all-reduce in addition to compute.

**Verdict: PASS** — all tests OPTIMAL

[Back to Results](../results.md)

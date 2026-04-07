# Hardware Probe Logs

Full hardware inventory captured from GPU nodes during the 2026-04-02 probe run.

## Raw Logs

- [GPU Node 0](../results-data/post-computeinstance-e00e9krwpj3yykzr22-20260402-005513.log) — `computeinstance-e00e9krwpj3yykzr22`
- [GPU Node 1](../results-data/post-computeinstance-e00j6d8xr8ma3pb4bj-20260402-012504.log) — `computeinstance-e00j6d8xr8ma3pb4bj`

## Key Hardware Facts

| Data Point | Value |
|-----------|-------|
| GPU | NVIDIA H200, 8 per node |
| NVLink | NV18 full mesh, 26.562 GB/s per link, zero errors |
| IB HCA | ConnectX-7 (MT4126), 8x Active, Rate 400 |
| NCCL | 2.23.4 |
| Driver | 550.163.01 (probe run) / 580.95.05 (preflight run) |
| CUDA | 12.4.1 (host) |
| DCGM Level 1 | 9/9 pass |
| Power limit | 700W per GPU |
| Idle power | 75-79W per GPU |
| CPU | Intel Xeon Platinum 8468, 128 cores |
| Memory | 1.5 TiB |

## Sample Summary JSON Note

The file `examples/sample-results/summary.json` in the redstone repo is a **synthetic example** showing the output format. It does not represent an actual run:

| Field | Sample JSON | Actual (20260403 run) |
|-------|-------------|----------------------|
| Driver | 550.54.15 | 580.95.05 |
| CUDA | 12.8 | 12.4 |
| NCCL all_reduce | 452 GB/s | 478.9 GB/s |
| Matmul TFLOPS | 989.2 | 770.0 |

The sample JSON is useful for understanding the output schema but should not be cited as measured performance.

[Back to Results](../results.md)

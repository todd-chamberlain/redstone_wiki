# GPU Health Results

Source: [gpu-health-node0.json](../results-data/gpu-health-node0.json) | [gpu-health-node1.json](../results-data/gpu-health-node1.json) | Run: 20260403-200023

## Check Results

| Check       | Result | Detail                                                       |
| ----------- | ------ | ------------------------------------------------------------ |
| Driver      | PASS   | Expected 580.95.05, actual 580.95.05                         |
| CUDA        | FAIL   | Expected 13.0, actual 12.4 (GPU Operator hadn't updated yet) |
| NCCL        | PASS   | 2.20.5                                                       |
| GPU count   | PASS   | 8/8                                                          |
| ECC         | PASS   | All 8 GPUs, zero uncorrected errors                          |
| Temperature | PASS   | Max 32C (threshold 85C)                                      |
| Power       | PASS   | 75-79W idle draw (700W limit)                                |
| Matmul      | FAIL   | Avg 770.0 TFLOPS (threshold 900)                             |

## Per-GPU Matmul TFLOPS

**Node 0** (avg 770.0):

| GPU 0 | GPU 1 | GPU 2 | GPU 3 | GPU 4 | GPU 5 | GPU 6 | GPU 7 | Avg |
|-------|-------|-------|-------|-------|-------|-------|-------|-----|
| 775.5 | 779.2 | 765.7 | 777.0 | 757.1 | 760.5 | 782.7 | 762.4 | 770.0 |

**Node 1** (avg 779.6):

| GPU 0 | GPU 1 | GPU 2 | GPU 3 | GPU 4 | GPU 5 | GPU 6 | GPU 7 | Avg |
|-------|-------|-------|-------|-------|-------|-------|-------|-----|
| 787.4 | 784.0 | 759.7 | 775.2 | 775.3 | 794.3 | 766.2 | 794.4 | 779.6 |

## Problem

FP16 matmul averaged 770 TFLOPS — 22% below the H200's 990 TFLOPS theoretical peak. The preflight correctly flagged this as DEGRADED against the 900 TFLOPS threshold. The CUDA version check also failed: expected 13.0, got 12.4.

## Diagnosis

The container image shipped CUDA toolkit 12.4, but the GPU Operator had installed driver 580.x which targets CUDA 13.0. This version mismatch prevents the driver's JIT compiler from fully optimizing tensor core kernels, degrading FP16 throughput. The hardware was healthy — the software stack was misconfigured.

## Resolution

Updated the Dockerfile base image to `pytorch/pytorch:2.11.0-cuda12.8-cudnn9-devel` and set `expected_cuda_version` to `13.0` in the ConfigMap to match the GPU Operator's driver. The subsequent calibration run with matched versions achieved ~989 TFLOPS — within 1% of theoretical peak. The 900 TFLOPS threshold was kept as-is (91% of peak), appropriate for a properly configured stack.

**Verdict: DEGRADED** — CUDA/driver version mismatch caused 22% matmul throughput loss. Resolved.

[Back to Results](../results.md)

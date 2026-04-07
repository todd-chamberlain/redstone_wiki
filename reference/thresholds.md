# Thresholds

All preflight thresholds with their defaults, code locations, and Terraform variable names.

See [ConfigMap Thresholds](../patterns/configmap-thresholds.md) for the design rationale.

---

## GPU Health Thresholds

| Threshold | Default | Terraform Variable | ConfigMap Key | Code Location |
|-----------|---------|-------------------|---------------|---------------|
| GPU temperature | 85 C | `gpu_temperature_threshold_c` | `gpu_temperature_threshold_c` | [GPU Health](../components/gpu-health.md) |
| Matmul TFLOPS | 900 | `matmul_tflops_threshold` | `matmul_tflops_threshold` | [GPU Health](../components/gpu-health.md) |
| Expected GPU count | 8 | `expected_gpu_count` | `expected_gpu_count` | [GPU Health](../components/gpu-health.md) |

## Version Compatibility Matrix

Defined in `compatibility_matrix`, consumed in [`gpu_health.py`](../components/gpu-health.md).

| Parameter | Default | Used By |
|-----------|---------|---------|
| `driver_min_major` | 550 | [GPU Health](../components/gpu-health.md) |
| `driver_max_major` | 590 | same |
| `cuda_min_major` | 12 | [GPU Health](../components/gpu-health.md) |
| `cuda_max_major` | 13 | same |
| `pytorch_min` | `2.0` | [GPU Health](../components/gpu-health.md) |
| `nccl_min_major` | 2 | [GPU Health](../components/gpu-health.md) |

## NCCL Intra-Node Thresholds

Per-collective thresholds for single-node NVSwitch bandwidth.

| Collective | Default (GB/s) | Terraform Variable | ConfigMap Key | Code |
|-----------|----------------|-------------------|---------------|------|
| all_reduce | 450 | `nccl_allreduce_bw_threshold` | `nccl_allreduce_bw_threshold` | [NCCL Test](../components/nccl-test.md) |
| all_gather | 340 | `nccl_allgather_bw_threshold` | `nccl_allgather_bw_threshold` | [NCCL Test](../components/nccl-test.md) |
| reduce_scatter | 340 | `nccl_reducescatter_bw_threshold` | `nccl_reducescatter_bw_threshold` | [NCCL Test](../components/nccl-test.md) |

## InfiniBand Thresholds

| Threshold | Default | Terraform Variable | ConfigMap Key | Code |
|-----------|---------|-------------------|---------------|------|
| IB write bandwidth | 390 Gb/s | `ib_bw_gbps_threshold` | `ib_bw_gbps_threshold` | [IB Topology](../components/ib-topology.md) |
| IB write latency | 5 us | `ib_latency_us_threshold` | `ib_latency_us_threshold` | [IB Topology](../components/ib-topology.md) |
| Multi-node NCCL | 450 GB/s | `nccl_multinode_bw_gbps_threshold` | `nccl_multinode_bw_gbps_threshold` | [`preflight-config.tf` L58](../infrastructure/terraform.md) |

## Storage Thresholds

| Threshold | Default | Terraform Variable | ConfigMap Key | Code |
|-----------|---------|-------------------|---------------|------|
| Seq read BW | 500 MB/s | `storage_seq_read_mbps` | `storage_seq_read_mbps` | [Storage Test](../components/storage-test.md) |
| Seq write BW | 400 MB/s | `storage_seq_write_mbps` | `storage_seq_write_mbps` | [Storage Test](../components/storage-test.md) |
| Random read IOPS | 5000 | `storage_rand_read_iops` | `storage_rand_read_iops` | [Storage Test](../components/storage-test.md) |
| Contention max | 50% | `storage_contention_max_pct` | `storage_contention_max` | [Orchestrator](../components/orchestrator.md) |

## TCP Threshold

| Threshold | Default | Terraform Variable | ConfigMap Key | Code |
|-----------|---------|-------------------|---------------|------|
| TCP bandwidth | 10 Gb/s | `tcp_bw_gbps_threshold` | `tcp_bw_gbps_threshold` | [TCP Test](../components/tcp-test.md) |

## Policy Thresholds

| Threshold | Default | Terraform Variable | ConfigMap Key | Code |
|-----------|---------|-------------------|---------------|------|
| Min viable GPUs | 8 | `min_viable_gpu_count` | `min_viable_gpu_count` | [Orchestrator](../components/orchestrator.md) |

## Distributed Validation Thresholds

Three-level thresholds: optimal / pass / floor.

### NCCL 16-GPU (Cross-Node)

| Level | Default (GB/s) | Terraform Variable |
|-------|----------------|-------------------|
| Optimal | 450 | `distributed_nccl_16gpu_optimal` |
| Pass | 400 | `distributed_nccl_16gpu_pass` |
| Floor | 300 | `distributed_nccl_16gpu_floor` |

### NCCL 8-GPU (Intra-Node)

| Level | Default (GB/s) | Terraform Variable |
|-------|----------------|-------------------|
| Optimal | 470 | `distributed_nccl_8gpu_optimal` |
| Pass | 440 | `distributed_nccl_8gpu_pass` |
| Floor | 350 | `distributed_nccl_8gpu_floor` |

### TP Scaling Ratio (TP=8 vs TP=16)

| Level | Default | Terraform Variable |
|-------|---------|-------------------|
| Optimal | 0.85 | `distributed_tp_ratio_optimal` |
| Pass | 0.70 | `distributed_tp_ratio_pass` |
| Floor | 0.50 | `distributed_tp_ratio_floor` |

### PP Bubble Overhead (Lower is Better)

| Level | Default | Terraform Variable |
|-------|---------|-------------------|
| Optimal | 0.10 | `distributed_pp_bubble_optimal` |
| Pass | 0.20 | `distributed_pp_bubble_pass` |
| Floor | 0.35 | `distributed_pp_bubble_floor` |

### RDMA Bandwidth

| Level | Default (Gb/s) | Terraform Variable |
|-------|----------------|-------------------|
| Optimal | 390 | `distributed_rdma_bw_optimal` |
| Pass | 350 | `distributed_rdma_bw_pass` |
| Floor | 200 | `distributed_rdma_bw_floor` |

## Probe Calibration Data

From [`preflight-config.tf` L5-L12](../infrastructure/terraform.md) (2026-04-02, fabric-7, H200, eu-north1):

| Metric | Measured | Threshold Set |
|--------|----------|---------------|
| NCCL intra-node all_reduce | 481 GB/s | 450 GB/s |
| NCCL multi-node all_reduce | 478 GB/s | 450 GB/s |
| IB per-port bandwidth | 395 Gb/s | 390 Gb/s |
| IB latency (same leaf) | 3.26 us | 5 us |

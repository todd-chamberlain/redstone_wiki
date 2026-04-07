# InfiniBand Results

Source: [ib-topology.json](../results-data/ib-topology.json) | Run: 20260403-200023 | 8 ports probed

## Problem

All 8 IB ports returned `parse_failed` with empty output. The perftest binaries (`ib_write_lat`, `ib_write_bw`) could not execute, so no per-port latency or bandwidth was captured and leaf/spine topology inference was not possible.

## Diagnosis

The IB topology job manifest lacked `IPC_LOCK` capability. The perftest binaries call `ibv_reg_mr()` to pin DMA buffers for RDMA — the kernel refuses this without `IPC_LOCK`. The distributed validation job, which runs with `privileged: true`, had no issue using IB via NCCL on the same fabric.

## IB Confirmed Healthy (Indirect Evidence)

- Raw probe logs (`ibstat`): 8x ConnectX-7 ports Active at Rate 400
- NCCL 16-GPU cross-node all_reduce: 481.72 GB/s bus BW (OPTIMAL) — impossible without functional IB
- TP=16 scaling: 85.1% of TP=8 — the all-reduce successfully crossed the IB fabric

## Resolution

Updated `ib-topology-job.yaml` to add `securityContext.capabilities.add: ["IPC_LOCK"]`. The orchestrator's dynamically-created IB server/client jobs already include this capability in the manifest template. A subsequent run with the corrected manifest would capture per-port latency/bandwidth measurements and enable the spread-based leaf/spine topology inference.

**Verdict: PARTIAL** — IB fabric is healthy (proven by NCCL), per-port profiling blocked by manifest issue (resolved).

[Back to Results](../results.md)

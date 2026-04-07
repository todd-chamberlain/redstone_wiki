# NCCL Fabric Results

## Final Run (20260406-233101) — Both Nodes

Source: [nccl-node0.json](../results-data/nccl-node0.json) | [nccl-node1.json](../results-data/nccl-node1.json) | Mode: intranode (NCCL_IB_DISABLE=1) | 8 GPUs per node

### Node 0 (`computeinstance-e00ma405v3wxc4avss`)

| Collective | Bus BW (GB/s) | Alg BW (GB/s) | Threshold | Verdict |
|-----------|---------------|---------------|-----------|---------|
| all_reduce | 481.43 | 275.10 | 450 | PASS |
| all_gather | 362.12 | 413.85 | 340 | PASS |
| reduce_scatter | 363.52 | 415.46 | 340 | PASS |

### Node 1 (`computeinstance-e00pwe6w4xnbfpxbx8`)

| Collective | Bus BW (GB/s) | Alg BW (GB/s) | Threshold | Verdict |
|-----------|---------------|---------------|-----------|---------|
| all_reduce | 480.75 | 274.71 | 450 | PASS |
| all_gather | 362.61 | 414.41 | 340 | PASS |
| reduce_scatter | 362.81 | 414.64 | 340 | PASS |

All collectives above threshold on both nodes. NVSwitch fabric healthy. Message size: 8 GB. Both nodes within 0.2% of each other on all_reduce.

## Initial Run (20260403-200023) — Single Node

Source: [nccl-intranode.json](../results-data/nccl-intranode.json)

| Collective | Bus BW (GB/s) | Alg BW (GB/s) | Threshold | Verdict |
|-----------|---------------|---------------|-----------|---------|
| all_reduce | 478.9 | 273.65 | 450 | PASS |
| all_gather | 367.89 | 420.45 | 340 | PASS |
| reduce_scatter | 362.30 | 414.05 | 340 | PASS |

**Verdict: PASS** (both runs, both nodes)

[Back to Results](../results.md)

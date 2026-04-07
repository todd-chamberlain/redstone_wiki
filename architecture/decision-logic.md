# Decision Logic

How the orchestrator computes the final READY / DEGRADED / NOT_READY verdict.

## Verdict Levels

| Verdict | Meaning | Action |
|---------|---------|--------|
| **READY** | All tests passed on all nodes | Proceed to training |
| **DEGRADED** | Some tests failed but enough GPUs for training | Investigate, may proceed with reduced capacity |
| **NOT_READY** | Insufficient viable GPUs | Block training, investigate and repair |

## Aggregation Logic

The [`aggregate()`](../components/orchestrator.md) function implements a cascading check:

### Step 1: GPU Viability

```python
viable_nodes = healthy_nodes + degraded_nodes
total_viable_gpus = len(viable_nodes) * expected_gpu_count
```

See [Orchestrator](../components/orchestrator.md)

- **DEGRADED** nodes (e.g., temperature warning but GPUs functional) are counted as viable
- **FAILED** nodes (critical: GPU count wrong, ECC errors) are excluded

### Step 2: NOT_READY Check

See [Orchestrator](../components/orchestrator.md)

If `total_viable_gpus < min_viable_gpu_count` → **NOT_READY**

Default: `min_viable_gpu_count = 8` (one full node). Set via [Terraform variable](../infrastructure/terraform.md).

### Step 3: DEGRADED from Failed Nodes

See [Orchestrator](../components/orchestrator.md)

If any nodes failed or are degraded → **DEGRADED** with remediation actions:
- Failed nodes: tainted NoSchedule, suggested to replace/repair
- Degraded nodes: suggested to investigate per-node results

### Step 4: Phase-Level Checks

If all nodes are healthy, check individual phase results:

| Check | Source | Verdict on Failure |
|-------|--------|-------------------|
| NCCL bandwidth | [Orchestrator](../components/orchestrator.md) | DEGRADED |
| Storage throughput | [Orchestrator](../components/orchestrator.md) | DEGRADED |
| Storage contention | [Orchestrator](../components/orchestrator.md) | DEGRADED |
| IB topology | [Orchestrator](../components/orchestrator.md) | DEGRADED |

All pass → **READY** (with topology annotation: same-leaf or multi-leaf).

### Step 5: Remediation

Each failure type generates a remediation entry with:
- `action`: what to investigate
- `nodes`: affected nodes
- `detail`: specific guidance

See [Orchestrator](../components/orchestrator.md)

## GPU Health Verdicts

The [`gpu_health.py` main()](../components/gpu-health.md) uses a simpler model:

| Condition | Verdict |
|-----------|---------|
| `gpu_count` or `ecc_status` fails | **FAIL** (critical) → node tainted |
| Any non-critical check fails | **DEGRADED** |
| All checks pass | **PASS** |

Critical failures trigger [`taint_node()`](../components/gpu-health.md) which applies a `redstone=failed:NoSchedule` taint.

## Distributed Validation Verdicts

[`distributed_validate.py`](../components/distributed-validate.md) uses a 4-level system:

| Level | Meaning |
|-------|---------|
| OPTIMAL | Within 95% of theoretical peak |
| PASS | Above minimum floor (training-viable) |
| DEGRADED | Below floor but functional |
| FAIL | Non-functional |

The [`verdict()`](../components/distributed-validate.md) function compares against three thresholds per metric: optimal, pass, floor. Supports both higher-is-better (bandwidth) and lower-is-better (bubble overhead).

## Cross-References

- [Orchestrator](../components/orchestrator.md) — the `aggregate()` implementation
- [GPU Health](../components/gpu-health.md) — per-node check details
- [ConfigMap Thresholds](../patterns/configmap-thresholds.md) — how thresholds are configured
- [Node Tainting](../patterns/node-tainting.md) — auto-exclusion of bad nodes

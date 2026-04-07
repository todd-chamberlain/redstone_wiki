# Node Tainting

Automatic exclusion of bad GPU nodes using Kubernetes taints to prevent workload scheduling on hardware that failed validation.

## How It Works

When [`gpu_health.py`](../components/gpu-health.md) detects a critical failure (GPU count mismatch or ECC errors), it calls [`taint_node()`](../components/gpu-health.md):

```python
taints.append(client.V1Taint(
    key="redstone", value=reason, effect="NoSchedule"
))
v1.patch_node(NODE_NAME, body)
```

This applies a `redstone=<reason>:NoSchedule` taint, which prevents any pod without a matching toleration from being scheduled on the node.

## Critical vs Non-Critical

| Check | Critical? | On Failure |
|-------|-----------|------------|
| `gpu_count` | Yes | Taint node, verdict FAIL |
| `ecc_status` | Yes | Taint node, verdict FAIL |
| `driver_version` | No | Verdict DEGRADED |
| `temperature` | No | Verdict DEGRADED |
| `matmul_bench` | No | Verdict DEGRADED |
| Any exception | Yes | Taint node, verdict FAIL |

See [`gpu_health.py`](../components/gpu-health.md):
```python
if name in ("gpu_count", "ecc_status"):
    critical_failure = True
```

## RBAC

The taint operation requires the `redstone-runner` ServiceAccount to have `patch` access on nodes. Granted by the [`redstone-node-access` ClusterRole](../infrastructure/kubernetes-manifests.md).

## Taint Clearing

Before each preflight run, [`make preflight-run`](../infrastructure/makefile.md) clears previous taints:

```bash
kubectl get nodes -l nvidia.com/gpu.present=true -o name | \
    xargs -I{} kubectl taint nodes {} redstone- 2>/dev/null
```

The trailing `-` removes the taint.

## Orchestrator Handling

The [`aggregate()`](../components/orchestrator.md) function includes tainted/failed nodes in remediation output:

```python
remediation.append({
    "action": "exclude_failed_nodes",
    "nodes": failed_nodes,
    "detail": "These nodes are tainted NoSchedule and will not receive workloads",
})
```

## Cross-References

- [Architecture Overview](../architecture/overview.md) — how node tainting fits into the system's self-healing design
- [GPU Health](../components/gpu-health.md) — where tainting is triggered
- [Decision Logic](../architecture/decision-logic.md) — how failed nodes affect the verdict
- [Kubernetes Manifests](../infrastructure/kubernetes-manifests.md) — RBAC for node patching

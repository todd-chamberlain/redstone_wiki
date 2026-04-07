# Poll-Based Orchestration

The orchestrator uses a reconciliation loop that polls for state changes rather than an event-driven workflow engine like Argo.

## Design

The [reconciliation loop](../components/orchestrator.md) runs continuously:

```python
while time.time() - start_time < MAX_RUNTIME:
    current_gpu_nodes = get_gpu_nodes()
    new_nodes = set(current_gpu_nodes) - validated_nodes
    # ... run phases on new nodes ...
    time.sleep(RECONCILE_INTERVAL)
```

Key properties:
- **Progressive**: validates nodes as they become available (handles staggered provisioning)
- **Idempotent**: safe to restart (checks what's already done before acting)
- **Self-healing**: detects new nodes and runs appropriate phases
- **Bounded**: `MAX_RUNTIME` (25 min) prevents infinite loops

## Why Not Argo / Tekton / DAG

| Concern | Argo/Tekton | Reconciliation Loop |
|---------|-------------|-------------------|
| **Dependencies** | Requires Argo install | Zero dependencies |
| **Dynamic topology** | DAG is static | Handles nodes joining at runtime |
| **Debugging** | Multi-pod logs | Single pod, linear log |
| **Failure modes** | DAG executor + worker pods | Single Python process |
| **Complexity** | YAML DAG + templates | ~200 lines of Python |

The main advantage of the reconciliation pattern is handling **dynamic node availability**: GPU nodes in cloud environments often provision at different times, and the orchestrator must adapt.

## Phase Dependencies

The orchestrator enforces implicit ordering:

```
Phase 1: GPU Health (DaemonSet, pre-deployed)
    ↓ healthy nodes identified
Phase 2: NCCL intra-node (per healthy node)
    ↓ NVSwitch validated
Phase 2.5: IB topology (when >= 2 nodes, once)
    ↓ IB fabric mapped
Phase 3: Storage (per viable node, isolation then contention)
    ↓ filesystem validated
Phase 4: TCP (when >= 2 nodes, once)
    ↓ control plane validated
Aggregate → verdict
```

Each phase only runs on nodes that haven't been tested yet (tracked via `nccl_tested`, `storage_tested` sets).

## State Tracking

```python
validated_nodes = set()    # passed health check
nccl_tested = set()        # have NCCL results
storage_tested = set()     # have storage results
multinode_tested = False   # IB + TCP done
```

See [Orchestrator](../components/orchestrator.md)

## Cross-References

- [Architecture Overview](../architecture/overview.md) — how poll-based orchestration fits into the system architecture
- [Orchestrator](../components/orchestrator.md) — implementation details
- [Execution Flow](../architecture/execution-flow.md) — how it fits in the pipeline
- [Shared FS Coordination](shared-fs-coordination.md) — how results pass between phases

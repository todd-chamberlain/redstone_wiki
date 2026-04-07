# Shared Filesystem Coordination

Redstone uses the shared NFS filesystem for inter-pod coordination: result passing, master IP discovery, and server readiness signaling.

## Result Passing

All test scripts write JSON results to the shared filesystem at:

```
/mnt/data/redstone-results/<RUN_ID>/<phase>/<node>.json
```

The orchestrator polls for these files using [`read_json_results`](../components/orchestrator.md):

```python
results = read_json_results(f"{RESULTS_DIR}/gpu-health/*.json")
```

Avoids the need for direct pod-to-pod communication or a message queue.

## Master IP Discovery

The [distributed validation job](../infrastructure/kubernetes-manifests.md) uses shared FS for master address discovery:

1. Pod with `JOB_COMPLETION_INDEX=0` writes its IP to `$SHARED_DIR/master-ip`
2. Other pods poll for this file (up to 120s)
3. All pods pass `MASTER_ADDR` to `torchrun`

No need for a headless Service or DNS for one-shot jobs.

## Server Ready Signaling

The [IB topology server](../components/ib-topology.md) signals readiness:

```python
ready_file = f"{RESULTS_DIR}/server-ready-{NODE_NAME}-{RUN_ID}"
with open(ready_file, "w") as f:
    f.write(f"{NODE_NAME}\n")
```

The [client](../components/ib-topology.md) polls for this file before connecting:

```python
for attempt in range(30):
    if os.path.exists(ready_file):
        break
    time.sleep(2)
```

The `RUN_ID` suffix prevents stale signals from previous runs.

## Why Not K8s Mechanisms

| Alternative | Why Not |
|-------------|---------|
| K8s Services | Overkill for ephemeral one-shot jobs |
| Pod readiness probes | DaemonSet pods don't have completion semantics |
| Init containers | Can't signal across independent Jobs |
| ConfigMaps | Would need write access from pods |
| Message queues | Extra infrastructure, complexity |

The shared filesystem is already provisioned (filestore), mounted on all nodes, and requires no additional setup.

## Mount Configuration

Every pod mounts the shared filesystem via hostPath:

```yaml
volumes:
  - name: results
    hostPath:
      path: /mnt/data/redstone-results
      type: DirectoryOrCreate
```

The filestore is provisioned by Terraform via the [k8s-training module](../infrastructure/terraform.md).

## Cross-References

- [Architecture Overview](../architecture/overview.md) — how shared filesystem coordination fits into the system design
- [Orchestrator](../components/orchestrator.md) — polls for results
- [IB Topology](../components/ib-topology.md) — server readiness signaling
- [Distributed Validate](../components/distributed-validate.md) — master IP discovery
- [Poll-Based Orchestration](poll-based-orchestration.md) — why polling instead of events
